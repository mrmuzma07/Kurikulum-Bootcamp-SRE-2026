# 03 — Security, Reliability, dan Observability Preparation: Membangun Pertahanan Berlapis pada Chart Helm

> **Ringkasan:** Di mata seorang *Site Reliability Engineer* (SRE), sebuah Helm Chart bukan sekadar kumpulan template YAML untuk meluncurkan aplikasi, melainkan **gerbang utama rantai pasok (*Software Supply Chain*)** menuju klaster Kubernetes produksi. Mengabaikan aspek keamanan dan keandalan pada tingkat Chart akan melahirkan aplikasi yang rentan terhadap eksploitasi kontainer (*container breakout*), pemadaman lalu lintas akibat *abrupt termination*, dan kebutaan sistem akibat ketiadaan telemetri. Dokumen ini membedah penguatan postur keamanan (*Pod Security Standards*), perancangan siklus hidup anggun (*Graceful Shutdown & Probes*), integrasi telemetri Prometheus via *ServiceMonitor*, serta penyusunan gerbang audit rilis terverifikasi.

---

## 1. Analogi Pemula: Benteng Kastil Berparit Air & Menara Pengawas 24/7

Bagi pemula, membuat manifest Kubernetes sering kali dilakukan seminimal mungkin: tidak ada batasan memori, tidak ada probe, dan kontainer dibiarkan berjalan sebagai `root`:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI BENTENG KASTIL BERLAPIS vs POD KUBERNETES           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ BENTENG AMATIR: RUMAH KAYU DENGAN PINTU TERBUKA LEBAR ]                  │
│  - Masalah: Siapa pun yang masuk ke dalam rumah otomatis diangkat menjadi   │
│    Raja yang memiliki kunci seluruh kerajaan (`runAsUser: 0` / root)!       │
│  - Jika ada penyusup (hacker) yang berhasil mengeksploitasi bug kode web,   │
│    ia bisa langsung memodifikasi file sistem operasi host fisik dan         │
│    menguasai seluruh node bare-metal (**CONTAINER BREAKOUT**)!              │
│  - Tidak ada alarm pemadam kebakaran, tidak ada penjaga jaga malam.         │
│                                                                             │
│  [ BENTENG SRE KASTIL BERLAPIS (RESTRICTED POD SECURITY) ]                  │
│  - **Parit Air Dalam (NetworkPolicy):**                                     │
│    Hanya kurir resmi dengan stempel kerajaan yang boleh mendekat ke gerbang.│
│  - **Pintu Gerbang Besi (Pod Security Context):**                           │
│    1. Siapa pun yang masuk dilucuti senjatanya (`drop: [ALL]` capabilities).│
│    2. Tamu dilarang menjadi raja (`runAsNonRoot: true`, user biasa 10001).  │
│    3. Tamu dilarang menulis atau mencoret dinding kastil                    │
│       (`readOnlyRootFilesystem: true`). Jika butuh menulis catatan,         │
│       disediakan kertas sementara di meja khusus (`emptyDir: /tmp`).        │
│  - **Menara Pengawas & Sirine 24/7 (Prometheus ServiceMonitor):**           │
│    Setiap detak jantung penghuni kastil dipantau radar telemetri secara     │
│    real-time, mendeteksi kebakaran sebelum api membesar!                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Penguatan Postur Keamanan Kontainer (Security Context Hardening)

Mengikuti rekomendasi standar keamanan resmi Kubernetes (**Pod Security Standards - Restricted Profile**), setiap Pod yang dihasilkan oleh Chart produksi wajib mengunci hak akses secara ketat:

### 2.1 Konfigurasi Deklaratif pada `templates/deployment.yaml`

