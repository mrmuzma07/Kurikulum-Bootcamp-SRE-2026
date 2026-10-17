# 01 — Loki, Label Strategy, LogQL, Redaction, dan Retention

> **Ringkasan:** Memahami filosofi arsitektur **Grafana Loki** (*"Like Prometheus, but for Logs"*), merancang strategi **Stream Labels** ber-kardinalitas rendah agar tidak merusak Object Storage (MinIO), menguasai bahasa kueri **LogQL** (pencarian teks, parsing JSON saat kueri, hingga ekstraksi metrik dari log), serta menerapkan kebijakan penyensoran rahasia (*redaction*) dan retensi otomatis.

---

## 1. Filosofi & Arsitektur Grafana Loki

### 1.1 Analogi Dunia Nyata: "Label Kardus Arsip Per Tanggal & Divisi (Loki) vs Indeks Kamus Setiap Kata (Elasticsearch)"
Bayangkan sebuah bank harus menyimpan 100 juta lembar bukti transaksi setiap bulan:
- **Pendekatan Full-Text Index (Seperti Elasticsearch):** Petugas mencatat **setiap kata, setiap angka, dan setiap nama jalan** yang tertulis di 100 juta kertas tersebut ke dalam buku kamus indeks raksasa.
  - *Kelebihan:* Mencari satu kata acak sangat cepat.
  - *Kelemahan Fatal:* Buku kamus indeksnya menjadi lebih tebal dan lebih mahal daripada kertas aslinya! Butuh puluhan server dengan RAM ratusan Gigabyte hanya untuk menjaga agar buku indeks tersebut tetap terbuka.
- **Pendekatan Grafana Loki (*Index Only Metadata Labels*):** Petugas **tidak mengindeks isi teks surat**. Petugas hanya menempelkan stiker label sederhana di luar kardus: `Cabang="Jakarta"`, `Divisi="Kredit"`, `Bulan="Oktober"`. Kertas-kertas di dalamnya dipadatkan (*compressed chunks*) dan dimasukkan ke gudang **MinIO Object Storage** yang sangat murah.
  - Saat auditor bertanya: *"Cari transaksi gagal milik nasabah Budi di Cabang Jakarta Divisi Kredit hari ini pukul 10:00–10:15"*, Loki langsung mengambil **hanya 1 kardus kecil** milik `Cabang="Jakarta", Divisi="Kredit"` pada jam tersebut, lalu mengerahkan 10 petugas pembaca paralel (*Queriers*) untuk memindai isi kardus itu dalam hitungan milidetik!

---

