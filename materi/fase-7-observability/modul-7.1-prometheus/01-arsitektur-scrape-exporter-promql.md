# 01 — Arsitektur Scrape, Exporter, dan PromQL

> **Ringkasan:** Memahami secara menyeluruh cara kerja mesin observabilitas **Prometheus** mulai dari mekanisme penarikan data (*pull-based scraping*), penyimpanan deret waktu (*Time-Series Database / TSDB*), penggunaan **Exporter** di lingkungan bare-metal on-premises, hingga penguasaan bahasa kueri **PromQL** dari tingkat dasar hingga mahir.

---

## 1. Konsep & Arsitektur Dasar Prometheus

### 1.1 Analogi Dunia Nyata: "Dokter Keliling yang Mencatat Tanda Vital Pasien (Pull-Based Checkup)"
Bayangkan sebuah rumah sakit besar dengan ratusan pasien di ruang rawat inap:
- **Pola Push (Pasien Lapor Sendiri):** Setiap pasien diminta menelepon meja perawat setiap menit untuk melaporkan detak jantung dan suhu tubuhnya. Masalahnya: jika ada pasien yang pingsan (server mati/hang), dia tidak bisa menelepon! Selain itu, jika 1.000 pasien menelepon di detik yang sama, telepon meja perawat akan macet total (*network flood*).
- **Pola Pull — Cara Kerja Prometheus (Dokter Keliling):** Perawat/dokter keliling mendatangi satu per satu tempat tidur pasien secara terjadwal setiap 15 detik (*scrape interval*), melihat layar monitor di samping kasur (*endpoint `/metrics`*), lalu mencatat angkanya ke buku rekam medis berurutan berdasarkan waktu (*Time-Series Database*). Jika saat didatangi pasien tidak merespons, dokter langsung mencatat di bukunya: `up = 0` (Pasien Kritis/Target Down!).

Di dunia SRE, **Prometheus** bekerja persis seperti dokter keliling tersebut. Ia secara aktif mengunjungi (*scrape*) daftar alamat target melalui protokol HTTP, mengambil teks metrik mentah, memberi stempel waktu (*timestamp*), dan menyimpannya ke dalam penyimpanan lokal yang sangat cepat.

---

### 1.2 Arsitektur Komponen Internal Prometheus