```yaml
spec:
  template:
    metadata:
      labels:
        {{- include "internal-app.selectorLabels" . | nindent 8 }}
    spec:
      # 1. PENGUATAN LEVEL POD (Pod-Level Security Context)
      securityContext:
        runAsNonRoot: true         # Mencegah kontainer berjalan sebagai root (UID 0)
        runAsUser: 10001           # Menetapkan UID non-root spesifik (sre-app)
        runAsGroup: 10001          # Menetapkan GID non-root spesifik
        fsGroup: 10001             # Memberikan izin kepemilikan volume storage
        seccompProfile:
          type: RuntimeDefault     # Mengaktifkan filter system call aman bawaan kernel

      # 2. PENGUATAN TOKEN IDENTITAS KUBERNETES
      # Jangan pasang token Kube-APIServer jika aplikasi tidak membutuhkannya!
      automountServiceAccountToken: false

      containers:
        - name: {{ .Chart.Name }}
          # 3. PENGUATAN LEVEL KONTAINER (Container-Level Security Context)
          securityContext:
            allowPrivilegeEscalation: false # Blokir ekskalasi hak akses (sudo/setuid)
            readOnlyRootFilesystem: true     # Kunci seluruh filesystem agar tidak bisa dimodifikasi
            capabilities:
              drop:
                - ALL                        # Lucuti seluruh Linux kernel capabilities berbahaya!

          # 4. PENYEDIAAN DIREKTORI TEMPORARY BERIZIN TULIS (Scratch Space)
          volumeMounts:
            - name: tmp-volume
              mountPath: /tmp
            - name: cache-volume
              mountPath: /app/cache

      volumes:
        - name: tmp-volume
          emptyDir:
            sizeLimit: 100Mi       # Batasi ukuran disk temporary agar tidak memakan disk node
        - name: cache-volume
          emptyDir:
            sizeLimit: 200Mi
```

> [!CAUTION]
> **Mengapa `readOnlyRootFilesystem: true` Sangat Krusial?**
> Jika seorang hacker berhasil mengeksekusi kode berbahaya via *Remote Code Execution* (RCE), tindakan pertamanya biasanya adalah mengunduh skrip crypto-miner atau exploit biner ke dalam folder `/tmp` atau `/bin`. Dengan mengaktifkan `readOnlyRootFilesystem: true`, kernel Linux akan seketika menolak operasi penulisan berkas dengan pesan `Read-only file system`, melumpuhkan serangan hacker seketika!

---

## 3. Arsitektur Keandalan: Graceful Shutdown & Probes Alignment

Di lingkungan produksi yang padat, Pod sering kali dipindahkan (*evicted*), di-*restart*, atau di-scale secara dinamis. Kegagalan mengatur pemadaman anggun (*graceful shutdown*) akan memicu lonjakan error `HTTP 502 Bad Gateway` pada pengguna.

