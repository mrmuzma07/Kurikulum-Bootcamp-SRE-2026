# 01 — Custom Chart dan Values per Environment: Membangun Kontrak Multi-Lingkungan yang Konsisten

> **Ringkasan:** Di dunia enterprise, sebuah aplikasi mikroservis harus mampu berjalan di berbagai lingkungan yang berbeda karakteristiknya—mulai dari lingkungan lokal developer (OrbStack ARM64 macOS), klaster pengujian (*Staging*), hingga klaster fisik privat (*Production On-Premises*). Kesalahan paling fatal seorang pemula adalah membuat direktori chart yang berbeda untuk setiap lingkungan (*copy-paste charts*). Pola produksi SRE yang benar adalah memisahkan antara **Cetak Biru Logika Aplikasi (*Single Universal Chart*)** dan **Parameter Karakteristik Lingkungan (*Layered Values Files*)**. Dokumen ini membedah arsitektur kontrak sumber daya (*Resource Contract*), teknik pelapisan konfigurasi (*Values Layering*), stabilitas selektor abadi, serta alur promosi rilis yang aman dan deterministik.

---

## 1. Analogi Pemula: Mobil Balap F1 dengan Ban Salju vs Ban Kering

Bagi pemula, godaan terbesar ketika ingin mendeploy aplikasi ke Staging dan Production adalah membuat dua folder chart: `app-staging/` dan `app-production/`.

Mari gunakan analogi tim balap Formula 1 (F1):

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI MOBIL BALAP MODULAR vs VALUES LAYERING              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ CARA AMATIR: MEMBUAT DUA MOBIL BALAP TERPISAH ]                          │
│  - Masalah: Sirkuit balap minggu ini bersalju, minggu depan aspal kering.   │
│  - Tim amatir merancang 2 mobil balap fisik yang berbeda dari nol:          │
│    - Mobil 1 untuk Sirkuit Salju (Staging).                                 │
│    - Mobil 2 untuk Sirkuit Aspal Kering (Production).                       │
│  - Akibat: Jika ada perbaikan pada sistem rem hidrolik, tim mekanik harus   │
│    membongkar dua mobil secara manual. Risikonya, rem Mobil 2 lupa dipasang │
│    dan mobil tabrakan saat balapan sesungguhnya (**PRODUCTION OUTAGE**)!    │
│                                                                             │
│  [ CARA SRE: SATU MOBIL BALAP DENGAN PENYETELAN MODULAR ]                   │
│  - **Chart Universal (Sasis, Mesin, Suspensi, Rem):**                       │
│    Satu mobil balap yang sama persis 100%! Tidak pernah diubah-ubah!        │
│  - **Values Layering (Setelan Ban & Bahan Bakar):**                         │
│    - Di Sirkuit Salju (Staging): Mekanik menyetel ban berpaku dan pendingin │
│      ekstra (`values-staging.yaml` -> 1 replika, RAM 256MB).                │
│    - Di Sirkuit Kering (Production): Mekanik memasang ban slick balap dan   │
│      bahan bakar oktan tinggi (`values-prod.yaml` -> 5 replika, RAM 2GB).   │
│  - Mobil diuji coba di sirkuit salju; jika lolos, mobil yang SAMA PERSIS    │
│    langsung dipromosikan ke sirkuit utama!                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Kontrak Sumber Daya Produksi (The Full Resource Contract)

Sebuah Chart aplikasi tingkat produksi yang dikembangkan oleh tim platform SRE harus mampu menghasilkan sekumpulan objek Kubernetes lengkap yang saling terikat secara harmonis:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 KONTRAK OBJEK KUBERNETES APLIKASI PRODUKSI                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ 1. WORKLOAD LAYER ]                                                      │
│  - apps/v1 Deployment (Stateless) atau StatefulSet (Stateful).              │
│  - Replicas, Strategy (RollingUpdate), PodSecurityContext, SecurityContext. │
│                                                                             │
│  [ 2. NETWORKING & ROUTING LAYER ]                                          │
│  - v1 Service (ClusterIP internal port 8080).                              │
│  - networking.k8s.io/v1 Ingress (Integrasi MetalLB / Ingress Nginx TLS).    │
│                                                                             │
│  [ 3. CONFIGURATION & IDENTITY LAYER ]                                      │
│  - v1 ConfigMap (Parameter aplikasi non-sensitif).                          │
│  - v1 ServiceAccount (Identitas RBAC minimal / no auto-mount API token).    │
│                                                                             │
│  [ 4. RELIABILITY & AVAILABILITY LAYER ]                                    │
│  - policy/v1 PodDisruptionBudget (PDB: jaminan ketersediaan minimum).       │
│  - Probes: Liveness, Readiness, dan Startup Probes.                         │
│  - TopologySpreadConstraints / PodAntiAffinity (Sebar Pod antar-node host). │
│                                                                             │
│  [ 5. OBSERVABILITY & QUALITY LAYER ]                                       │
│  - monitoring.coreos.com/v1 ServiceMonitor (Scrape metrik Prometheus).      │
│  - templates/tests/test-connection.yaml (Helm Smoke Test Pod).              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Pola Pelapisan Nilai Multi-Lingkungan (Values Layering Pattern)