Berikut adalah diagram alur data di dalam ekosistem Prometheus pada lingkungan Kubernetes (k3s) dan server bare-metal:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARSITEKTUR EKOSISTEM PROMETHEUS                          │
│                                                                             │
│  [ Target Bare-Metal ]         [ Target Kubernetes Pod ]                    │
│  ┌───────────────────┐         ┌─────────────────────────┐                  │
│  │   node_exporter   │         │ Aplikasi Bisnis (Go/Py) │                  │
│  │  :9100/metrics    │         │     :8080/metrics       │                  │
│  └─────────▲─────────┘         └────────────▲────────────┘                  │
│            │                                │                               │
│            └──────────────┐  ┌──────────────┘                               │
│                           │  │ (1. HTTP GET Scrape setiap 15s)              │
│  ┌────────────────────────┴──┴───────────────────────────────────────────┐  │
│  │                        PROMETHEUS SERVER                              │  │
│  │                                                                       │  │
│  │  ┌──────────────────────┐        ┌─────────────────────────────────┐  │  │
│  │  │ Service Discovery    │ ─────> │Retrieval / Scrape Engine        │  │  │
│  │  │ (K8s API / Static)   │        │(Relabeling & Target Validation) │  │  │
│  │  └──────────────────────┘        └────────────────┬────────────────┘  │  │
│  │                                                   │                   │  │
│  │                                                   ▼                   │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │ Local Storage (TSDB - Time Series Database)                     │  │  │
│  │  │  ├─ Head Block (In-Memory RAM) <── Write-Ahead Log (WAL Disk)   │  │  │
│  │  │  └─ Immutable Blocks (2-hour chunks compacted on SSD/NVMe)      │  │  │
│  │  └──────────────────────┬─────────────────────────┬────────────────┘  │  │
│  │                         │                         │                   │  │
│  │                         ▼                         ▼                   │  │
│  │  ┌──────────────────────────────┐  ┌───────────────────────────────┐  │  │
│  │  │ Rule & Alert Evaluator       │  │ PromQL Query Engine           │  │  │
│  │  └──────────────┬───────────────┘  └──────────────▲────────────────┘  │  │
│  └─────────────────┼─────────────────────────────────┼───────────────────┘  │
│                    │ (2. Push Firing Alerts)         │ (3. Query PromQL)    │
│                    ▼                                 │                      │
│          ┌───────────────────┐             ┌─────────┴─────────┐            │
│          │   Alertmanager    │             │  Grafana / Web UI │            │
│          └───────────────────┘             └───────────────────┘            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Terminologi Inti: Target, Job, Instance, dan Time-Series
1. **Target:** Endpoint jaringan (IP + Port + Path, misal `192.168.10.21:9100/metrics`) yang menyediakan data metrik dalam format teks standar Prometheus.
2. **Instance:** Label otomatis (`instance="192.168.10.21:9100"`) yang mengidentifikasi satu target spesifik secara unik.
3. **Job:** Kumpulan beberapa *instance* yang memiliki fungsi atau peran yang sama (misal `job="node-exporter"` untuk kumpulan 10 server fisik Linux, atau `job="payment-service"` untuk 5 replika pod pembayaran).
4. **Sample & Time-Series:** Satu baris metrik terdiri dari **Nama Metrik + Kumpulan Label (Key-Value) + Nilai Angka (Float64) + Stempel Waktu (Int64 Milliseconds)**:
   ```text
   http_requests_total{job="payment-api", instance="10.42.1.15:8080", method="POST", status="200"}  14592   1729156800000
   └────────┬────────┘ └─────────────────────────────────────┬───────────────────────────────────┘  └──┬──┘ └──────┬──────┘
      Metric Name                         Labels (Dimensi Identitas)                                 Nilai     Timestamp
   ```

> [!WARNING]
> **Jebakan Pemula SRE: `up == 1` Bukan Berarti Aplikasi Sehat!**
> Setiap kali Prometheus melakukan scrape, ia otomatis membuat metrik internal bernama `up{job="...", instance="..."}`. Jika scrape berhasil, nilainya `1`; jika gagal/timeout, nilainya `0`. Ingat: **`up == 1` hanya membuktikan bahwa web server metrik bisa dihubungi oleh Prometheus**, bukan bukti bahwa aplikasi bisnis berhasil memproses transaksi nasabah atau koneksi databasenya normal!

---

## 2. Empat Tipe Metrik Prometheus & Arsitektur TSDB

Memilih tipe metrik yang salah akan membuat rumus kueri dan grafik dashboard Anda menampilkan informasi yang menyesatkan.

### 2.1 Empat Tipe Metrik Standar

| Tipe Metrik | Karakteristik Nilai | Contoh Metrik Nyata | Fungsi PromQL Utama |
|---|---|---|---|
| **1. Counter** | Nilai **hanya bisa naik** (kumulatif sejak aplikasi menyala) dan hanya kembali ke `0` jika proses aplikasi restart. | - `http_requests_total`<br>- `node_network_receive_bytes_total`<br>- `payment_errors_total` | `rate()`, `irate()`, `increase()`<br>*(Jangan pernah mem-plot Counter mentah langsung ke grafik!)* |
| **2. Gauge** | Nilai **bisa naik dan turun** secara bebas seperti jarum spidometer atau termometer suhu saat ini. | - `node_memory_MemAvailable_bytes`<br>- `go_goroutines`<br>- `queue_pending_jobs` | `avg()`, `max()`, `min()`, `deriv()`, `predict_linear()` |
| **3. Histogram** | Mengelompokkan observasi (seperti durasi latensi atau ukuran request) ke dalam ember batas atas (*buckets* `le` / *less-than-or-equal*). Menghasilkan 3 series otomatis: `_bucket`, `_sum`, dan `_count`. | - `http_request_duration_seconds_bucket{le="0.1"}`<br>- `http_request_duration_seconds_sum`<br>- `http_request_duration_seconds_count` | `histogram_quantile(0.95, sum by (le) (rate(..._bucket[5m])))` |
| **4. Summary** | Menghitung persentil (`quantile="0.95"`) langsung di sisi aplikasi klien sebelum dikirim ke Prometheus. | - `jvm_gc_pause_seconds{quantile="0.99"}` | **Hindari untuk cluster multi-pod!** Nilai persentil dari Summary **tidak bisa digabungkan (di-rata-rata/dijumlahkan)** lintas pod. Gunakan Histogram! |