### 3.1 Penyelarasan Tiga Jenis Probes (Probes Alignment)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PERBEDAAN FUNGSIONAL TIGA JENIS PROBES                      │
├─────────────────┬──────────────────────────────┬────────────────────────────┤
│ Jenis Probe     │ Pertanyaan yang Diajukan     │ Tindakan jika GAGAL        │
├─────────────────┼──────────────────────────────┼────────────────────────────┤
│ 1. Startup      │ "Apakah aplikasi selesai     │ Kubelet menunggu; jika     │
│    Probe        │ inisialisasi awal & cache?"  │ timeout -> RESTART Pod.    │
├─────────────────┼──────────────────────────────┼────────────────────────────┤
│ 2. Readiness    │ "Apakah aplikasi siap        │ Kubelet MELEPAS Pod dari   │
│    Probe        │ menerima lalu lintas user?"  │ Endpoints Service (No 502)!│
├─────────────────┼──────────────────────────────┼────────────────────────────┤
│ 3. Liveness     │ "Apakah aplikasi masih hidup │ Kubelet MEMBUNUH & RESTART │
│    Probe        │ atau tersangkut deadlock?"   │ Pod kontainer seketika!    │
└─────────────────┴──────────────────────────────┴────────────────────────────┘
```

### 3.2 Siklus Hidup Pemadaman Anggun (Graceful Termination Lifecycle)

Saat Kubernetes memutuskan untuk mematikan sebuah Pod (misal saat proses *rolling update*), ada jeda waktu tunda antara pencabutan IP Pod dari tabel iptables Kube-Proxy dan pengiriman sinyal `SIGTERM`:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR PEMADAMAN ANGGUN (GRACEFUL TERMINATION)                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Kubelet menerima perintah terminate Pod                                 │
│            │                                                                │
│            ├─────────────────────────────────┐                              │
│            ▼                                 ▼                              │
│  2. Hapus IP Pod dari Service Endpoints   3. Jalankan `preStop` Hook        │
│     (Kube-Proxy butuh waktu 1-3 detik       (Tidur sejenak: `sleep 5`)      │
│      untuk mengupdate iptables node)         Aplikasi tetap melayani trafik │
│            │                                 yang sedang berjalan!          │
│            └────────────────┬────────────────┘                              │
│                             │                                               │
│                             ▼                                               │
│  4. Kirim sinyal `SIGTERM` ke aplikasi web                                  │
│     (Aplikasi menolak request baru, menyelesaikan request aktif yang ada)   │
│                             │                                               │
│                             ▼ Selesai dalam batas waktu?                    │
│        [ YA (Exit 0) ]                     [ TIDAK (Melebihi 60 Detik) ]    │
│              │                                           │                  │
│              ▼                                           ▼                  │
│     Pod Berhenti Anggun                     Kirim `SIGKILL` (Paksa Mati!)   │
│     (Zero Error HTTP 502!)                                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Implementasi pada manifest `deployment.yaml`:
```yaml
spec:
  template:
    spec:
      # Batas waktu toleransi pemadaman anggun (Default 30s, naikkan ke 60s untuk sistem transaksi)
      terminationGracePeriodSeconds: 60
      containers:
        - name: app
          lifecycle:
            preStop:
              exec:
                # Beri jeda 5 detik agar tabel iptables selesai diperbarui sebelum aplikasi mati
                command: ["/bin/sh", "-c", "sleep 5"]
```

---

## 4. Kesiapan Observabilitas (Observability Preparation)

Sebuah Chart produksi harus siap diintegrasikan dengan tumpukan pemantauan (*Full-Stack Observability*) tanpa perlu memodifikasi kode aplikasi. Standar industri Kubernetes adalah menggunakan Custom Resource Definition (CRD) **ServiceMonitor** dari Prometheus Operator.

### 4.1 Template Deklaratif `templates/servicemonitor.yaml`

```yaml
{{- if .Values.monitoring.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "internal-app.fullname" . }}
  labels:
    {{- include "internal-app.labels" . | nindent 4 }}
    # Label pemilih yang dibaca oleh server Prometheus untuk memulai scraping
    release: {{ .Values.monitoring.prometheusRelease | default "prometheus" }}
spec:
  selector:
    matchLabels:
      {{- include "internal-app.selectorLabels" . | nindent 6 }}
  endpoints:
    - port: http
      path: {{ .Values.monitoring.path | default "/metrics" }}
      interval: {{ .Values.monitoring.interval | default "15s" }}
      scrapeTimeout: {{ .Values.monitoring.scrapeTimeout | default "10s" }}
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: "(http_requests_total|http_request_duration_seconds_bucket|jvm_memory_used_bytes)"
          action: keep
{{- end }}
```

Dengan template di atas, operator cukup menyalakan parameter di `values-prod.yaml`:
```yaml
monitoring:
  enabled: true
  path: /metrics
  interval: 15s
