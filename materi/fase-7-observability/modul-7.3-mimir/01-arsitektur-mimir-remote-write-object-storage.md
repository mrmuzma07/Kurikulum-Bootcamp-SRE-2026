# 01 — Arsitektur Mimir, Remote-Write, dan Object Storage

> **Ringkasan:** Memahami mengapa **Grafana Mimir** dibutuhkan untuk mengatasi keterbatasan skala Prometheus tunggal, membedah jalur tulis (*Write Path*) dan jalur baca (*Read Path*) berbasis *Hash Ring*, mengintegrasikan **MinIO Object Storage** di lingkungan on-premises secara aman, serta menerapkan isolasi **Multi-Tenancy** (`X-Scope-OrgID`).

---

## 1. Mengapa Butuh Grafana Mimir? (Prometheus Lokal vs Mimir Terdistribusi)

### 1.1 Analogi Dunia Nyata: "Rak Buku Kamar Pribadi vs Perpustakaan Nasional & Gudang Arsip Kontainer"
Bayangkan perbedaan cara menyimpan catatan:
- **Prometheus Tunggal (Rak Buku di Kamar Pribadi):** Sangat praktis dan cepat untuk penggunaan sehari-hari. Namun, ia memiliki tiga kelemahan fisik:
  1. **Tidak Ada High Availability (HA) Sejati:** Jika hard disk komputer kamar itu rusak, seluruh sejarah grafik hilang. Menjalankan 2 Prometheus kembar pun sering menghasilkan grafik yang sedikit berbeda (*drift*) saat salah satu direstart.
  2. **Kapasitas Terbatas Satu Mesin (*Vertical Scaling Limit*):** Jika Anda punya 10 cluster Kubernetes yang mengirim 20 juta metrik, satu server fisik tidak akan sanggup menampung RAM dan disk-nya.
  3. **Sulit Melihat Gambaran Global (*Global View*):** Anda harus berpindah-pindah datasource untuk melihat metrik Cluster A dan Cluster B.
- **Grafana Mimir (Perpustakaan Nasional Terdistribusi + Gudang MinIO):** Mimir memecah fungsi database menjadi tim petugas spesialis yang bisa ditambah jumlahnya kapan saja (*horizontally scalable*) dan menyimpan arsip jangka panjangnya di gudang **Object Storage (MinIO S3 On-Prem)** yang murah, tahan banting, dan berkapasitas Terabyte/Petabyte!

---

### 1.2 Arsitektur Komponen Mimir: Write Path, Storage, dan Read Path

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   ARSITEKTUR DATA PATH GRAFANA MIMIR                        │
│                                                                             │
│  [1. WRITE PATH (Jalur Masuk Data via Remote-Write)]                        │
│                                                                             │
│   Grafana Alloy / Prometheus                                                │
│        │ (HTTP POST /api/v1/push + Header: X-Scope-OrgID: prod-tenant)      │
│        ▼                                                                    │
│   ┌─────────────────┐                                                       │
│   │   DISTRIBUTOR   │ (Stateless: Validasi format, Rate-Limit per Tenant)   │
│   └────────┬────────┘                                                       │
│            │ (Replikasi Hash Ring ke 3 Ingester / RF=3)                     │
│      ┌─────┴───────────────┬─────────────────────┐                          │
│      ▼                     ▼                     ▼                          │
│  ┌────────────┐        ┌────────────┐        ┌────────────┐                 │
│  │ INGESTER 1 │        │ INGESTER 2 │        │ INGESTER 3 │                 │
│  │ (RAM + WAL)│        │ (RAM + WAL)│        │ (RAM + WAL)│                 │
│  └─────┬──────┘        └─────┬──────┘        └─────┬──────┘                 │
│        └─────────────────────┼─────────────────────┘                        │
│                              │ (Flush blok setiap 2 jam)                    │
│                              ▼                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │            MINIO OBJECT STORAGE ON-PREMISES (S3-Compatible)           │  │
│  │    Bucket: [mimir-tsdb-blocks]    [mimir-ruler]    [mimir-alert]      │  │
│  └───────────────────────────▲─────────────────────────┬─────────────────┘  │
│                              │                         │                    │
│  ┌───────────────────────────┴──────┐                  │                    │
│  │            COMPACTOR             │                  │ (Baca blok > 2 jam)│
│  │ (Gabung blok, Deduplikasi & Retensi)                ▼                    │
│  └──────────────────────────────────┘        ┌───────────────────┐          │
│                                              │   STORE-GATEWAY   │          │
│  [2. READ PATH (Jalur Kueri PromQL)]         └─────────▲─────────┘          │
│                                                        │                    │
│   Grafana UI ──> [ QUERY-FRONTEND ] ──> [ QUERIER ] ───┴─(Baca < 2 jam dari │
│                  (Split & Cache)        (Gabung Hasil)    Ingester RAM)     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 1.3 Bedah Peran 7 Komponen Spesialis Mimir