### 2.2 Bagaimana TSDB Menyimpan Data di Disk Lokal?
Prometheus menyimpan sampel ke dalam penyimpanan lokal menggunakan struktur **LSM-Tree / Time-Partitioned Blocks**:
1. **Write-Ahead Log (WAL) & Head Block:** Saat sampel baru masuk dari proses scrape, data ditulis ke **Head Block** di dalam memori RAM agar sangat cepat, sekaligus dicatat ke file **WAL** di disk. Jika listrik server mati mendadak, Prometheus membaca ulang WAL saat menyala agar data 2 jam terakhir tidak hilang.
2. **Block Compaction (Setiap 2 Jam):** Setiap 2 jam, data di Head Block dibekukan (*flushed*) menjadi satu direktori blok *immutable* di disk (berisi file index dan chunks terkompresi). Background process kemudian menggabungkan (*compaction*) blok-blok kecil menjadi blok yang lebih besar dan menghapus blok yang sudah melewati batas usia simpan (`--storage.tsdb.retention.time=15d`).

---

## 3. Exporter di Lingkungan On-Premises & Konfigurasi Scrape

Tidak semua perangkat lunak (seperti kernel Linux, switch jaringan, database PostgreSQL lama, atau endpoint web pihak ketiga) mampu menyediakan format metrik Prometheus secara bawaan. Untuk itu kita menggunakan **Exporter**—penerjemah kecil yang mengambil statistik sistem lalu menyajikannya di endpoint `/metrics`.

### 3.1 Dua Exporter Wajib bagi SRE On-Premises
1. **`node_exporter` (Port 9100):**
   - Dipasang di setiap server fisik bare-metal atau VM Linux (via Ansible di Fase 4 atau DaemonSet di K8s).
   - Mengekspos metrik hardware & OS: utilisasi CPU per core (`node_cpu_seconds_total`), pemakaian RAM (`node_memory_*`), kapasitas partisi hard disk (`node_filesystem_*`), antrean I/O disk (`node_disk_*`), dan saturasi kartu jaringan (`node_network_*`).
2. **`blackbox_exporter` (Port 9115):**
   - Mengukur ketersediaan layanan dari sudut pandang pengguna luar (*active synthetic probing*) menggunakan protokol **HTTP, HTTPS, TCP, DNS, dan ICMP (Ping)**.
   - Sangat krusial di on-prem untuk memantau: apakah VIP MetalLB merespons HTTP 200, apakah DNS internal CoreDNS bisa me-resolve domain lokal, dan **berapa hari lagi sertifikat TLS/SSL internal akan kedaluwarsa** (`probe_ssl_earliest_cert_expiry`).

---

### 3.2 Konfigurasi `prometheus.yml` dengan Relabeling yang Aman

Dalam produksi, kita menggunakan **Relabeling** untuk menyaring target dan membatasi label agar tidak mencemari database dengan label sampah.