### 1.2 Arsitektur Alur Data Loki (Write Path & Read Path)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ARSITEKTUR LOG PIPELINE GRAFANA LOKI                     │
│                                                                             │
│  [1. PENGUMPULAN & PENYENSORAN (DI WORKER NODE)]                            │
│   Pod Container stdout/stderr ──> /var/log/pods/*.log                       │
│                                           │                                 │
│                                           ▼                                 │
│   Grafana Alloy (`loki.process`):                                           │
│    ├─ Pasang Stream Labels Terbatas: {namespace="prod", service="payment"}  │
│    ├─ Sensor Rahasia/PII: Hapus Bearer Token, Password, Email               │
│    └─ Kirim HTTP POST /loki/api/v1/push (+ Header X-Scope-OrgID)            │
│                                           │                                 │
│  [2. PENYIMPANAN LOKI & MINIO OBJECT STORAGE]                               │
│                                           ▼                                 │
│   [ Distributor ] ──(Hash Ring RF=3)──> [ Ingester (RAM Chunks + WAL) ]     │
│                                                   │                         │
│                                    (Flush Chunks & TSDB Index)              │
│                                                   ▼                         │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │            MINIO OBJECT STORAGE ON-PREMISES (S3-Compatible)          │  │
│   │   Bucket: [loki-chunks-prod] (Isi Teks Terkompresi + TSDB Index)     │  │
│   └───────────────────────────────────────────────▲──────────────────────┘  │
│                                                   │                         │
│  [3. EKSEKUSI KUERI LOGQL PARALEL]                │                         │
│   Grafana Explore ──> [ Query-Frontend ] ──> [ Queriers ] (Buka & Grep!)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Aturan Emas Strategi Label di Loki (Stream Labels vs Structured Metadata)

### 2.1 Mengapa High-Cardinality Labels Menghancurkan Loki ("Million Tiny Chunks Problem")
Di Loki, **setiap kombinasi unik dari Stream Labels membentuk satu aliran file terpisah (*Log Stream*)**.
- Jika Anda menggunakan label yang benar: `{cluster="prod", namespace="payment", service="checkout", level="error"}`, maka hanya ada sekitar **50 – 200 stream aktif**. Ingester mengumpulkan log ke dalam *chunk* berukuran padat (1–2 MB) sebelum diunggah ke MinIO.
- **Bencana jika Anda menjadikan `request_id` atau `client_ip` sebagai Stream Label:**
  - Setiap 1 request pengguna akan membuat **1 Stream Baru**!
  - Jika ada 100.000 request dalam 10 menit, Loki Ingester terpaksa membuat **100.000 file chunk berukuran sangat kecil (hanya 200 Bytes per file)** dan melakukan **100.000 operasi `PUT Object` ke MinIO**!
  - Akibatnya: Disk MinIO lumpuh karena kehabisan IOPS, RAM Ingester meledak (`OOMKilled`), dan kueri menjadi sangat lambat!

### 2.2 Tabel Panduan Desain Label Loki vs Isi Log JSON

| Jenis Data / Field | Boleh Jadi Stream Label Loki? | Cara Penyimpanan & Pencarian yang Benar |
|---|---|---|
| `cluster`, `sre_environment` | **WAJIB (Ya)** | Label statis tingkat klaster (hanya 2–5 nilai). |
| `namespace`, `service` / `app` | **WAJIB (Ya)** | Identitas utama aplikasi (bounded cardinality). |
| `level` / `severity` | **BOLEH (Ya)** | Hanya memiliki maksimal 5 nilai (`debug`, `info`, `warn`, `error`, `fatal`). |
| `trace_id`, `span_id`, `request_id`| **DILARANG KERAS (Tidak!)** | Simpan di dalam **Structured Metadata** Loki atau di dalam **isi teks JSON log**, lalu cari menggunakan filter `\|= "<trace_id>"` atau `\| json \| trace_id="..."`. |
| `user_id`, `order_id`, `client_ip`| **DILARANG KERAS (Tidak!)** | Biarkan di dalam isi teks JSON log (pastikan sudah disensor jika bersifat PII!). |
| `Authorization`, `password`, `token`| **HARAM Masuk ke Loki!** | Wajib dihapus/disensor (*redacted*) di **Grafana Alloy** sebelum dikirim ke Loki! |

---

## 3. Menguasai LogQL dari Dasar hingga Mahir

**LogQL (Loki Query Language)** terinspirasi langsung dari PromQL. Setiap kueri LogQL terdiri dari dua bagian utama:
1. **Stream Selector `{...}` (Wajib):** Memilih kardus arsip mana yang akan dibuka berdasarkan label.
2. **Log Pipeline `| ...` (Opsional & Berantai):** Menyaring teks, mem-parsing JSON, dan memanipulasi hasil secara berurutan dari kiri ke kanan.

```text
{namespace="production", service="payment-api"} |= "error" | json | status_code >= 500
└──────────────────────┬──────────────────────┘ └───┬────┘ └──┬─┘ └────────┬─────────┘
        1. Stream Selector                      2. Line    3. JSON   4. Ekspresi Filter
         (Buka kardus label)                     Filter     Parser     Field Hasil Parse
```

### 3.1 Kueri Pencarian Baris Log (*Log Queries*)

> **Tips Performa SRE:** Selalu letakkan **Line Filter teks mentah (`|=` atau `!=`) SEBELUM `| json`**! Menyaring teks mentah dengan `|=` jauh lebih cepat karena Loki langsung membuang 99% baris yang tidak relevan tanpa perlu membuang waktu CPU untuk mem-parsing struktur JSON-nya.

```logql
# 1. Cari baris log yang mengandung kata "timeout" TAPI bukan dari healthcheck
{namespace="production", service="payment-api"} |= "timeout" != "/healthz"

# 2. Cari dengan Regex (Case-Insensitive `(?i)`) untuk error database atau redis
{namespace="production", service="order-api"} |~ "(?i)(connection refused|deadlock|redis timeout)"

# 3. Filter Cepat + Parse JSON + Filter Angka (Latensi > 500ms dan Status 5xx)
{namespace="production", service="payment-api"}
  |= "duration_ms"
  | json
  | status >= 500 and duration_ms > 500

# 4. Merapikan tampilan output log agar mudah dibaca saat investigasi insiden
{namespace="production", service="payment-api"}
  | json
  | level="error"
  | line_format "[{{.level}}] Route={{.path}} Status={{.status}} Latency={{.duration_ms}}ms TraceID={{.trace_id}} Msg={{.message}}"
```

---

### 3.2 Kueri Metrik dari Log (*Metric Queries in LogQL*)

Apa yang harus Anda lakukan jika sebuah aplikasi *legacy* belum memiliki metrik Prometheus di `/metrics`, tetapi mencatat waktu proses dan kode status di dalam log? Anda bisa mengubah baris log menjadi grafik angka secara *real-time* menggunakan LogQL!

```logql
# 1. Menghitung jumlah baris log ERROR per detik untuk tiap service (selama 5 menit)
sum by (service) (
  rate({namespace="production"} |= "error" | json | level="error" [5m])
)

# 2. Menghitung Rasio Error dari Log HTTP Access (Error 5xx dibagi Total Log Request)
sum by (service) (
  rate({namespace="production", service="legacy-app"} | json | status >= 500 [5m])
)
/
sum by (service) (
  rate({namespace="production", service="legacy-app"} | json | status >= 100 [5m])
)

# 3. Menghitung Persentil Latensi ke-95 (p95) dengan mengekstrak angka dari JSON (`unwrap`)
quantile_over_time(
  0.95,
  {namespace="production", service="payment-api"}
    | json
    | __error__=""
    | unwrap duration_ms [5m]
) by (service)
```

---

## 4. Redaksi Data Sensitif, Retensi, dan Konfigurasi `loki.yaml`

### 4.1 Konfigurasi Produksi Loki + MinIO S3 dengan Retensi Otomatis

Berikut adalah konfigurasi `loki.yaml` yang menggunakan skema indeks modern **TSDB** di atas **MinIO S3 On-Premises** serta mengaktifkan **Compactor Retention**:

```yaml
auth_enabled: true # Wajib: aktifkan multi-tenancy (Header X-Scope-OrgID)

server:
  http_listen_port: 3100

common:
  path_prefix: /var/loki
  replication_factor: 3 # Gunakan 1 untuk lab lokal OrbStack, 3 untuk produksi
  storage:
    s3:
      endpoint: "minio.storage.svc.cluster.local:9000"
      bucketnames: "loki-chunks-prod"
      s3forcepathstyle: true
      insecure: true
      # Disuntikkan dengan argumen CLI -config.expand-env=true dari K8s Secret
      access_key_id: "${MINIO_ACCESS_KEY_ID}"
      secret_access_key: "${MINIO_SECRET_ACCESS_KEY}"

# Gunakan skema indeks TSDB modern (hemat memori & disimpan langsung di S3/MinIO)
schema_config:
  configs:
    - from: "2026-01-01"
      store: tsdb
      object_store: s3
      schema: v13
      index:
        prefix: loki_index_
        period: 24h

# Aktifkan penghapusan log otomatis sesuai masa retensi oleh Compactor
compactor:
  working_directory: /var/loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  delete_request_store: s3

# Batas Pelindung Kardinalitas & Retensi per Tenant
limits_config:
  # Hapus log yang berusia lebih dari 30 hari (720 jam) secara otomatis
  retention_period: 720h
  # Batasi maksimal 5.000 stream aktif per tenant (Cegah ledakan label!)
  max_global_streams_per_user: 5000
  # Batasi laju pengiriman log maksimal 10 MB/detik per tenant
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  # Izinkan penyimpanan trace_id di dalam Structured Metadata tanpa merusak indeks!
  allow_structured_metadata: true
```

---

### 4.2 SOP Insiden SRE: Bagaimana Jika Kredensial Terlanjur Masuk ke Loki?
Jika meskipun sudah dipasang filter di Alloy, ditemukan sebuah password atau private key baru yang lolos dan tersimpan di dalam Loki:
1. **Langkah 1 — Rotasi & Cabut Kredensial Segera (*Immediate Revocation*):** Anggap kredensial tersebut sudah hangus/bocor. Ganti password/token di sistem sumber terlebih dahulu!
2. **Langkah 2 — Perbaiki Filter Redaksi di Hulu:** Tambahkan pola regex baru pada `loki.process` (`stage.replace`) di Grafana Alloy agar kebocoran berhenti.
3. **Langkah 3 — Eksekusi Penghapusan Terarah (*Targeted Log Deletion API*):** Gunakan endpoint `/loki/api/v1/delete` pada Loki Compactor untuk memusnahkan baris log yang mengandung rahasia tersebut dari bucket MinIO disertai pencatatan tiket audit keamanan, dan **jangan pernah menyalin teks rahasia tersebut ke dalam laporan insiden/postmortem!**

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Kemungkinan Penyebab | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Alloy Mencatat Error `429 Too Many Requests: Ingestion rate limit exceeded`** | Satu aplikasi mengalami *error loop* dan memuntahkan ratusan MB log per detik melampaui `ingestion_rate_mb`. | 1. Identifikasi service paling berisik dengan kueri LogQL: `topk(5, sum by (service) (bytes_rate({namespace="production"}[5m])))`.<br>2. Pasang komponen `stage.limit` (rate-limiting per pod) di Alloy atau perbaiki bug looping aplikasi. |
| **Alloy Mencatat Error `400 Bad Request: maximum of series (5000) reached`** | Ada label ber-kardinalitas tinggi (seperti `ip`, `pid`, atau `trace_id`) yang masuk ke `stage.labels` di Alloy. | Periksa label aktif di Loki dengan `logcli labels` dan hapus label dinamis tersebut dari blok `stage.labels` pada konfigurasi Alloy. |
| **Kueri LogQL `Timeout` atau Sangat Lambat** | 1. Pengguna hanya menulis `{namespace="production"} | json` tanpa menyertakan filter teks `|=` terlebih dahulu pada rentang waktu yang lebar.<br>2. Rentang waktu terlalu besar (misal 7 hari sekaligus). | Selalu tambahkan *Line Filter* spesifik (`|= "kata_kunci"`) sebelum `| json` dan batasi jendela waktu pencarian awal menjadi 15–60 menit. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting (`logcli`)

```bash
# ==============================================================================
# 1. MENGGUNAKAN LOGCLI UNTUK KUERI & AUDIT LABEL DARI TERMINAL
# ==============================================================================

# Mengatur alamat server Loki dan header tenant via environment variable
export LOKI_ADDR="http://localhost:3100"
export LOKI_ORG_ID="onprem-prod-tenant"

# Menampilkan seluruh daftar nama label yang ada di Loki (Audit Kardinalitas!)
logcli labels

# Menampilkan seluruh variasi nilai pada sebuah label tertentu
logcli labels service

# Menjalankan pencarian log 1 jam terakhir dari terminal (seperti grep terpusat)
logcli query '{namespace="production", service="payment-api"} |= "error" | json' \
  --since=1h --limit=50

# Memantau aliran log secara langsung (Live Tail) dari terminal
logcli query '{namespace="production", service="payment-api"}' --tail

# ==============================================================================
# 2. MENGUJI PENGIRIMAN & PENGHAPUSAN LOG VIA HTTP API
# ==============================================================================

# Memeriksa statistik volume log (Bytes & Streams) yang masuk per tenant
curl -s -H "X-Scope-OrgID: onprem-prod-tenant" http://localhost:3100/loki/api/v1/index/stats \
  --data-urlencode 'query={namespace="production"}' | jq .
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Apa perbedaan mendasar cara penyimpanan dan pengindeksan data antara **Grafana Loki** dengan sistem pencatat log tradisional berbasis *Full-Text Index* (seperti Elasticsearch)?
- A. Loki mengindeks setiap kata dan angka di dalam kalimat log sehingga membutuhkan memori RAM 10 kali lebih besar.
- B. Loki **hanya mengindeks sekumpulan kecil label metadata (Stream Labels)** yang sama seperti label Prometheus, lalu menyimpan isi teks log mentah dalam bentuk *compressed chunks* di Object Storage (MinIO/S3), sehingga sangat hemat biaya dan RAM.
- C. Loki hanya bisa menyimpan maksimal 100 baris log per hari.
- D. Loki menyimpan seluruh baris log di dalam browser pengguna.

#### Soal 2
Manakah urutan penulisan kueri **LogQL** berikut yang paling cepat dan hemat pemakaian CPU bagi server Loki saat mencari error transaksi untuk `trace_id="9f8e7d6c"`?
- A. `{namespace="production"} | json | trace_id="9f8e7d6c"`
- B. `{namespace="production", service="payment-api"} |= "9f8e7d6c" | json | trace_id="9f8e7d6c"`
- C. `{trace_id="9f8e7d6c"}` (Menjadikan `trace_id` sebagai Stream Label utama).
- D. `{namespace=~".*"} | json`

#### Soal 3
Mengapa ketiadaan baris log error (`0 log lines returned`) pada kueri `{namespace="production", service="checkout"} |= "ERROR"` **TIDAK BOLEH** langsung disimpulkan bahwa layanan `checkout` sedang berjalan 100% sehat?
- A. Karena kata "ERROR" tidak pernah digunakan dalam bahasa pemrograman modern.
- B. Karena ketiadaan log error bisa saja disebabkan oleh: (1) aplikasi mati total sehingga tidak menghasilkan log sama sekali, (2) tidak ada trafik pengguna yang masuk karena Ingress/LoadBalancer putus, atau (3) agen pengumpul log (Alloy) mengalami kemacetan/drop.
- C. Karena Loki selalu menghapus semua log setiap 10 detik.
- D. Karena LogQL hanya bekerja pada hari kerja.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Seorang developer di tim Anda mengeluh bahwa pencarian log di Grafana Loki terasa lambat saat ia ingin mencari log berdasarkan `order_id`. Ia mengajukan Merge Request ke repositori GitOps untuk mengubah konfigurasi `loki.process` di Grafana Alloy menjadi seperti ini:

```hcl
stage.json {
  expressions = {
    order_id   = "order_id",
    client_ip  = "client_ip",
    user_agent = "user_agent",
  }
}
stage.labels {
  values = {
    order_id   = "order_id",
    client_ip  = "client_ip",
    user_agent = "user_agent",
  }
}
```
Developer tersebut berargumen: *"Kalau `order_id`, `client_ip`, dan `user_agent` dijadikan label di Loki, nanti saya tinggal ketik `{order_id="ORD-991823"}` dan hasilnya langsung keluar cepat!"*

**Pertanyaan & Tugas Anda sebagai SRE Auditor:**
1. Mengapa Anda **WAJIB MENOLAK (Reject)** Merge Request tersebut? Jelaskan apa yang akan terjadi pada komponen **Loki Ingester** dan **MinIO Object Storage** jika perubahan itu diterapkan di produksi (500 pesanan per menit)!
2. Bagaimana cara yang benar untuk menyimpan `order_id` atau `trace_id` di Loki versi modern tanpa memasukkannya ke dalam `stage.labels` (Stream Labels)?
3. Tuliskan contoh kueri LogQL yang cepat dan efisien bagi developer tersebut untuk mencari `ORD-991823` tanpa menjadikan `order_id` sebagai label stream!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Desain *"index only metadata labels"* adalah kunci arsitektur Loki. Dengan menyamakan label stream log dengan label metrik Prometheus (`namespace`, `service`), biaya penyimpanan indeks turun drastis dan memudahkan perpindahan konteks dari grafik metrik ke baris log.
2. **B.** Menyaring dengan stream label spesifik (`service="payment-api"`) lalu memasang *Line Filter* teks sederhana (`|= "9f8e7d6c"`) **sebelum** parser `| json` membuat Loki hanya perlu mem-parse baris JSON yang benar-benar memuat string ID tersebut.
3. **B.** Dalam prinsip SRE, *ketiadaan sinyal error bukanlah bukti kesehatan sistem*. Validasi kesehatan layanan wajib menggabungkan volume total trafik/log yang masuk dan metrik keberhasilan positif.

#### Solusi Soal Kasus Nyata:
1. **Alasan Penolakan MR (Bahaya High-Cardinality Stream Explosion):**
   - `order_id`, `client_ip`, dan `user_agent` memiliki kombinasi nilai yang nyaris tak terbatas (*unbounded cardinality*).
   - Dengan 500 pesanan baru per menit, Loki akan dipaksa membuka **500 stream baru setiap menit (720.000 stream per hari)**.
   - Setiap stream hanya berisi 2–3 baris log lalu tidak pernah dipakai lagi, menghasilkan ratusan ribu file *chunk* mikro berukuran beberapa ratus byte (*Million Tiny Chunks Problem*) yang membuat RAM Ingester habis (`OOMKilled`) dan membebani tabel objek MinIO hingga macet.
2. **Solusi Penyimpanan Modern (Structured Metadata atau JSON Payload):**
   - Biarkan `order_id`, `client_ip`, dan `user_agent` tetap berada di dalam isi teks JSON log, atau gunakan fitur **`stage.structured_metadata`** pada Alloy/Loki (yang menempelkan metadata seperti `trace_id` atau `order_id` pada blok chunk untuk akselerasi bloom-filter tanpa membuat stream indeks baru!).
3. **Kueri LogQL yang Cepat & Efisien:**
   ```logql
   {namespace="production", service="order-service"} |= "ORD-991823" | json | order_id="ORD-991823"
   ```

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai strategi label Loki, kueri LogQL, serta kebijakan redaksi dan retensi:
- Praktikkan di [LAB-01 — Alloy ke Loki Log Pipeline](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.4-loki-tempo/lab/LAB-01-alloy-loki-log-pipeline.md).
- Lanjutkan ke [02 — Tempo, OTLP, dan Correlation Metrics-Logs-Traces](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.4-loki-tempo/02-tempo-otlp-traces-correlation.md) untuk mempelajari pelacakan terdistribusi (*Distributed Tracing*) dan menghubungkan ketiga pilar observabilitas dalam satu klik!