Prinsip dasar *Values Layering* adalah: **Nilai yang lebih spesifik akan menimpa nilai yang lebih umum (*Cascading Merge*)**.

```text
       ┌────────────────────────┐
       │      values.yaml       │  (Tingkat 1: Nilai bawaan umum yang aman)
       └───────────┬────────────┘
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
┌─────────────────┐ ┌─────────────────┐
│values-stage.yaml│ │values-prod.yaml │  (Tingkat 2: Parameter spesifik lingkungan)
└─────────────────┘ └─────────────────┘
```

### 3.1 Berkas Nilai Lingkungan Staging (`values-staging.yaml`)
Lingkungan staging didesain untuk efisiensi resource namun tetap mencerminkan fungsionalitas aplikasi:

```yaml
# values-staging.yaml - Konfigurasi Lingkungan Pengujian Staging

replicaCount: 1

image:
  repository: registry.internal.corp/bootcamp/payment-service
  tag: "v2.4.1-rc1"
  pullPolicy: Always

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "internal-ca-issuer"
  hosts:
    - host: staging-payment.internal.corp
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: staging-payment-tls
      hosts:
        - staging-payment.internal.corp

podDisruptionBudget:
  enabled: false # Staging hanya 1 replika, tidak butuh PDB

monitoring:
  enabled: true
  interval: 30s
```

### 3.2 Berkas Nilai Lingkungan Produksi (`values-prod.yaml`)
Lingkungan produksi menuntut ketersediaan tinggi, batasan resource yang ketat, dan penyebaran pod multi-node:

```yaml
# values-prod.yaml - Konfigurasi Lingkungan Produksi Bare-Metal

replicaCount: 3 # Minimal 3 replika untuk High Availability

image:
  repository: registry.internal.corp/bootcamp/payment-service
  # Di produksi, wajib menggunakan IMMUTABLE IMAGE DIGEST!
  tag: "sha256:8f9b2c1743a19e24f706e25776d6ec1438914619736c4b2b1a8d0e5124fa68b1"
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 2Gi

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "production-ca-issuer"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
  hosts:
    - host: payment.internal.corp
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: payment-tls-cert
      hosts:
        - payment.internal.corp

podDisruptionBudget:
  enabled: true
  minAvailable: 2 # Wajib minimal 2 Pod hidup setiap detik!

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - payment-service
          topologyKey: "kubernetes.io/hostname" # Sebar Pod ke server fisik berbeda!

monitoring:
  enabled: true
  interval: 15s
```

---

## 4. Stabilitas Selektor Abadi (Immutable Selectors)