```
Dan seketika metrik **RED (Rate, Errors, Duration)** dari aplikasi akan mengalir ke Prometheus, Mimir, dan Dashboard Grafana!

---

## 5. Gerbang Kualitas CI/CD & Bukti Audit Rilis (Release Evidence)

Sebelum Helm Chart diizinkan masuk ke lingkungan produksi, pipeline CI/CD wajib menjalankan serangkaian gerbang pengujian mutu berlapis:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 GERBANG KUALITAS DAN KEAMANAN CI/CD HELM                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ GERBANG 1: SYNTAX & LINTING ] ──► `helm lint ./internal-app`             │
│                                                │                            │
│  [ GERBANG 2: SCHEMA VALIDATION ] ─► `values.schema.json validation`        │
│                                                │                            │
│  [ GERBANG 3: SECURITY SCAN ] ─────► `trivy config ./internal-app`          │
│                                      (Deteksi privilege escalation/root)    │
│                                                │                            │
│  [ GERBANG 4: SMOKE TEST POD ] ────► `helm test <release> (Di Staging)`     │
│                                                │                            │
│  [ GERBANG 5: HUMAN APPROVAL ] ────► Persetujuan 4-Eyes Principle           │
│                                                │                            │
│  [ OUTPUT: IMMUTABLE RELEASE EVIDENCE RECORD ]                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Format Berkas Ringkasan Bukti Audit Rilis (`release-evidence.json`)

Berkas bukti audit disimpan pada sistem pencatatan perubahan tanpa menyertakan kredensial:

```json
{
  "release_evidence_id": "HELM-REL-2026-10-17-08",
  "timestamp_utc": "2026-10-17T09:45:00Z",
  "provenance": {
    "git_commit_sha": "8f9b2c1743a19e24f706e25776d6ec1438914619",
    "chart_name": "internal-app",
    "chart_semver": "1.2.0",
    "app_version": "2.4.1",
    "oci_artifact_digest": "sha256:8f9b2c1743a19e24f706e25776d6ec1438914619736c4b2b1a8d0e5124fa68b1",
    "container_image_digest": "registry.internal.corp/payment@sha256:4a8b...9c"
  },
  "security_compliance": {
    "pod_security_standard": "restricted",
    "run_as_non_root": true,
    "read_only_root_filesystem": true,
    "trivy_misconfiguration_critical": 0
  },
  "runtime_verification": {
    "target_namespace": "production",
    "replicas_deployed": 3,
    "replicas_ready": 3,
    "helm_test_status": "PASSED",
    "endpoints_active": true
  }
}
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA MASALAH KEAMANAN & KEANDALAN               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "CrashLoopBackOff: Read-only file system" ]                     │
│  Gejala: Pod gagal start seketika setelah `readOnlyRootFilesystem: true`.   │
│  Akar Masalah:                                                              │
│  - Framework aplikasi (misal: Java Spring Boot / Node.js) mencoba menulis   │
│    file log atau cache sementara ke direktori `/app/logs` atau `/tmp`.      │
│  Solusi SRE:                                                                │
│  1. Identifikasi direktori target penulisan dari log error.                │
│  2. Pasang volume temporary `emptyDir` khusus untuk path tersebut:          │
│     volumeMounts:                                                           │
│       - name: app-logs                                                      │
│         mountPath: /app/logs                                                │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "HTTP 502 BAD GATEWAY" SAAT PROSES ROLLING UPGRADE ]            │
│  Gejala: Pengguna mengalami error 502 selama 2-3 detik setiap kali rilis.   │
│  Akar Masalah:                                                              │
│  - Ketiadaan `preStop` hook dengan `sleep`. Pod langsung mati seketika      │
│    menerima `SIGTERM` padahal iptables Kube-Proxy pada node lain belum      │
│    selesai mencabut alamat IP Pod tersebut dari daftar routing aktif!       │
│  Solusi SRE:                                                                │
│  Tambahkan lifecycle hook penunda pada container:                           │
│  lifecycle:                                                                 │
│    preStop:                                                                 │
│      exec:                                                                  │
│        command: ["/bin/sh", "-c", "sleep 5"]                                │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: METRIK TIDAK MUNCUL DI GRAFANA EXPLORE ]                        │
│  Gejala: ServiceMonitor sudah dideploy, tapi Prometheus tidak scrape target│
│  Akar Masalah:                                                              │
│  - Label pada `ServiceMonitor` tidak cocok dengan selektor Prometheus server│
│    (misal server mencari `release: prometheus-stack`).                      │
│  Solusi SRE:                                                                │
│  1. Cek selektor Prometheus: `kubectl get prometheus -o yaml | grep serviceMonitorSelector`│
│  2. Samakan label pada `templates/servicemonitor.yaml`.                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Memindai celah keamanan dan salah konfigurasi pada Helm Chart via Trivy
trivy config ./internal-app