| Nama Komponen | Sifat State | Tugas Utama & Failure Domain |
|---|---|---|
| **1. Distributor** | Stateless | Gerbang depan penerima paket `remote_write`. Memeriksa autentikasi tenant, memvalidasi label, mencegah satu tenant membanjiri server (*rate limiting*), lalu meneruskan sampel ke **Ingester** menggunakan algoritma *Consistent Hash Ring*. |
| **2. Ingester** | Stateful (RAM + Disk WAL) | Menyimpan sampel yang baru masuk di dalam memori RAM dan mencatatnya ke disk *Write-Ahead Log (WAL)*. Dengan **Replication Factor (RF) = 3**, setiap sampel disimpan di 3 pod Ingester berbeda. Jika 1 pod Ingester mati, data **tidak hilang**! Setiap 2 jam, Ingester membungkus data RAM menjadi blok TSDB dan mengunggahnya ke MinIO. |
| **3. Object Storage (MinIO)** | Persistent Storage | Tempat penyimpanan abadi untuk seluruh blok TSDB yang berusia lebih dari 2 jam. Menggantikan ketergantungan pada local disk besar di setiap pod. |
| **4. Compactor** | Stateless (Disk Scratch) | Bekerja di latar belakang mengunduh blok-blok 2 jam dari MinIO, menghapus data duplikat hasil replikasi 3 Ingester (*deduplication*), menggabungkannya menjadi blok besar yang efisien, serta menghapus blok yang sudah melewati masa **Retention**. |
| **5. Store-Gateway**| Semi-Stateful (Cache Disk)| Menyimpan indeks blok MinIO di memori agar ketika ada kueri yang mencari data bulan lalu, Mimir tahu persis bagian file mana di MinIO yang harus diambil tanpa perlu memindai seluruh isi bucket. |
| **6. Query-Frontend**| Stateless | Menerima kueri PromQL dari Grafana. Jika pengguna meminta grafik rentang **30 hari**, Query-Frontend memecahnya menjadi **30 kueri kecil per 1 hari** yang dieksekusi secara paralel oleh banyak **Querier**, sehingga grafik muncul 10x lebih cepat! |
| **7. Querier & Ruler**| Stateless | **Querier** mengambil data terbaru (< 2 jam) dari Ingester dan data historis (> 2 jam) dari Store-Gateway lalu menghitung rumus PromQL. **Ruler** mengevaluasi Recording Rules dan Alerting Rules secara berkala. |

---

## 2. Kontrak Remote-Write & Multi-Tenancy (`X-Scope-OrgID`)