```yaml
global:
  scrape_interval: 15s      # Standar industri: tarik metrik setiap 15 detik
  scrape_timeout: 10s       # Batalkan jika target tidak merespons dalam 10 detik
  evaluation_interval: 15s  # Evaluasi recording & alerting rules setiap 15 detik
  external_labels:
    cluster: "k3s-onprem-prod-01"
    sre_environment: "production"

scrape_configs:
  # 1. Scrape Node Exporter pada Server Bare-Metal / VM
  - job_name: "node-exporter-baremetal"
    static_configs:
      - targets:
          - "192.168.50.11:9100"
          - "192.168.50.12:9100"
          - "192.168.50.13:9100"
        labels:
          sre_owner: "platform-infra"
          hardware_arch: "arm64"

  # 2. Scrape Blackbox Exporter untuk Uji HTTP & Sertifikat TLS
  - job_name: "blackbox-http-probe"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - "https://argocd.internal.example.com"
          - "https://gitlab.internal.example.com"
    relabel_configs:
      # Salin alamat URL target ke parameter ?target=... milik blackbox_exporter
      - source_labels: [__address__]
        target_label: __param_target
      # Simpan URL asli sebagai label 'instance' di hasil metrik
      - source_labels: [__param_target]
        target_label: instance
      # Arahkan koneksi HTTP scrape sebenarnya ke alamat pod blackbox_exporter
      - target_label: __address__
        replacement: "blackbox-exporter.monitoring.svc.cluster.local:9115"

  # 3. Kubernetes Pod Discovery dengan Allowlist Ketat
  - job_name: "kubernetes-pods-allowlisted"
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - production
            - staging
    relabel_configs:
      # HANYA scrape pod yang memiliki anotasi eksplisit: prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      # Gunakan hanya label yang disetujui (Allowlist), JANGAN gunakan action: labelmap tanpa filter!
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
        target_label: service
```

> [!CAUTION]
> **Aturan Higienitas Rahasia (*Secrets Hygiene*) pada Konfigurasi Prometheus:**
> Jika target scrape memerlukan autentikasi (`basic_auth`, `bearer_token`, atau `tls_config.key_file`), **DILARANG KERAS** menuliskan password atau token secara plaintext di dalam `prometheus.yml`! Gunakan fitur `basic_auth.password_file: /etc/prometheus/secrets/<SECRET_FILE_REF>` atau `authorization.credentials_file` yang di-mount dari Kubernetes Secret terenkripsi (SealedSecret / Vault).

---

## 4. Menguasai PromQL dari Nol hingga Mahir

**PromQL (Prometheus Query Language)** adalah bahasa fungsional untuk memilih, menghitung laju perubahan, dan mengelompokkan data deret waktu.

### 4.1 Dua Tipe Data Dasar di PromQL
1. **Instant Vector:** Mengambil **satu nilai terbaru** per series pada satu titik waktu tertentu.
   - Contoh: `node_memory_MemAvailable_bytes{namespace="production"}`
   - *Kegunaan:* Menampilkan angka saat ini atau membandingkan kondisi terbaru.
2. **Range Vector:** Mengambil **sekumpulan nilai historis** dalam rentang jendela waktu tertentu (ditandai dengan kurung siku `[5m]`, `[1h]`).
   - Contoh: `http_requests_total{service="payment-api"}[5m]`
   - *Catatan:* Range vector tidak bisa langsung digambar di grafik tanpa dimasukkan ke fungsi pengolah seperti `rate()` atau `avg_over_time()`.

---

### 4.2 Perbedaan Kritis: `rate()` vs `irate()` vs `increase()`

Saat mengolah metrik bertipe **Counter**, pemula sering bingung memilih antara `rate()` dan `irate()`:

```text
Data Sampel Counter (Setiap 15 detik dalam jendela 1 menit):
Detik:   0s     15s     30s     45s     60s
Nilai:  [100]  [105]   [110]   [112]   [160]  <-- Lonjakan tajam di 15 detik terakhir!

1. rate(metric[1m]):
   Mengambil selisih titik PERTAMA (0s) dan TERAKHIR (60s) dibagi durasi (60s):
   = (160 - 100) / 60 detik = 1.0 request/detik (Meratakan lonjakan/Halus)

2. irate(metric[1m]):
   Mengambil selisih DUA TITIK TERAKHIR SAJA (45s dan 60s) dibagi selisih waktunya (15s):
   = (160 - 112) / 15 detik = 3.2 request/detik (Sangat reaktif/Tajam/Noisy)
```