Salah satu kesalahan paling sering yang melumpuhkan rilis Helm di produksi adalah modifikasi selektor Deployment.

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "internal-app.fullname" . }}
spec:
  # SELEKTOR INI BERSIFAT PERMANEN & ABADI (IMMUTABLE)!
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "internal-app.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
```

> [!CAUTION]
> **Hukum Immutability Selektor Kubernetes:**
> Sekali sebuah Deployment dibuat di klaster Kubernetes, kolom `spec.selector` **DILARANG DIUBAH**. Jika Anda menambahkan label baru ke dalam `selector.matchLabels` pada rilis berikutnya (misal: menambahkan `app.kubernetes.io/version`), Kubernetes API akan menolak perintah upgrade dengan error:
> `The Deployment "payment" is invalid: spec.selector: Invalid value: field is immutable`
> Menggunakan flag `--force` sebagai jalan pintas akan **MENGHANCURKAN DAN MEMBUAT ULANG POD SECARA MENDADAK**, memicu pemadaman layanan (*Total Service Outage*)! Label versi hanya boleh ditaruh di `metadata.labels` atau `spec.template.metadata.labels`, BUKAN di `selector`!

---

## 5. Alur Kerja Promosi Rilis Antar-Lingkungan (Promotion Workflow)

Bagaimana sebuah rilis dipromosikan dari Staging ke Production secara aman dan terverifikasi?

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR PROMOSI RILIS DETERMINISTIK SRE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TAHAP 1: ARTIFACT BUILDING & PACKAGING ]                                 │
│  - Kode aplikasi di-build menjadi Docker Image dengan Digest SHA256.        │
│  - Chart dikemas dan di-push ke GitLab OCI Registry:                        │
│    `oci://registry.gitlab.com/charts/payment-service:1.2.0`                 │
│                                     │                                       │
│                                     ▼                                       │
│                                                                             │
│  [ TAHAP 2: DEPLOY & TEST DI LINGKUNGAN STAGING ]                           │
│  - Deploy: `helm upgrade payment oci://... -f values-staging.yaml --wait`   │
│  - Eksekusi Pengujian: `helm test payment -n staging`                       │
│  - Automated Integration & Load Tests berjalan selama 30 menit.             │
│                                     │                                       │
│                                     ▼ LULUS 100%                            │
│                                                                             │
│  [ TAHAP 3: PERSETUJUAN RILIS PRODUKSI (4-EYES APPROVAL) ]                  │
│  - Lead SRE meninjau perbedaan manifest render:                             │
│    `helm diff upgrade payment oci://... -f values-prod.yaml`                │
│  - Memastikan PDB, toleransi kapasitas node fisik, dan image digest cocok.  │
│                                     │                                       │
│                                     ▼ DISETUJUI                             │
│                                                                             │
│  [ TAHAP 4: DEPLOY PRODUKSI DENGAN ATOMIC ROLLOUT ]                         │
│  - Deploy: `helm upgrade payment oci://... -f values-prod.yaml --atomic`    │
│  - Pantau metrik latensi RED & error budget burn rate di Grafana.           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA MASALAH VALUES & MULTI-ENV                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: POD PENDING KARENA KAPASITAS NODE FISIK KURANG ]                │
│  Gejala: Staging sukses berjalan, tapi di Production Pod berstatus Pending: │
│          "0/3 nodes are available: 3 Insufficient memory".                 │
│  Akar Masalah:                                                              │
│  - Nilai `resources.requests.memory: 2Gi` pada `values-prod.yaml` terlalu   │
│    besar untuk kapasitas node k3s on-prem yang sedang menampung banyak Pod. │
│  Solusi SRE:                                                                │
│  1. Hitung alokasi memori aktual node: `kubectl describe nodes | grep Alloc`│
│  2. Sesuaikan nilai `requests` secara realistis dan aktifkan HPA            │
│     (Horizontal Pod Autoscaler) untuk scale-out dinamis.                    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "STATEFULSET ILLUSION" (SALAH MEMILIH JENIS WORKLOAD) ]         │
│  Gejala: Aplikasi lambat saat startup dan Pod tidak bisa di-reschedule.     │
│  Akar Masalah:                                                              │
│  - Engineer menggunakan `StatefulSet` untuk aplikasi web biasa hanya karena │
│    aplikasi tersebut menyambung ke database Postgres luar klaster.          │
│  Solusi SRE:                                                                │
│  Aplikasi yang tidak menyimpan status di disk lokal adalah **STATELESS**!   │
│  Gunakan selalu `Deployment` biasa agar replika Pod dapat berpindah host    │
│  secara cepat dan fleksibel saat terjadi node failure.                      │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: INGRESS HOSTNAME CONFLICT ANTAR-LINGKUNGAN ]                    │
│  Gejala: Lalu lintas pengguna staging bocor ke rilis produksi atau 404.     │
│  Akar Masalah:                                                              │
│  - Developer lupa mengganti host Ingress di `values-staging.yaml` sehingga  │
│    kedua lingkungan mendaftarkan hostname yang sama ke Ingress Controller!  │
│  Solusi SRE:                                                                │
│  Wajibkan aturan validasi di `values.schema.json` atau gunakan templating   │
│  otomatis berbasis release name:                                            │
│  host: "{{ .Release.Namespace }}-payment.internal.corp"                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menampilkan perbedaan manifest antara Staging dan Production secara visual
helm template payment ./internal-app -f values-staging.yaml > /tmp/staging.yaml
helm template payment ./internal-app -f values-prod.yaml > /tmp/prod.yaml
diff -u /tmp/staging.yaml /tmp/prod.yaml

# 2. Menginspeksi nilai values yang aktif pada rilis tertentu di klaster
helm get values payment -n production

# 3. Menguji rendering manifest produksi lengkap dengan verifikasi schema
helm lint ./internal-app -f values-prod.yaml

# 4. Melakukan simulasi upgrade tanpa menyentuh klaster (Server-Side Dry Run)
helm upgrade payment ./internal-app -f values-prod.yaml --dry-run=server -n production