# 2. Memeriksa apakah Pod mematuhi aturan restricted security context
kubectl get pods -n production -o custom-columns=NAME:.metadata.name,USER:.spec.securityContext.runAsUser,READONLY:.spec.containers[0].securityContext.readOnlyRootFilesystem

# 3. Mengecek apakah ServiceMonitor berhasil didaftarkan di Prometheus Operator
kubectl get servicemonitors -n production

# 4. Memverifikasi status target scrape di antarmuka Prometheus
# (Port-forward dashboard Prometheus internal)
kubectl port-forward -n monitoring svc/prometheus-k8s 9090:9090

# 5. Menguji penghentian anggun secara manual (Simulasi Graceful Termination)
kubectl delete pod <nama-pod> -n production && kubectl get events -w -n production
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa dalam arsitektur Pod produksi yang andal, kita sangat disarankan menambahkan siklus `preStop: exec: command: ["/bin/sh", "-c", "sleep 5"]` pada kontainer aplikasi web?
- A. Untuk mendinginkan suhu prosesor CPU node.
- B. Memberikan jeda waktu tunda 5 detik agar seluruh Kube-Proxy dan Ingress Controller di klaster sempat mencabut IP Pod tersebut dari daftar routing aktif, mencegah trafik baru dikirim ke Pod yang sedang bersiap mati (*Eliminating HTTP 502*).
- C. Mempercepat proses compile kode Go.
- D. Agar kontainer otomatis menghapus file log lama.

*Kunci Jawaban:* **B**. `preStop sleep` adalah pola standar SRE untuk mengatasi *race condition* antara perambatan pembaruan Endpoints jaringan dan penghentian proses aplikasi.

### Soal 2 (Pilihan Ganda)
Apa fungsi utama dari menyetel `readOnlyRootFilesystem: true` di dalam `securityContext` kontainer?
- A. Memperkecil pemakaian kuota RAM hingga 50%.
- B. Mengunci sistem berkas kontainer menjadi *read-only* murni, mencegah penyusup (*attacker*) menuliskan malware, biner eksploitasi, atau skrip berbahaya ke dalam kontainer jika terjadi insiden kebocoran kode (*RCE*).
- C. Mencegah developer melakukan `git commit`.
- D. Mematikan seluruh koneksi internet keluar.

*Kunci Jawaban:* **B**. `readOnlyRootFilesystem` merupakan instrumen pertahanan berlapis (*defense-in-depth*) terpenting untuk menghentikan serangan persistensi penyerang di dalam kontainer.

### Soal 3 (Pilihan Ganda)
Kapan sebaiknya flag `automountServiceAccountToken` disetel ke nilai `false` pada manifest Pod?
- A. Hanya jika aplikasi berjalan di sistem operasi Windows.
- B. Pada SEMUA aplikasi bisnis reguler yang tidak memerlukan komunikasi atau otentikasi langsung ke Kube-APIServer, guna mencegah pencurian token RBAC internal jika kontainer diretas.
- C. Saat aplikasi membutuhkan akses ke database MySQL.
- D. Tidak pernah boleh disetel ke `false` karena klaster akan mati.