- **Aturan Emas SRE:** Selalu gunakan **`rate()`** (dengan jendela minimal 4x `scrape_interval`, misal `[1m]` atau `[5m]`) untuk **Alerting Rules** dan **Perhitungan SLO** agar tidak memicu *false alarm* akibat lonjakan sesaat. Gunakan `irate()` hanya saat melakukan investigasi visual interaktif pada rentang waktu sempit.

---

### 4.3 Empat Rumus PromQL Wajib Kuasai bagi SRE

Berikut adalah 4 kueri standar produksi beserta bedah anatomi rumusnya:

#### 1. Persentase Utilisasi CPU per Server/Node
```promql
100 * (
  1 - avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle", sre_environment="production"}[5m])
  )
)
```
*Penjelasan:* Kernel Linux mencatat berapa detik CPU menganggur (`mode="idle"`). `rate(...[5m])` menghasilkan rasio waktu menganggur dari `0.0` hingga `1.0` per core. `avg by (instance)` merata-ratakan seluruh core CPU di satu server. Dikurangi dari `1` lalu dikali `100` menghasilkan persentase CPU yang sibuk (`0%` - `100%`).

#### 2. Persentase Pemakaian Memori RAM Aktual
```promql
100 * (
  1 - (
    node_memory_MemAvailable_bytes{sre_environment="production"}
    /
    node_memory_MemTotal_bytes{sre_environment="production"}
  )
)
```
*Penjelasan:* Jangan gunakan `MemFree_bytes` karena Linux menggunakan RAM kosong untuk disk cache/buffer yang bisa dilepas kapan saja. Selalu gunakan **`MemAvailable_bytes`** dibagi **`MemTotal_bytes`**.

#### 3. Rasio Error HTTP 5xx per Service (Dengan Penanganan Pembagi Nol)
```promql
sum by (service) (
  rate(http_requests_total{status=~"5..", sre_environment="production"}[5m])
)
/
(
  sum by (service) (
    rate(http_requests_total{sre_environment="production"}[5m])
  ) > 0
)
```
*Penjelasan:* Menghitung jumlah request gagal (`500`–`599`) dibagi seluruh request yang masuk. Filter `> 0` pada penyebut mencegah hasil `NaN` (*Not a Number*) saat layanan sedang tidak menerima trafik sama sekali di tengah malam.

#### 4. Latensi Persentil ke-95 (p95 Latency) dari Histogram
```promql
histogram_quantile(
  0.95,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket{sre_environment="production"}[5m])
  )
)
```
*Penjelasan:* **Wajib menyertakan label `le` di dalam klausa `sum by (le, ...)`!** Jika label `le` (*less-than-or-equal bucket*) dihilangkan saat melakukan `sum`, fungsi `histogram_quantile` tidak tahu batas keranjang distribusinya dan akan mengembalikan hasil kosong atau error!

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Penyebab Umum | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Status Target `DOWN` (`up == 0`)** | 1. Proses exporter mati.<br>2. Firewall (`ufw` / NetworkPolicy) memblokir port.<br>3. `scrape_timeout` terlalu pendek untuk exporter yang berat. | 1. Dari pod Prometheus atau node, uji konektivitas:<br>`curl -v http://<TARGET_IP>:9100/metrics`<br>2. Cek durasi scrape via metrik `scrape_duration_seconds`. |
| **Hasil Kueri PromQL Kosong (`Empty query result`)** | 1. Salah ketik nama label atau nilai label (`env="prod"` vs `sre_environment="production"`).<br>2. Jam sistem (*system clock*) antara node dan server Prometheus selisih jauh (*clock skew*).<br>3. Rentang waktu `[15s]` terlalu sempit (hanya memuat 1 sampel, sehingga `rate()` gagal menghitung selisih). | 1. Cek daftar label asli dengan menjalankan kueri nama metrik saja: `http_requests_total`.<br>2. Pastikan daemon NTP/`chronyd` aktif di semua node (`timedatectl status`).<br>3. Gunakan jendela minimal **4x scrape_interval** (misal `[1m]` untuk interval `15s`). |
| **Grafik `histogram_quantile` Bernilai `NaN` atau Datar di Batas Maksimal** | 1. Tidak ada trafik masuk pada jendela waktu tersebut.<br>2. Seluruh request lebih lambat daripada bucket `le` terbesar yang didefinisikan developer (sehingga semua masuk ke `+Inf`). | 1. Inspeksi distribusi bucket mentah:<br>`rate(http_request_duration_seconds_bucket[5m])`.<br>2. Minta developer menyesuaikan batas bucket histogram di kode aplikasi agar sesuai dengan profil latensi nyata. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. VALIDASI KONFIGURASI PROMETHEUS (STATIC CHECK)
# ==============================================================================