### 2.1 Apa yang Terjadi di Balik Layar Protokol `remote_write`?
Saat Grafana Alloy atau Prometheus mengirim metrik ke Mimir:
1. Sampel metrik dikumpulkan dalam satu kelompok (*batch*), diubah menjadi format biner **Protobuf**, dan dikompresi dengan algoritma **Snappy** (atau Zstd).
2. Dikirim melalui `HTTP POST` ke endpoint `/api/v1/push` milik Mimir Distributor.
3. **Aturan Kode Status HTTP:**
   - **`200 OK` / `204 No Content`:** Sampel berhasil diterima oleh kuorum Ingester.
   - **`400 Bad Request`:** Sampel cacat (misal nama label ilegal, timestamp terlalu tua/*out-of-order*, atau melampaui batas kardinalitas tenant). Alloy **tidak akan mencoba mengirim ulang (tidak di-retry)** error 4xx agar antrean tidak macet!
   - **`429 Too Many Requests` / `5xx Server Error`:** Mimir sedang sibuk atau dalam proses rolling restart. Alloy akan menyimpan sampel di dalam disk WAL lokal dan **mencoba mengirim ulang (*exponential backoff retry*)** sampai Mimir kembali pulih.

> [!WARNING]
> **Prinsip Bukti SRE (*Evidence Boundary*):**
> Melihat status HTTP `200 OK` pada log `remote_write` Alloy **hanya membuktikan bahwa Distributor/Ingester menerima paket di memori**, bukan bukti bahwa blok sudah tersimpan abadi di MinIO atau bahwa kueri Grafana Anda sudah benar! Selalu verifikasi dari ujung ke ujung (*end-to-end query verification*).

---

### 2.2 Isolasi Multi-Tenancy dengan Header `X-Scope-OrgID`

Mimir dirancang sejak awal untuk melayani banyak tim atau banyak cluster sekaligus tanpa mencampuradukkan data mereka:
- Setiap request pengiriman (`remote_write`) maupun pembacaan (`PromQL query`) **wajib** menyertakan HTTP Header **`X-Scope-OrgID`** (contoh: `X-Scope-OrgID: production` atau `X-Scope-OrgID: payment-team`).
- Di dalam bucket MinIO, Mimir otomatis memisahkan folder penyimpanan per tenant:
  ```text
  s3://mimir-tsdb-blocks/
  ├── production/
  │   ├── 01JACTG9.../ (Blok TSDB milik tenant production)
  ├── staging/
  │   ├── 01JACTH2.../ (Blok TSDB milik tenant staging)
  ```
- Keuntungan SRE: Jika aplikasi di environment `staging` mengalami bug yang memicu ledakan kardinalitas, Mimir hanya akan membatasi (*throttle*) kuota tenant `staging`, sementara metrik tenant `production` tetap berjalan 100% aman tanpa gangguan!

---

## 3. Konfigurasi Mimir dengan MinIO On-Premises (Secret-Safe)

Dalam lingkungan latihan lokal (OrbStack) kita dapat menjalankan Mimir dalam mode **Monolithic** (seluruh 7 komponen berjalan di dalam 1 binary proses tunggal untuk menghemat RAM), sedangkan di lingkungan bare-metal produksi kita menjalankannya dalam mode **Microservices**. Konfigurasi file `mimir.yaml` keduanya tetap sama!

Berikut adalah konfigurasi `mimir.yaml` tingkat produksi yang terhubung ke MinIO S3 On-Premises **tanpa menuliskan kredensial rahasia secara plaintext**:

```yaml
# Aktifkan isolasi multi-tenancy wajib
multitenancy_enabled: true

# Konfigurasi Penyimpanan Blok Utama ke MinIO S3 On-Prem
blocks_storage:
  backend: s3
  s3:
    endpoint: "minio.storage.svc.cluster.local:9000"
    bucket_name: "mimir-tsdb-blocks"
    insecure: true # Gunakan false dan sertakan CA cert untuk produksi dengan TLS
    # Kredensial disuntikkan melalui variabel environment dari K8s Secret (-config.expand-env=true)
    access_key_id: "${MINIO_ACCESS_KEY_ID}"
    secret_access_key: "${MINIO_SECRET_ACCESS_KEY}"

  tsdb:
    dir: "/data/ingester-tsdb"
    # Durasi penyimpanan blok secara keseluruhan (misal 90 hari di MinIO)
    retention_period: 90d

# Konfigurasi Hash Ring Ingester & Faktor Replikasi
ingester:
  ring:
    kvstore:
      store: memberlist
    # Di produksi minimal 3 replika agar tahan terhadap matinya 1 node fisik!
    # (Di lab lokal single-node dapat diset ke 1)
    replication_factor: 3

# Konfigurasi Penyimpanan Rules (Recording & Alerting Rules) di MinIO
ruler_storage:
  backend: s3
  s3:
    endpoint: "minio.storage.svc.cluster.local:9000"
    bucket_name: "mimir-ruler-rules"
    access_key_id: "${MINIO_ACCESS_KEY_ID}"
    secret_access_key: "${MINIO_SECRET_ACCESS_KEY}"

# Batas Pelindung per Tenant (Mencegah Ledakan Kardinalitas!)
limits:
  # Maksimal 150.000 series aktif per tenant
  max_global_series_per_user: 150000
  # Maksimal laju pengiriman 25.000 sampel/detik per tenant
  ingestion_rate: 25000
  ingestion_burst_size: 50000
  # Tolak sampel yang jamnya meleset terlalu jauh
  out_of_order_time_window: 30m
  # Aktifkan penghapusan otomatis blok tua oleh Compactor setelah 90 hari
  compactor_blocks_retention_period: 90d
```

> [!CAUTION]
> **Aturan Higienitas Rahasia (*Secrets Hygiene*) pada Konfigurasi Mimir:**
> Jangan pernah meng-commit nilai asli `access_key_id` atau `secret_access_key` MinIO ke dalam file YAML GitOps! Aktifkan argumen CLI **`-config.expand-env=true`** pada container Mimir, lalu suntikkan variabel `${MINIO_ACCESS_KEY_ID}` dan `${MINIO_SECRET_ACCESS_KEY}` menggunakan `envFrom.secretRef` dari Kubernetes Secret yang dikelola secara terenkripsi.

---

## 4. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Kemungkinan Penyebab Utama | Langkah Investigasi & Solusi SRE |
|---|---|---|
| **Alloy Mencatat Error `400 Bad Request: per-user series limit (150000) exceeded`** | Sebuah aplikasi baru di tenant tersebut menciptakan *High Cardinality* sehingga jumlah series aktif menyentuh batas `max_global_series_per_user`. | 1. **Jangan langsung menaikkan limit tanpa investigasi!**<br>2. Cek metrik dengan kardinalitas tertinggi di tenant tersebut.<br>3. Buang label bermasalah di Alloy (`prometheus.relabel` dengan `labeldrop`). |
| **Alloy Mencatat Error `500 Internal Server Error: ring has no healthy instances`** | Jumlah pod **Ingester** yang sehat kurang dari syarat kuorum `replication_factor` (misal setting `replication_factor: 3` tetapi Anda hanya menjalankan 1 pod Ingester di lab!). | Sesuaikan nilai `ingester.ring.replication_factor` dengan jumlah replika pod Ingester aktual (gunakan `1` untuk lab single-node, dan minimal `3` untuk klaster produksi multi-node). |
| **Grafik 1 Jam Terakhir Muncul, tapi Grafik Minggu Lalu Kosong** |Data terbaru (< 2 jam) berhasil dibaca dari RAM Ingester, namun **Store-Gateway** gagal membaca blok lama dari bucket **MinIO** atau **Compactor** gagal bekerja. | 1. Periksa log pod `store-gateway` dan `compactor`.<br>2. Pastikan kredensial bucket MinIO valid dan network policy mengizinkan akses ke port `9000` MinIO. |
| **Kueri Grafana Selalu Kosong Padahal Ingester Menerima Data** | Ketidakcocokan header **`X-Scope-OrgID`** antara pengirim (Alloy) dan pembaca (Grafana Datasource). | Pastikan Datasource Prometheus/Mimir di Grafana dikonfigurasi dengan Custom HTTP Header `X-Scope-OrgID` yang **sama persis** (peka huruf besar/kecil) dengan yang dikirim oleh Alloy! |

---

## 5. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. INSPEKSI KESEHATAN HASH RING & KOMPONEN MIMIR
# ==============================================================================

# Membuka halaman status Ingester Hash Ring via port-forward (Cek apakah semua ACTIVE)
kubectl port-forward -n monitoring svc/mimir-distributor 8080:8080
curl -s http://localhost:8080/ingester/ring

# Memeriksa daftar seluruh modul yang aktif pada binary Mimir
curl -s http://localhost:8080/services

# ==============================================================================
# 2. MENGUJI PENGIRIMAN & KUERI METRIK DENGAN HEADER MULTI-TENANT
# ==============================================================================

# Mengeksekusi kueri PromQL ke Mimir Query-Frontend dengan menyertakan X-Scope-OrgID
curl -sG http://localhost:8080/prometheus/api/v1/query \
  -H "X-Scope-OrgID: onprem-prod-tenant" \
  --data-urlencode 'query=up' | jq .

# Memeriksa statistik penggunaan kardinalitas dan batas limit per tenant
curl -s http://localhost:8080/ingester/all_user_stats
```

---

## 6. Evaluasi Pemahaman Mandiri

### 6.1 Pilihan Ganda (3 Soal)

#### Soal 1
Dalam arsitektur **Grafana Mimir**, komponen manakah yang bertugas menerima sampel metrik di memori RAM, mencatatnya ke *Write-Ahead Log (WAL)*, lalu membungkusnya setiap 2 jam menjadi blok TSDB untuk diunggah ke **MinIO Object Storage**?
- A. Query-Frontend
- B. Ingester
- C. Store-Gateway
- D. Alertmanager

#### Soal 2
Mengapa ketika menjalankan kueri PromQL untuk rentang waktu 7 hari yang lalu di Mimir, **Querier** harus mengambil data dari **Ingester** SEKALIGUS dari **Store-Gateway**?
- A. Karena Ingester hanya menyimpan data metrik terbaru yang belum di-flush (biasanya 2 jam terakhir di RAM), sedangkan data historis yang berusia lebih dari 2 jam sudah berada di dalam blok Object Storage (MinIO) yang dilayani oleh Store-Gateway.
- B. Karena Store-Gateway hanya bisa membaca data metrik bertipe Counter, sedangkan Ingester hanya bisa membaca Gauge.
- C. Karena Mimir tidak memiliki fitur penyimpanan disk.
- D. Untuk menghabiskan bandwidth jaringan lokal secara sengaja.

#### Soal 3
Apa yang terjadi jika Grafana Alloy mengirimkan metrik dengan header `X-Scope-OrgID: production-cluster`, tetapi Datasource di Grafana dikonfigurasi dengan header `X-Scope-OrgID: prod-cluster`?
- A. Mimir secara cerdas menerjemahkan singkatan `prod` menjadi `production` secara otomatis.
- B. Kueri di Grafana akan mengembalikan hasil kosong (`Empty query result`) karena Mimir mengisolasi ruang penyimpanan tiap tenant secara ketat berdasarkan kesamaan persis string `X-Scope-OrgID`.
- C. Seluruh data di bucket MinIO akan terhapus.
- D. Pod Mimir Distributor akan langsung mati (`CrashLoopBackOff`).

---

### 6.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Anda sedang menyiapkan lingkungan observabilitas laboratorium di MacBook (OrbStack) menggunakan 1 replika pod Mimir (`replicas: 1`). Anda menyalin mentah-mentah file konfigurasi `mimir.yaml` dari repositori produksi yang memiliki pengaturan `ingester.ring.replication_factor: 3`.

Saat Anda menjalankan pod Mimir dan mengarahkan Grafana Alloy untuk melakukan `remote_write`, log Alloy terus-menerus memuntahkan pesan error:
`server returned HTTP status 500 Internal Server Error: at least 2 live replicas required, could only find 1`

**Pertanyaan & Tugas Anda:**
1. Jelaskan secara matematis (konsep **Quorum**) mengapa Mimir meminta *"at least 2 live replicas required"* ketika `replication_factor` bernilai `3`!
2. Mengapa di lingkungan produksi bare-metal yang sesungguhnya nilai `replication_factor: 3` dengan minimal 3 pod Ingester di 3 node berbeda sangat diwajibkan?
3. Apa perubahan yang harus Anda lakukan agar lab single-node di OrbStack dapat berjalan lancar tanpa error, dan bagaimana cara Anda memisahkan nilai konfigurasi lab vs produksi di Helm/GitOps?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** **Ingester** adalah komponen stateful pada jalur tulis yang menahan sampel terbaru di RAM/WAL sebelum mem-flush blok 2 jam ke Object Storage.
2. **A.** Pemisahan ini adalah kunci efisiensi Mimir: data panas (*hot data* < 2 jam) dilayani langsung dari RAM Ingester, sedangkan data dingin (*cold data* > 2 jam hingga bertahun-tahun) dilayani dari Object Storage via Store-Gateway.
3. **B.** Multi-tenancy pada Mimir menggunakan header `X-Scope-OrgID` sebagai kunci partisi utama. Perbedaan satu karakter saja akan dianggap sebagai tenant yang sama sekali berbeda dan kosong.

#### Solusi Soal Kasus Nyata:
1. **Penjelasan Matematis Quorum (`N / 2 + 1`):**
   - Mimir menggunakan konsensus berbasis kuorum mayoritas untuk memastikan ketahanan data: `Quorum = floor(Replication_Factor / 2) + 1`.
   - Jika `Replication_Factor = 3`, maka kuorum minimal yang wajib berhasil menulis data adalah `floor(3 / 2) + 1 = 2` Ingester sehat. Karena di lab hanya ada 1 pod Ingester, syarat minimal 2 replika tidak terpenuhi sehingga Distributor menolak penulisan dengan HTTP 500.
2. **Pentingnya `replication_factor: 3` di Produksi:**
   - Karena data 2 jam terakhir masih berada di RAM/WAL Ingester dan belum masuk ke MinIO, jika hanya ada 1 Ingester dan server fisiknya terbakar atau di-reboot saat upgrade kernel, metrik 2 jam terakhir tidak bisa diakses. Dengan 3 replika yang tersebar di 3 server fisik (`topologySpreadConstraints`), matinya 1 server fisik tidak menghentikan penulisan maupun pembacaan metrik sedikit pun (karena 2 Ingester sisanya masih memenuhi syarat kuorum!).
3. **Solusi untuk Lab vs Produksi di GitOps:**
   - Pada `values-lab.yaml` (untuk OrbStack), atur `ingester.ring.replication_factor: 1` dan `replicas: 1`.
   - Pada `values-prod.yaml` (untuk cluster produksi), pertahankan `ingester.ring.replication_factor: 3` dan minimal `replicas: 3` dengan aturan *Pod Anti-Affinity* antar node.

---

## 7. Langkah Selanjutnya & Kaitan Modul

Setelah memahami arsitektur komponen Mimir dan integrasi Object Storage MinIO:
- Praktikkan di [LAB-01 — Mimir Remote-Write dan Query](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.3-mimir/lab/LAB-01-mimir-remote-write-query.md).
- Lanjutkan ke [02 — Retention, HA, Query, Recording Rules, dan Recovery](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.3-mimir/02-retention-ha-query-recording-rules.md) untuk menghitung kapasitas penyimpanan jangka panjang, merancang High Availability & Disaster Recovery, serta mengelola evaluasi Rules terpusat di Mimir Ruler.
