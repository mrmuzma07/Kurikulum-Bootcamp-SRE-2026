# 02 — OTel Pipeline, DaemonSet, Helm, dan Debugging

> **Ringkasan:** Merancang deployment **Grafana Alloy** tingkat produksi sebagai **Kubernetes DaemonSet** melalui Helm dan GitOps, memahami kontrak standar **OpenTelemetry Protocol (OTLP)**, mengamankan akses filesystem node (`hostPath` read-only) dan RBAC minimal, serta melakukan debugging pipeline telemetri tanpa membocorkan rahasia ke dalam log sistem.

---

## 1. Konsep & Arsitektur Topologi Deployment Alloy

### 1.1 Analogi Dunia Nyata: "Petugas Inspeksi di Setiap Lantai Gedung (DaemonSet) vs Kantor Pos Pusat (Deployment)"
Bayangkan sebuah gedung pencakar langit 50 lantai (Cluster Kubernetes dengan 50 Node):
- **Pola DaemonSet (Satu Agen di Setiap Lantai/Node):** Di setiap lantai ditempatkan tepat 1 petugas khusus. Tugasnya sangat spesifik: memeriksa meteran listrik lantai tersebut (*node metrics*), mengambil kertas catatan sampah dari setiap ruangan di lantai itu (*file log lokal `/var/log/pods`*), serta menerima paket surat dari penghuni lantai tersebut (*node-local OTLP*), lalu mengirimkannya ke gudang pusat.
- **Risiko Sabotase / Ledakan Radius (*Blast Radius*):** Karena petugas DaemonSet berada di **setiap lantai** dan memegang kunci akses ke lorong gedung (`hostPath`), jika petugas tersebut ceroboh (misal memakan memori tanpa batas atau diberi kunci master ke seluruh brankas penghuni), maka **seluruh 50 lantai gedung bisa lumpuh serentak!**

Oleh karena itu, mendesain DaemonSet telemetri membutuhkan disiplin keamanan dan pembatasan resource (*resource boundary*) yang jauh lebih ketat dibandingkan aplikasi biasa.

---