# Memvalidasi sintaks prometheus.yml sebelum di-reload ke server/cluster
promtool check config /etc/prometheus/prometheus.yml

# Memeriksa kesesuaian format teks metrik dari output exporter secara langsung
curl -s http://localhost:9100/metrics | promtool check metrics

# ==============================================================================
# 2. INSPEKSI TARGET & RUNTIME VIA HTTP API PROMETHEUS
# ==============================================================================

# Menampilkan daftar seluruh target yang sedang berstatus DOWN via API
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health != "up") | {job: .labels.job, instance: .labels.instance, lastError: .lastError}'

# Mengeksekusi instant query PromQL langsung dari terminal CLI
curl -sG http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up{sre_environment="production"}' | jq '.data.result'

# Memicu reload konfigurasi Prometheus secara aman tanpa mematikan proses (Zero-Downtime)
# (Memerlukan flag --web.enable-lifecycle pada proses Prometheus)
curl -X POST http://localhost:9090/-/reload
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Seorang engineer ingin membuat grafik laju penambahan error HTTP per detik dari metrik `http_requests_total{status="500"}` yang bertipe **Counter**, lalu menggunakannya sebagai dasar pemicu **Alerting Rule**. Fungsi dan jendela waktu manakah yang paling tepat digunakan jika `scrape_interval` adalah `15s`?
- A. Mem-plot `http_requests_total{status="500"}` secara langsung tanpa fungsi apa pun.
- B. Menggunakan `irate(http_requests_total{status="500"}[15s])` karena hanya membutuhkan 1 titik sampel.
- C. Menggunakan `rate(http_requests_total{status="500"}[5m])` karena meratakan fluktuasi sesaat, tahan terhadap counter reset, dan memuat cukup titik sampel.
- D. Menggunakan `histogram_quantile(0.95, http_requests_total{status="500"})`.

#### Soal 2
Mengapa saat menghitung latensi persentil ke-99 (`p99`) dari sebuah aplikasi yang berjalan di 10 replika Pod Kubernetes, SRE **wajib** menggunakan metrik bertipe **Histogram** (`_bucket`) dan **bukan Summary**?
- A. Karena tipe Summary tidak didukung oleh bahasa pemrograman Go dan Python.
- B. Karena nilai kuantil yang sudah dihitung di sisi klien pada tipe Summary tidak dapat diagregasi (dijumlahkan atau dirata-rata secara matematis) lintas replika Pod, sedangkan bucket pada Histogram dapat dijumlahkan dengan `sum by (le) (rate(...))` sebelum dihitung persentilnya.
- C. Karena Histogram tidak memakan ruang penyimpanan di dalam TSDB Prometheus.
- D. Karena Summary hanya bisa digunakan untuk mengukur kapasitas hard disk.