*Kunci Jawaban:* **B**. Menonaktifkan pemasangan token API secara otomatis adalah prinsip hak akses terendah (*Least Privilege*) yang menutup celah eksfiltrasi kredensial klaster.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Sebuah aplikasi perbankan berbasis microservice Java sering kali mengalami restart acak (*random restart*) saat jam sibuk transaksi. Setelah diinvestigasi oleh tim SRE, ternyata aplikasi tersebut sedang mengalami *Garbage Collection (GC) Pause* selama 10 detik, yang menyebabkan *Liveness Probe* gagal merespons selama 3 kali berturut-turut, memicu Kubelet membunuh kontainer tersebut secara mendadak. Jelaskan mengapa arsitektur probe di atas keliru, dan susun perbaikan parameter probe agar ketersediaan aplikasi terjaga!

*Panduan Solusi:*
1. **Analisis Kekeliruan Arsitektur Probe:**
   Tim developer salah menyamakan antara "Aplikasi sedang sibuk memproses memori / GC Pause" dengan "Aplikasi mengalami deadlock / mati total".
   - **Liveness Probe** bertugas memantau *deadlock* fatal yang tidak bisa pulih sendiri. Menyetel threshold liveness terlalu agresif pada aplikasi Java akan memicu *Restart Storm*: aplikasi yang sedang sibuk dibunuh, saat bangun kembali ia kembali sibuk memuat cache, gagal probe lagi, dan dibunuh lagi berulang-ulang (*Cascading Failure*).
   - Penanganan kelebihan beban sementara (*temporary overload / GC pause*) seharusnya menjadi tugas **Readiness Probe** (cukup lepaskan sementara dari Service agar tidak menerima transaksi baru, JANGAN DIBUNUH!).
2. **Perbaikan Konfigurasi Probe Baku SRE:**
   - **Longgarkan Toleransi Liveness Probe:**
     Berikan ambang kegagalan (*failureThreshold*) yang lebih tinggi dan jeda pemeriksaan lebih manusiawi:
     ```yaml
     livenessProbe:
       httpGet:
         path: /healthz
         port: 8080
       periodSeconds: 15
       timeoutSeconds: 5
       failureThreshold: 5 # Wajib gagal 5x berturut-turut (75 detik) baru dibunuh!
     ```
   - **Kendalikan Beban via Readiness Probe:**
     ```yaml
     readinessProbe:
       httpGet:
         path: /ready
         port: 8080
       periodSeconds: 5
       timeoutSeconds: 2
       failureThreshold: 2 # Jika gagal 2x (10 detik), cukup cabut dari Service!
     ```
   Dengan perbaikan ini, saat GC Pause terjadi, Pod hanya dicabut sementara dari Service tanpa dibunuh, lalu kembali menerima trafik saat GC selesai.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Custom Chart dan Values per Environment](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/01-custom-chart-environment-values.md) — Mengatur nilai pelapisan sumber daya.
  - [02 — Hooks, Tests, NOTES, dan Siklus Upgrade](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/02-hooks-tests-notes-upgrade.md) — Mengorkestrasi migrasi dan smoke testing rilis.
- **Praktik Laboratorium:**
  - [LAB-01 — Custom Application Chart](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/lab/LAB-01-custom-app-chart.md) — Membangun custom chart aplikasi internal lengkap.
  - [LAB-02 — Observability, OCI, Test, dan Rollback](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/lab/LAB-02-observability-oci-test-rollback.md) — Menguji integrasi ServiceMonitor, paket OCI, dan prosedur rollback.
- **Fase Berikutnya:**
  - [Fase 6 — GitOps dengan ArgoCD & GitLab CI](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/) — Mengotomasi deployment Chart Helm ini secara pull-based menggunakan ArgoCD.
  - [Fase 7 — Observability: Prometheus, Grafana, Loki, dan Tempo](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/) — Mengalirkan metrik dari ServiceMonitor ke sistem monitoring penuh.