### 1.2 Topologi DaemonSet & Alur Kontrak OpenTelemetry (OTLP)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   WORKER NODE KUBERNETES (ARM64 / AMD64)                    │
│                                                                             │
│  ┌─────────────────────────┐          ┌─────────────────────────┐           │
│  │  Pod Aplikasi (Payment) │          │   Pod Aplikasi (Order)  │           │
│  │  - OTLP SDK (gRPC/HTTP) │          │  - OTLP SDK (gRPC/HTTP) │           │
│  └────────────┬────────────┘          └────────────┬────────────┘           │
│               │                                    │                        │
│               │ 1. Push OTLP (:4317/:4318)         │ 2. Tulis stdout/stderr │
│               ▼                                    ▼                        │
│  ┌───────────────────────────────────┐   ┌───────────────────────────────┐  │
│  │      GRAFANA ALLOY DAEMONSET      │   │Filesystem Host Node:          │  │
│  │                                   │   │ /var/log/pods/*/*.log         │  │
│  │  [OTLP Receiver]   [Log Reader] ◄─┼───┼─(Mount Read-Only / ro)        │  │
│  │         │               │         │   └───────────────────────────────┘  │
│  │         ▼               ▼         │                                      │
│  │  [Memory Limiter & Redaction]     │   ┌───────────────────────────────┐  │
│  │                 │                 │   │ ConfigMap Reloader Sidecar    │  │
│  │                 ▼                 │   │ (Auto-reload saat GitOps sync)│  │
│  │      [Batch & WAL Exporter]       │   └───────────────────────────────┘  │
│  └─────────────────┬─────────────────┘                                      │
└────────────────────┼────────────────────────────────────────────────────────┘
                     │ 3. Remote-Write / Push Terkompresi (Snappy/Zstd/gRPC)
                     ▼
      ┌─────────────────────────────────────────────────────┐
      │     BACKEND OBSERVABILITY (Mimir / Loki / Tempo)    │
      └─────────────────────────────────────────────────────┘
```

---

## 2. Kontrak Standar OpenTelemetry (OTLP) & Atribut Semantik

Agar Metrics, Logs, dan Traces dapat saling dihubungkan (*correlated*) secara otomatis di layar Grafana, seluruh aplikasi dan agen Alloy wajib mematuhi kontrak **OpenTelemetry Semantic Conventions**:

### 2.1 Spesifikasi Protokol & Transport OTLP

| Parameter Kontrak | Nilai Standar Produksi | Penjelasan & Alasan Teknis |
|---|---|---|
| **Protokol & Port** | - **gRPC:** Port `4317`<br>- **HTTP/Protobuf:** Port `4318` | Gunakan **gRPC (`4317`)** untuk komunikasi berkecepatan tinggi di dalam cluster karena mendukung *multiplexing* HTTP/2 dan kompresi biner Protobuf. |
| **Kompresi** | `gzip` atau `zstd` | Mengurangi pemakaian bandwidth jaringan antar node hingga 70%–80%. |
| **Header Propagasi** | **W3C Trace Context** (`traceparent`) | Format header HTTP standar (`00-<32-hex-trace-id>-<16-hex-span-id>-01`) yang diteruskan dari frontend -> API gateway -> backend microservices. |
| **Resource Attributes Wajib** | - `service.name`<br>- `service.namespace`<br>- `service.version`<br>- `deployment.environment`<br>- `k8s.cluster.name` | Lima atribut identitas utama yang memungkinkan Grafana melompat dari metrik sebuah service langsung ke log dan span trace milik versi yang sama. |

> [!CAUTION]
> **Larangan Atribut OTLP:**
> Jangan pernah memasukkan kredensial (`db.password`, `http.request.header.authorization`), query parameter mentah yang berisi token, atau nomor identitas pribadi ke dalam *Resource Attributes* maupun *Span Attributes*.

---

## 3. Hardened DaemonSet Design via Helm & GitOps

Saat men-deploy Grafana Alloy sebagai DaemonSet di cluster k3s (baik di lingkungan lab Apple Silicon ARM64 maupun server bare-metal AMD64), SRE wajib memeriksa 6 aspek pengamanan berikut:

### 3.1 Enam Pilar Keamanan DaemonSet Alloy
1. **Multi-Architecture Compatibility (`linux/arm64` & `linux/amd64`):**
   - Pastikan image resmi `grafana/alloy` ditarik menggunakan *multi-arch manifest digest* sehingga pod dapat dijadwalkan di node ARM64 (Mac/OrbStack) maupun node x86_64 tanpa error `exec format error`.
2. **Prinsip Least-Privilege RBAC (`ClusterRole`):**
   - Alloy membutuhkan izin membaca Kubernetes API untuk menemukan daftar Pod dan Node (*Service Discovery*).
   - **HANYA berikan izin `get`, `list`, dan `watch` pada resource `nodes`, `pods`, `services`, dan `endpoints`.**
   - **DILARANG KERAS** memberikan izin membaca `secrets` atau `configmaps` di seluruh namespace kepada ServiceAccount Alloy!
3. **Read-Only HostPath Volume Mounts:**
   - Untuk membaca file log container dari host node, Alloy me-mount direktori `/var/log/pods` dan `/var/lib/docker/containers`.
   - Mount tersebut **WAJIB** dipasang dengan flag **`readOnly: true`**. Agen pengumpul log tidak punya alasan apa pun untuk memodifikasi atau menghapus file di hard disk utama host!
4. **Strict CPU & Memory Limits:**
   - Karena berjalan di setiap node, kebocoran memori pada Alloy akan memengaruhi workload produksi di node yang sama. Tetapkan `requests` dan `limits` yang terukur (misal RAM limit `512Mi` dan sinkronkan dengan `otelcol.processor.memory_limiter`).
5. **Tolerations untuk Control-Plane Node:**
   - Agar node master/control-plane k3s juga terpantau log dan metriknya, tambahkan `tolerations` terhadap taint `node-role.kubernetes.io/control-plane:NoSchedule`.
6. **Dynamic Config Reload Sidecar:**
   - Gunakan sidecar `config-reloader` bawaan chart Helm agar setiap perubahan pada `ConfigMap` yang di-sync oleh ArgoCD langsung memicu HTTP POST `/-/reload` tanpa perlu me-restart seluruh pod DaemonSet.

---

### 3.2 Konfigurasi Helm Values Produksi (`values-alloy-prod.yaml`)

Berikut adalah contoh file `values.yaml` untuk Helm chart resmi `grafana/alloy` yang aman untuk di-commit ke repositori GitOps:

```yaml
fullnameOverride: "alloy"

controller:
  # Jalankan sebagai DaemonSet (1 pod per node fisik/VM)
  type: "daemonset"

  # Izinkan berjalan di node control-plane k3s agar master node ikut terpantau
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
    - key: "node-role.kubernetes.io/master"
      operator: "Exists"
      effect: "NoSchedule"

alloy:
  # Gunakan ConfigMap eksternal yang dikelola via GitOps
  configMap:
    create: false
    name: "alloy-pipeline-config"
    key: "config.alloy"

  # Buka port penerima OpenTelemetry (OTLP)
  extraPorts:
    - name: "otlp-grpc"
      port: 4317
      targetPort: 4317
      protocol: "TCP"
    - name: "otlp-http"
      port: 4318
      targetPort: 4318
      protocol: "TCP"

  # Mount direktori log host secara READ-ONLY
  mounts:
    varlog: true
    dockercontainers: false

  # Batas pemakaian CPU dan RAM per node (Aman untuk OrbStack & Bare-Metal)
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"

  # Suntikkan kredensial backend dari Kubernetes Secret (Bukan Plaintext!)
  extraEnv:
    - name: "MIMIR_TENANT_PASSWORD"
      valueFrom:
        secretKeyRef:
          name: "<APPROVED_ALLOY_BACKEND_SECRET_REF>"
          key: "mimir-basic-auth-password"

# Sidecar otomatis untuk me-reload konfigurasi saat ConfigMap berubah
configReloader:
  enabled: true
  resources:
    requests:
      cpu: "10m"
      memory: "32Mi"
    limits:
      cpu: "50m"
      memory: "64Mi"

rbac:
  create: true

serviceAccount:
  create: true
  name: "alloy-telemetry-sa"
```

---

## 4. Teknik Debugging Pipeline Tanpa Membocorkan Data Sensitif

Saat telemetri tidak muncul di Grafana, banyak engineer tergoda untuk mengaktifkan komponen `otelcol.exporter.debug` (atau `logging`) dengan level `verbosity = "detailed"`.

### 4.1 Bahaya Fatal `verbosity = "detailed"` di Produksi
Jika Anda memasang `otelcol.exporter.debug` dengan mode `detailed` sebelum komponen redaksi:
- Alloy akan mencetak **seluruh isi payload HTTP, header `Authorization`, Cookie, dan isi teks log mentah** ke dalam `stdout` pod Alloy sendiri!
- Akibatnya:
  1. Terjadi **Infinite Log Loop** (Alloy membaca log pod di node -> mencetak ulang log tersebut ke stdout miliknya -> membaca lagi log miliknya sendiri -> ukuran log berlipat ganda hingga disk penuh 100% dalam hitungan menit!).
  2. Seluruh rahasia yang seharusnya disensor justru tercetak terang-terangan di dalam log container Alloy!

### 4.2 Alur Debugging 4 Langkah yang Aman (Zero-Leakage Debugging)

```text
┌─────────────────────────┐     ┌─────────────────────────┐
│ 1. Cek Sintaks & Load   │ ──> │ 2. Cek Component Health │
│ (`alloy_config_hash`)   │     │ (Healthy vs Unhealthy)  │
└─────────────────────────┘     └────────────┬────────────┘
                                             │
┌─────────────────────────┐                  ▼
│ 4. Cek HTTP Response    │ <── ┌─────────────────────────┐
│ (`status_code="200"`)   │     │ 3. Cek Counter In/Out   │
│ (Tanpa cetak payload!)  │     │ (Received vs Exported)  │
└─────────────────────────┘     └─────────────────────────┘
```

Gunakan metrik internal Alloy untuk melacak di titik mana sinyal terhenti tanpa melihat isi data nasabah:
1. **Apakah Config Berhasil Dimuat?**
   - Periksa metrik `alloy_config_last_load_successful == 1`. Jika bernilai `0`, berarti ada kesalahan sintaks di `config.alloy`.
2. **Apakah Target Ditemukan oleh Discovery?**
   - Periksa `prometheus_sd_discovered_targets` pada endpoint `/metrics` Alloy. Jika `0`, berarti aturan `discovery.kubernetes` atau `discovery.relabel` membuang semua pod.
3. **Apakah Data Masuk ke Receiver tetapi Gagal di Exporter?**
   - Bandingkan `otelcol_receiver_accepted_spans` (jumlah span diterima) dengan `otelcol_exporter_sent_spans` (jumlah span terkirim) dan `otelcol_exporter_send_failed_spans` (jumlah span gagal).
4. **Jika Terpaksa Menggunakan `otelcol.exporter.debug`:**
   - Gunakan hanya di environment lab/disposable, letakkan **SETELAH** komponen redaksi (`otelcol.processor.attributes`), dan wajib gunakan `verbosity = "basic"` (hanya mencetak jumlah span, bukan isi datanya!).

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Penyebab Umum | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Disk Node Penuh Mendadak Setelah Deploy Alloy** | **Infinite Log Feedback Loop:** Alloy ikut menyedot file log miliknya sendiri (`/var/log/pods/monitoring_alloy-*`) sambil mengaktifkan debug logging. | Tambahkan aturan `discovery.relabel` untuk **mengecualikan (`action = "drop"`)** pod dengan label `app.kubernetes.io/name = "alloy"` dari sumber pengumpulan log `loki.source.kubernetes`! |
| **Alloy Gagal Membaca `/var/log/pods` (`Permission Denied`)** | Pod Alloy berjalan sebagai unprivileged user tanpa grup akses ke file log root di node host. | Untuk DaemonSet pembaca file log host di `/var/log/pods`, sesuaikan `securityContext` pada container utama atau gunakan mode pengumpulan via Kubernetes API (`loki.source.kubernetes`) jika kebijakan keamanan melarang akses filesystem host. |
| **Pod Alloy `CrashLoopBackOff` di Node Baru (`exec format error`)** | Image container custom atau sidecar hanya di-build untuk `linux/amd64` padahal node berjalan di atas Apple Silicon `linux/arm64`. | Periksa arsitektur image dengan `docker buildx imagetools inspect`. Gunakan image resmi multi-arch (`linux/amd64, linux/arm64`). |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. VERIFIKASI HELM CHART & MANIFES DAEMONSET ALLOY (STATIC LANE)
# ==============================================================================

# Memvalidasi sintaks chart dan values sebelum di-commit ke GitOps
helm lint ./charts/alloy -f values-alloy-prod.yaml

# Merender manifes YAML dan memeriksa apakah ada hostPath yang tidak sengaja writable
helm template alloy ./charts/alloy -f values-alloy-prod.yaml | grep -A 5 -B 2 "mountPath: /var/log"

# ==============================================================================
# 2. INSPEKSI STATUS DAEMONSET & LOG TANPA BOCORAN SECRET (RUNTIME LANE)
# ==============================================================================

# Memastikan seluruh pod DaemonSet Alloy berjalan sehat di setiap node k3s
kubectl get daemonset alloy -n monitoring -o wide

# Memeriksa apakah proses reload konfigurasi terakhir berhasil (1 = Sukses, 0 = Gagal)
kubectl exec -n monitoring daemonset/alloy -c alloy -- \
  wget -qO- http://localhost:12345/metrics | grep "alloy_config_last_load_successful"

# Membandingkan jumlah span OTLP yang diterima vs yang berhasil dikirim ke Tempo
kubectl exec -n monitoring daemonset/alloy -c alloy -- \
  wget -qO- http://localhost:12345/metrics | grep -E "otelcol_(receiver_accepted|exporter_sent|exporter_send_failed)_spans"
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa saat men-deploy Grafana Alloy sebagai **DaemonSet** untuk membaca file log container dari direktori `/var/log/pods` di host node, volume mount tersebut **wajib** dikonfigurasi dengan `readOnly: true`?
- A. Karena sistem operasi Linux tidak mengizinkan dua program membuka file yang sama secara bersamaan.
- B. Untuk menerapkan prinsip *least privilege* dan membatasi *blast radius*; agen pengumpul telemetri hanya bertugas membaca log dan dilarang keras memodifikasi, merusak, atau menghapus file di dalam sistem penyimpanan host fisik.
- C. Karena `readOnly: true` akan menghapus file log secara otomatis setiap 5 menit.
- D. Karena Helm tidak bisa merender template jika `readOnly` bernilai `false`.

#### Soal 2
Apa risiko operasional terbesar jika seorang engineer mengaktifkan `otelcol.exporter.debug` dengan `verbosity = "detailed"` pada DaemonSet Alloy di cluster produksi sementara Alloy juga sedang mengumpulkan seluruh log pod dari `/var/log/pods`?
- A. Dashboard Grafana akan berubah warna menjadi hitam putih.
- B. Terjadi **kebocoran data sensitif (PII/Secret)** ke dalam log stdout Alloy serta memicu **Infinite Log Feedback Loop** (Alloy menyedot log hasil cetakannya sendiri secara berulang tanpa henti hingga disk node penuh dan kuota Loki habis).
- C. Koneksi SSH ke seluruh server bare-metal akan menjadi lebih cepat.
- D. Server OpenTofu akan menghapus state di MinIO.

#### Soal 3
Atribut standar OpenTelemetry (*Resource Attribute*) manakah yang digunakan untuk menghubungkan metrik, baris log, dan span trace yang berasal dari aplikasi yang sama di Grafana?
- A. `http.request.header.cookie`
- B. `service.name`, `service.namespace`, dan `deployment.environment`
- C. `db.user.password`
- D. `host.mac_address`

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Anda baru saja memperbarui konfigurasi `ConfigMap` milik Grafana Alloy melalui repositori GitOps untuk menambahkan pipeline OTLP traces baru. ArgoCD menunjukkan status aplikasi `Synced` dan `Healthy`, serta seluruh pod DaemonSet Alloy berstatus `Running (2/2 Ready)`.

Namun, saat Anda membuka Grafana Explore (Tempo), tidak ada satu pun data trace yang masuk. Rekan tim Anda menyarankan untuk menjalankan perintah:
`kubectl exec -n monitoring alloy-xyz -- env` dan mencetak seluruh trafik jaringan mentah dengan `tcpdump -A` ke kanal Slack grup agar semua orang bisa membantu melihat isinya.

**Pertanyaan & Tugas Anda:**
1. Mengapa saran rekan Anda (`kubectl exec ... env` dan mem-posting dump payload mentah ke Slack) melanggar aturan keamanan SRE (*Secrets Hygiene*)?
2. Jelaskan mengapa status ArgoCD `Healthy` dan pod Alloy `Running` bisa terjadi meskipun konfigurasi baru di `ConfigMap` sebenarnya gagal dimuat oleh Alloy!
3. Sebutkan langkah-langkah diagnostik yang aman (*secret-safe*) untuk menemukan penyebab mengapa trace tidak sampai ke Tempo!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** DaemonSet berjalan di setiap node fisik/VM. Memberikan akses *read-write* ke filesystem host akan membuka celah fatal jika pod collector mengalami eksploitasi atau kesalahan konfigurasi.
2. **B.** Mencetak setiap span/log yang masuk ke `stdout` pod Alloy akan membuat file log container Alloy tumbuh secara eksponensial jika Alloy sendiri ikut memantau direktori `/var/log/pods` miliknya.
3. **B.** Konvensi semantik OpenTelemetry menetapkan `service.name`, `service.namespace`, dan `deployment.environment` sebagai identitas universal lintas sinyal telemetri.

#### Solusi Soal Kasus Nyata:
1. **Pelanggaran Keamanan (Secrets Hygiene):**
   - Menjalankan `env` di dalam pod Alloy dan membagikannya ke kanal obrolan akan membocorkan kredensial backend (`MIMIR_TENANT_PASSWORD`, token API, atau kunci enkripsi) yang disuntikkan melalui `secretKeyRef`. Demikian pula, `tcpdump -A` mentah dapat memuat token JWT dan data pribadi pelanggan.
2. **Mengapa Pod Tetap `Running` Saat Config Baru Error:**
   - Saat sidecar `config-reloader` memicu `POST /-/reload` ke Alloy, jika file konfigurasi baru mengandung kesalahan sintaks atau referensi komponen yang salah, Alloy secara defensif **menolak konfigurasi baru dan tetap berjalan menggunakan konfigurasi lama di memori** agar pengumpulan telemetri yang sudah ada tidak mati. Oleh karena itu, proses tidak crash dan pod tetap `Running`.
3. **Langkah Diagnostik yang Aman:**
   - **Langkah 1:** Cek metrik `alloy_config_last_load_successful` di `:12345/metrics`. Jika bernilai `0`, periksa log perubahan sintaks dengan `alloy fmt --test`.
   - **Langkah 2:** Jika bernilai `1`, periksa metrik `otelcol_receiver_accepted_spans`. Jika `0`, masalah ada di antara pod aplikasi dan Service Kubernetes Alloy (cek port `4317` dan variabel `OTEL_EXPORTER_OTLP_ENDPOINT` di pod aplikasi).
   - **Langkah 3:** Jika `otelcol_receiver_accepted_spans > 0` tetapi `otelcol_exporter_send_failed_spans > 0`, masalah berada pada koneksi jaringan atau autentikasi dari Alloy menuju `tempo-distributor:4317`.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai desain DaemonSet Alloy dan pipeline OpenTelemetry:
- Praktikkan di [LAB-02 — Alloy Logs dan OTLP Pipeline](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.2-alloy-telemetry-pipeline/lab/LAB-02-alloy-logs-otlp-pipeline.md).
- Lanjutkan ke [Modul 7.3 — Grafana Mimir (Long-Term Metrics TSDB)](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.3-mimir/README.md) untuk memahami arsitektur penyimpanan metrik skala besar berbasis Object Storage (MinIO/S3).