#### Soal 3
Apa yang terjadi jika seorang engineer menulis kueri `histogram_quantile(0.95, sum by (service) (rate(http_request_duration_seconds_bucket[5m])))` tanpa menyertakan label `le` di dalam klausa `sum by (...)`?
- A. Prometheus otomatis menambahkan label `le` di belakang layar.
- B. Kueri gagal menghasilkan nilai persentil karena informasi batas atas ember distribusi (`le`) hilang tergabung menjadi satu nilai tunggal.
- C. Server Prometheus akan langsung crash akibat kehabisan memori.
- D. Nilai yang dihasilkan menjadi 100 kali lebih akurat.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Anda sedang bertugas sebagai SRE On-Call. Manajer produk melaporkan bahwa pengguna tidak bisa melakukan *checkout* barang di aplikasi `order-service` dan menerima pesan *504 Gateway Timeout*. Namun, saat Anda memeriksa dashboard infrastruktur dasar, indikator status target Prometheus menampilkan `up{job="order-service"} == 1` berwarna hijau terang.

**Pertanyaan & Tugas Anda:**
1. Jelaskan secara teknis mengapa metrik `up == 1` tetap berwarna hijau padahal layanan transaksi *checkout* sedang lumpuh bagi pengguna!
2. Sebutkan 2 metrik/metode pemantauan tambahan yang harus Anda pasang agar kegagalan fungsional seperti ini langsung terdeteksi di masa depan!
3. Tuliskan kueri PromQL untuk mendeteksi persentase request yang mengalami status `5xx` pada `order-service` selama 5 menit terakhir!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **C.** Fungsi `rate()` membutuhkan minimal 2 sampel (idealnya >= 4x interval scrape) untuk menghitung rata-rata laju peningkatan per detik dan secara otomatis menangani kondisi ketika proses aplikasi restart (*counter reset* kembali ke 0).
2. **B.** Persentil dari berbagai populasi sampel tidak sah secara statistik jika dirata-ratakan (`avg(p99_pod1, p99_pod2)` adalah salah!). Dengan Histogram, kita menjumlahkan jumlah kejadian per keranjang waktu (`sum by (le)`) dari seluruh pod terlebih dahulu, baru menghitung persentil gabungannya.
3. **B.** Fungsi `histogram_quantile()` mewajibkan keberadaan label `le` (*less than or equal*) pada vektor input untuk menginterpolasi posisi persentil di antara batas-batas bucket.

#### Solusi Soal Kasus Nyata:
1. **Mengapa `up == 1` Tetap Hijau:**
   - Metrik `up == 1` hanya menandakan bahwa goroutine/thread HTTP handler `/metrics` di dalam pod `order-service` berhasil menjawab panggilan HTTP GET dari Prometheus. Jika koneksi aplikasi ke database utama mati atau terjadi *deadlock* hanya pada fungsi `/checkout`, endpoint `/metrics` tetap merespons `200 OK` dengan lancar.
2. **Dua Metode Pemantauan Tambahan:**
   - **Instrumentasi Metrik Aplikasi (RED Metrics):** Mengukur `http_requests_total` yang dikelompokkan berdasarkan label `route="/checkout"` dan `status`, serta `http_request_duration_seconds_bucket`.
   - **Synthetic Probing via `blackbox_exporter`:** Membuat endpoint `/readyz` atau `/healthz/deep` yang memvalidasi koneksi database & dependency, lalu di-probe secara berkala oleh `blackbox_exporter`.
3. **Kueri PromQL Rasio Error 5xx:**
   ```promql
   100 * (
     sum(rate(http_requests_total{service="order-service", status=~"5.."}[5m]))
     /
     sum(rate(http_requests_total{service="order-service"}[5m]))
   )
   ```

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah memahami cara kerja scraping, exporter, dan penulisan kueri PromQL, lanjutkan ke materi berikutnya:
- [02 — Rules, Alertmanager, Retention, dan Cardinality](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.1-prometheus/02-rules-alertmanager-retention-cardinality.md) — Mengubah kueri PromQL menjadi Recording Rules dan Alerting Rules, mengatur perutean notifikasi di Alertmanager, serta mencegah ledakan kardinalitas (*cardinality explosion*).
- Praktikkan langsung di [LAB-01 — Prometheus, Node Exporter, dan Blackbox Scrape](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.1-prometheus/lab/LAB-01-prometheus-node-blackbox-scrape.md).