# 5. Mengecek status penyebaran Pod di berbagai host fisik node
kubectl get pods -n production -o wide -l app.kubernetes.io/name=payment-service
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa membuat dua direktori Chart terpisah (`chart-staging/` dan `chart-prod/`) untuk aplikasi yang sama dianggap sebagai *anti-pola* berbahaya dalam rekayasa keandalan SRE?
- A. Karena Helm menolak membaca direktori yang berakhiran kata `-prod`.
- B. Melanggar prinsip DRY (*Don't Repeat Yourself*), melipatgandakan beban pemeliharaan kode, dan menciptakan risiko *configuration drift* di mana perbaikan bug di staging lupa diterapkan di production.
- C. Karena ukuran repositori Git akan bertambah 50 Megabyte.
- D. Agar aplikasi tidak bisa diakses dari browser.

*Kunci Jawaban:* **B**. Praktik terbaik SRE mewajibkan satu Chart universal (*single source of truth*) yang perilakunya disesuaikan menggunakan berkas *layered values*.

### Soal 2 (Pilihan Ganda)
Apa yang akan terjadi jika Anda mengubah nilai pada kolom `spec.selector.matchLabels` di dalam template `deployment.yaml` saat melakukan `helm upgrade` pada rilis produksi yang sedang aktif?
- A. Kubernetes otomatis mengubah nama Pod dalam waktu 1 detik.
- B. Kubernetes API menolak pembaruan tersebut dengan pesan error `field is immutable`, karena selektor Deployment tidak boleh diubah sepanjang masa hidup objek tersebut.
- C. Seluruh cluster k3s otomatis mati.
- D. Helm otomatis mengubah Deployment menjadi DaemonSet.

*Kunci Jawaban:* **B**. Selektor Deployment adalah pengidentifikasi permanen Pod; mengubah selektor mewajibkan penghapusan objek atau strategi migrasi bertahap.

### Soal 3 (Pilihan Ganda)
Dalam berkas `values-prod.yaml`, mengapa tim SRE mengonfigurasi aturan `podAntiAffinity` dengan topology key `kubernetes.io/hostname`?
- A. Agar Pod hanya berjalan di mesin laptop developer.
- B. Memastikan Kubernetes menjadwalkan replika Pod ke server fisik (*node*) yang berbeda-beda, sehingga jika salah satu server fisik mati, aplikasi tetap hidup di server lain (*High Availability*).
- C. Untuk menghemat pemakaian listrik data center.
- D. Agar Pod tidak bisa saling berkirim pesan jaringan.

*Kunci Jawaban:* **B**. `podAntiAffinity` mendistribusikan replika Pod ke berbagai domain kegagalan (*failure domains*) demi menjamin ketersediaan layanan saat terjadi insiden fisik node.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Anda adalah Lead SRE yang meninjau berkas `values-prod.yaml` yang diajukan oleh tim developer. Di dalam berkas tersebut tertulis:
```yaml
replicaCount: 1
resources: {}
podDisruptionBudget:
  enabled: false
```
Jelaskan mengapa konfigurasi di atas merupakan "bom waktu" bagi keandalan sistem produksi, dan susun perbaikan konfigurasi nilai yang memenuhi standar *Zero Downtime Architecture*!

*Panduan Solusi:*
1. **Analisis Bahaya Keandalan:**
   - **`replicaCount: 1` & Ketiadaan PDB:** Memiliki hanya 1 replika tanpa PDB berarti aplikasi memiliki *Single Point of Failure* (SPOF). Jika node tempat Pod berjalan mengalami pemeliharaan rutin (*rolling OS patch / drain*), aplikasi akan mati total dan pengguna mengalami *HTTP 502 Bad Gateway*.
   - **`resources: {}` (Tanpa Batas CPU & Memori):** Ketiadaan `requests` membuat Kubernetes scheduler tidak tahu beban aplikasi, berisiko menempatkan Pod di node yang sudah penuh. Ketiadaan `limits` memungkinkan aplikasi menghabiskan seluruh RAM node jika terjadi *memory leak*, yang akan memicu Linux OOM-Killer mematikan proses penting sistem (seperti Kubelet atau MetalLB speaker).
2. **Konfigurasi Produksi Baku SRE:**
   ```yaml
   replicaCount: 3 # Minimal 3 replika
   
   resources:
     requests:
       cpu: 200m
       memory: 256Mi
     limits:
       cpu: "1"
       memory: 1Gi
       
   podDisruptionBudget:
     enabled: true
     minAvailable: 2 # Menjamin minimal 2 Pod tetap melayani trafik saat maintenance
   ```

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Modul 5.1: 01 — Struktur Chart, Values, dan Go-template](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/01-struktur-chart-values-template.md) — Memahami sintaksis dan helpers chart.
  - [Modul 5.1: 03 — Chart Repository dan OCI di GitLab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/03-repository-chart-oci-gitlab.md) — Mengambil chart universal dari OCI registry.
- **Topik Berikutnya:**
  - [02 — Hooks, Tests, NOTES, dan Siklus Upgrade](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/02-hooks-tests-notes-upgrade.md) — Mengotomasi migrasi database dan pengujian rilis dengan Helm Hooks.
  - [03 — Security, Reliability, dan Observability Preparation](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/03-production-chart-security-reliability.md) — Penguatan postur keamanan kontainer dan integrasi telemetri.
