# 02 — Retention, HA, Query, Recording Rules, dan Recovery

> **Ringkasan:** Menghitung kapasitas memori dan penyimpanan **Grafana Mimir** untuk retensi jangka panjang (bulanan hingga tahunan), merancang arsitektur **High Availability (HA)** tahan kegagalan node (*zone-aware replication*), mengelola **Recording & Alerting Rules** per tenant menggunakan `mimirtool`, serta menyusun prosedur pemulihan bencana (*Disaster Recovery*).

---

## 1. Perencanaan Kapasitas (Capacity Planning) & Retensi Mimir

### 1.1 Analogi Dunia Nyata: "Meja Kerja Akuntan (RAM Ingester) vs Gudang Kontainer Arsip (MinIO Disk)"
Merencanakan kapasitas Mimir jauh berbeda dengan Prometheus biasa karena Mimir memisahkan **Memori Aktif (Ingester RAM)** dari **Penyimpanan Arsip (MinIO Object Storage)**:
1. **Meja Kerja Akuntan (*Ingester RAM*):** Ukurannya ditentukan oleh **berapa banyak metrik unik (*active series*) yang sedang dikirim saat ini**, sama sekali tidak peduli apakah Anda menyimpan arsip selama 7 hari atau 5 tahun! Namun ingat: karena ada 3 akuntan yang mencatat hal yang sama (**Replication Factor = 3**), total kebutuhan meja kerja harus **dikalikan 3**.
2. **Gudang Kontainer Arsip (*MinIO Object Storage*):** Ukurannya ditentukan oleh **laju sampel per detik dikali lama waktu simpan (*retention window*)**. Begitu Compactor selesai bekerja dan membuang salinan kembar dari 3 Ingester (*deduplication*), data di MinIO kembali menjadi **1 salinan bersih yang terkompresi padat**!

---

### 1.2 Rumus Matematika Sizing RAM Ingester & Disk MinIO

Gunakan dua rumus eksplisit berikut sebelum menentukan spesifikasi VM atau server bare-metal on-premises:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ RUMUS 1: KEBUTUHAN TOTAL RAM INGESTER CLUSTER                               │
│                                                                             │
│ Total RAM Ingester = Active Series × Replication Factor (3) × 8 KB × 1.3    │
│ (Catatan: ~8 KB adalah rata-rata jejak memori Go per series di Head Block,  │
│  dan 1.3 adalah 30% headroom untuk GC / lonjakan kardinalitas sementara)    │
├─────────────────────────────────────────────────────────────────────────────┤
│ RUMUS 2: KEBUTUHAN KAPASITAS OBJECT STORAGE (MINIO S3)                      │
│                                                                             │
│ Total MinIO Storage = (Active Series ÷ Scrape Interval)                     │
│                       × Bytes per Sample (~2 Bytes)                         │
│                       × Retention Window (dalam detik)                      │
│                       × Compaction & Index Overhead (1.25)                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Contoh Studi Perhitungan Nyata (Skala Produksi Menengah):
Misalkan perusahaan Anda memantau **200.000 Active Series** dengan `scrape_interval = 15s`, `replication_factor = 3`, dan ingin menyimpan histori metrik selama **1 Tahun (365 Hari)** di MinIO:

1. **Hitung Total RAM untuk Pod Ingester:**
   - `200.000 series × 3 (RF) = 600.000 replicated series` di dalam klaster Ingester.
   - `600.000 × 8 KB = 4.800.000 KB ≈ 4,8 GB`.
   - Ditambah 30% *safety headroom* (`× 1.3`) = **`~6,24 GB Total RAM`** (Jika dibagi ke 3 Pod Ingester, masing-masing Pod Ingester cukup diberi `requests/limits: 2.5Gi` RAM!).
2. **Hitung Total Disk Bucket MinIO untuk Retensi 1 Tahun (365 Hari):**
   - Laju sampel masuk = `200.000 ÷ 15 detik = 13.333 samples/detik`.
   - Jumlah detik dalam 1 tahun = `365 × 86.400 = 31.536.000 detik`.
   - Ukuran data murni = `13.333 × 2 Bytes × 31.536.000 = 840.938.880.000 Bytes ≈ 841 GB`.
   - Ditambah 25% ruang kerja sementara Compactor & Index (`× 1.25`) = **`~1,05 TB (Terabyte)`** di MinIO untuk menyimpan metrik penuh selama 1 tahun!

---

## 2. High Availability (HA), Kuorum, dan Disaster Recovery

### 2.1 Arsitektur Zone-Aware Replication & Rolling Upgrade Aman

Agar Mimir benar-benar tahan terhadap matinya satu server fisik bare-metal tanpa kehilangan satu pun titik grafik:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│             TOPOLOGI HIGH AVAILABILITY (3 SERVER BARE-METAL)                │
│                                                                             │
│  [ Server Fisik / Node 1 ]  [ Server Fisik / Node 2 ]  [ Server Fisik / Node 3 ]
│  ┌───────────────────────┐  ┌───────────────────────┐  ┌───────────────────────┐
│  │  Pod Ingester-0 (Z-A) │  │  Pod Ingester-1 (Z-B) │  │  Pod Ingester-2 (Z-C) │
│  │  StatefulSet + PVC WAL│  │  StatefulSet + PVC WAL│  │  StatefulSet + PVC WAL│
│  └───────────▲───────────┘  └───────────▲───────────┘  └───────────▲───────────┘
│              │                          │                          │        │
│              └──────────────────────────┼──────────────────────────┘        │
│                                         │                                   │
│             Distributor menulis ke 3 Zona; Kuorum sukses minimal = 2        │
│             Jika Node 2 mati listrik ──> Node 1 & Node 3 tetap melayani     │
│             100% Write & Read tanpa downtime dan tanpa kehilangan data!     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Klasifikasi Kondisi Data Saat Terjadi Insiden (Data Durability States)

Saat terjadi gangguan besar (misal jaringan data center terputus atau cluster direstart paksa), SRE wajib membedakan 4 status keberadaan data telemetri:

| Kategori Status Data | Lokasi Fisik Data | Risiko & Cara Pemulihan (Recovery Path) |
|---|---|---|
| **1. Data in Collector Queue (Delayed)** | Berada di dalam disk lokal / WAL milik **Grafana Alloy** di worker node karena koneksi ke Mimir terputus. | **Data Belum Hilang!** Begitu koneksi jaringan ke Mimir Distributor pulih, Alloy akan mengirim ulang antrean WAL secara otomatis. Grafik yang tadinya bolong akan terisi kembali. |
| **2. Data in Ingester Memory/WAL (Hot Data < 2h)** | Berada di RAM dan PVC disk WAL milik **3 Pod Ingester**. Belum diunggah ke MinIO. | Selama minimal **2 dari 3 Ingester** masih hidup (atau PVC disk WAL-nya tidak terhapus saat pod restart), Ingester akan melakukan *WAL Replay* saat menyala dan data 2 jam terakhir pulih 100%. |
| **3. Data Durable in Object Storage (Cold Data > 2h)** | Sudah ter-flush menjadi blok TSDB di dalam bucket **MinIO S3 On-Prem**. | **Aman Permanen!** Meskipun seluruh pod Mimir di Kubernetes terhapus dan di-install ulang dari nol, begitu Mimir baru dihubungkan ke bucket MinIO yang sama, `store-gateway` langsung mengenali seluruh data historis! |
| **4. Data Truly Lost (Dropped)** | Dibuang karena antrean Alloy penuh melampaui batas waktu (`queue_dropped_total`) atau seluruh 3 PVC Ingester hancur bersamaan sebelum flush. | Data pada jendela waktu tersebut hilang permanen. Catat rentang waktu yang hilang di dokumen Postmortem agar tidak disalahartikan sebagai masa "0 trafik". |

---

## 3. Optimasi Kueri & Manajemen Rules di Mimir (`mimirtool`)

### 3.1 Bagaimana Query-Frontend Melindungi Mimir dari "Query of Death"
Di Prometheus tunggal, jika seorang pengguna tidak sengaja menjalankan kueri berat (misal menghitung `rate(container_cpu_usage_seconds_total[30d])` tanpa filter label), server Prometheus bisa langsung hang atau `OOMKilled`.

Mimir mencegah hal ini melalui tiga pelindung di **Query-Frontend** dan **Limits**:
1. **Query Splitting & Parallelism:** Kueri rentang 30 hari dipecah menjadi potongan interval 24 jam dan dikerjakan beramai-ramai oleh semua replica pod **Querier**.
2. **Results Caching:** Hasil perhitungan blok masa lalu (yang bersifat *immutable*) disimpan di cache sehingga jika 20 engineer membuka dashboard yang sama, Mimir tidak perlu menghitung ulang dari awal.
3. **Per-Tenant Query Guardrails (`limits`):**
   ```yaml
   limits:
     max_fetched_series_per_query: 50000     # Batalkan kueri yang menarik > 50.000 series!
     max_query_length: 90d                   # Larang satu kueri melebihi rentang 90 hari
     max_query_parallelism: 16               # Batas paralelisme per kueri
     query_timeout: 1m                       # Hentikan kueri yang berjalan lebih dari 1 menit
   ```

---

### 3.2 Mengelola Recording & Alerting Rules per Tenant dengan `mimirtool`

Pada arsitektur Mimir, komponen **Ruler** mengevaluasi Recording Rules dan Alerting Rules secara terpusat untuk setiap tenant (`X-Scope-OrgID`), lalu mengirim alert yang `Firing` ke **Mimir Alertmanager**.

Definisi file YAML rules sama persis dengan format Prometheus standar, namun diunggah ke bucket MinIO melalui API Mimir menggunakan CLI resmi **`mimirtool`**:

```yaml
# File: rules-prod-tenant.yaml
namespace: production_service_slos
groups:
  - name: payment_api_sli_recording
    interval: 30s
    rules:
      # 1. Recording Rule Latensi p95
      - record: cluster_service:http_request_duration_seconds:p95_rate5m
        expr: |
          histogram_quantile(
            0.95,
            sum by (le, cluster, service) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          )

      # 2. Alerting Rule Burn-Rate Error Cepat
      - alert: PaymentServiceFastErrorBurn
        expr: |
          (
            sum by (cluster, service) (rate(http_requests_total{service="payment-api", status=~"5.."}[5m]))
            /
            sum by (cluster, service) (rate(http_requests_total{service="payment-api"}[5m]))
          ) > 0.02
        for: 5m
        labels:
          severity: critical
          sre_owner: payment-squad
        annotations:
          summary: "Laju error tinggi (>2%) pada {{ $labels.service }} di {{ $labels.cluster }}"
          runbook_url: "https://git.internal.example.com/sre/runbooks/-/blob/main/payment/FastErrorBurn.md"
          no_data_policy: "alert-and-investigate"
```

---

## 4. Memisahkan Rantai Bukti (Evidence Chain) di Mimir

Sebagai seorang SRE Auditor, Anda tidak boleh menyimpulkan bahwa *"Sistem Observabilitas & SLO sudah berjalan sehat"* hanya karena melihat pod Mimir berstatus `Running`. Setiap klaim operasional membutuhkan bukti spesifik pada lapisannya:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│             RANTAI VERIFIKASI BUKTI (EVIDENCE CHAIN) MIMIR                  │
├──────────────────────────┬──────────────────────────────────────────────────┤
│ Klaim Tahapan            │ Bukti Validasi (Evidence) yang Wajib Ditunjukkan │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 1. Remote-Write Accepted │ Counter `prometheus_remote_storage_samples_total`│
│                          │ meningkat & kode status HTTP = 200 (0 failed).   │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 2. Ingester Ring Healthy │ Halaman `/ingester/ring` menunjukkan seluruh     │
│                          │ instance berstatus `ACTIVE` dengan token sehat.  │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 3. Storage Block Flushed │ Bucket MinIO `mimir-tsdb-blocks/<tenant>/`       │
│                          │ berisi direktori ULID blok (`meta.json`, `index`)│
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 4. Queryable via Gateway │ Eksekusi `curl` PromQL ke Query-Frontend         │
│                          │ mengembalikan vektor JSON untuk data > 2 jam.    │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 5. Ruler Evaluates Rules │ `mimirtool rules print` menampilkan rule aktif & │
│                          │ metrik recording rule muncul saat di-query.      │
└──────────────────────────┴──────────────────────────────────────────────────┘
```

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Kueri Grafana Gagal dengan Pesan `max_fetched_series_per_query limit exceeded`** | Pengguna menjalankan kueri PromQL yang terlalu luas (misal `{__name__=~".+"}` atau tanpa filter `service`/`namespace`). | 1. Edukasi pengguna untuk selalu menyertakan label filter spesifik.<br>2. Gunakan **Recording Rules** untuk mengagregasi metrik ber-kardinalitas tinggi terlebih dahulu sebelum ditampilkan di panel dashboard. |
| **Saat Upgrade Mimir, Data 1 Jam Terakhir Hilang** | Pod Ingester dimatikan secara paksa (`SIGKILL`) tanpa sempat melakukan *WAL Flush* ke MinIO atau PVC disk lokalnya dihapus (`emptyDir`). | 1. **Wajib gunakan `StatefulSet` dengan `PersistentVolumeClaim` (PVC)** untuk pod Ingester, jangan pernah gunakan `Deployment` dengan `emptyDir` untuk Ingester di produksi!<br>2. Lakukan rolling update satu per satu (`maxUnavailable: 1`) dan tunggu Ingester kembali `ACTIVE` di ring sebelum me-restart pod berikutnya. |
| **Recording Rule Sudah Di-upload tapi Metriknya Tidak Muncul** | 1. Komponen `ruler` belum diaktifkan pada konfigurasi Mimir.<br>2. Rule di-upload ke `X-Scope-OrgID` yang berbeda dengan tempat data metrik berada. | 1. Cek `/services` apakah modul `ruler` berstatus `Running`.<br>2. Pastikan argumen `--id=<TENANT_ID>` pada perintah `mimirtool rules load` sama persis dengan tenant pengirim metrik. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. MANAJEMEN RULES PER TENANT DENGAN MIMIRTOOL
# ==============================================================================

# Memeriksa sintaks dan keabsahan file rules secara lokal sebelum di-upload
mimirtool rules check rules-prod-tenant.yaml

# Melihat perbedaan (diff) antara file rules lokal di Git vs yang aktif di server Mimir
mimirtool rules diff rules-prod-tenant.yaml \
  --address=http://localhost:8080 \
  --id="onprem-prod-tenant"

# Mengunggah dan menerapkan (sync) file rules ke Mimir Ruler untuk tenant tertentu
mimirtool rules load rules-prod-tenant.yaml \
  --address=http://localhost:8080 \
  --id="onprem-prod-tenant"

# Menampilkan seluruh rules yang sedang berjalan di Mimir Ruler untuk tenant tersebut
mimirtool rules list \
  --address=http://localhost:8080 \
  --id="onprem-prod-tenant"

# ==============================================================================
# 2. OPERASI PEMELIHARAAN INGESTER & COMPACTOR
# ==============================================================================

# Memicu Ingester untuk segera mem-flush data di RAM ke Object Storage MinIO (Sebelum Maintenance)
curl -X POST http://localhost:8080/ingester/flush

# Memeriksa metrik kesehatan Compactor (Harus bernilai 1 = Sukses)
curl -s http://localhost:8080/metrics | grep "cortex_compactor_runs_failed_total"
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Faktor apakah yang paling menentukan besarnya kebutuhan **Memori RAM** pada klaster **Ingester** di Grafana Mimir?
- A. Durasi retensi penyimpanan blok di MinIO (apakah 30 hari atau 5 tahun).
- B. Jumlah **Active Series** saat ini dikalikan dengan **Replication Factor** (misal RF=3), karena Ingester hanya menahan data aktif 2 jam terakhir di dalam RAM sebelum mem-flush blok ke Object Storage.
- C. Jumlah dashboard yang tersimpan di database Grafana.
- D. Kecepatan kipas pendingin server bare-metal.

#### Soal 2
Mengapa pod **Ingester** pada deployment Grafana Mimir tingkat produksi **wajib** dijalankan sebagai **StatefulSet** dengan *PersistentVolumeClaim (PVC)* untuk menyimpan direktori *Write-Ahead Log (WAL)*, dan **bukan** sebagai Deployment biasa dengan `emptyDir`?
- A. Karena Kubernetes melarang penggunaan Deployment untuk program berbahasa Go.
- B. Agar apabila pod Ingester mengalami crash atau direstart saat upgrade node, proses Ingester baru dapat membaca ulang file WAL dari PVC yang tetap utuh, sehingga data metrik 2 jam terakhir yang belum sempat di-flush ke MinIO tidak hilang.
- C. Karena `StatefulSet` tidak membutuhkan CPU sama sekali.
- D. Agar alamat IP pod berubah setiap 5 detik.

#### Soal 3
Apa fungsi utama dari komponen **Compactor** di dalam siklus hidup penyimpanan jangka panjang Grafana Mimir?
- A. Melakukan HTTP scrape ke `node_exporter` di setiap server.
- B. Mengirim pesan notifikasi ke Slack dan PagerDuty.
- C. Menggabungkan blok-blok TSDB kecil (2 jam) di Object Storage (MinIO) menjadi blok yang lebih besar, menghilangkan sampel duplikat hasil replikasi 3 Ingester (*deduplication*), serta menghapus blok yang sudah melewati batas usia retensi (`compactor_blocks_retention_period`).
- D. Mengenkripsi password login pengguna aplikasi web.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Perusahaan Anda berencana memigrasikan sistem metrik dari 1 server Prometheus lama ke klaster **Grafana Mimir + MinIO On-Premises**.
Dari data audit saat ini:
- Jumlah total *Active Series*: **300.000 series**
- Interval Scrape: **15 detik**
- Target Retensi Arsip di MinIO: **180 Hari (6 Bulan)**
- Target High Availability: Tahan terhadap matinya 1 server fisik penuh (**Replication Factor = 3**).

**Pertanyaan & Tugas Anda:**
1. Hitunglah estimasi **Total RAM** yang dibutuhkan untuk klaster Ingester (gunakan asumsi `8 KB/series` dan `30% safety headroom`)!
2. Hitunglah estimasi **Kapasitas Penyimpanan Bucket MinIO** yang dibutuhkan untuk menyimpan data selama 180 hari tersebut (gunakan asumsi `2 Bytes/sample` dan `25% overhead compaction/index`)!
3. Jika pada suatu malam koneksi jaringan antara worker node K8s dan server Mimir terputus selama 20 menit akibat penggantian kabel switch, jelaskan mengapa grafik di Mimir tetap bisa pulih utuh tanpa lubang (*gap*) setelah kabel switch tersambung kembali!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Di Mimir, ukuran RAM Ingester berbanding lurus dengan jumlah *active series* × *replication factor*, sedangkan durasi retensi jangka panjang (bulanan/tahunan) hanya membebani ruang disk Object Storage (MinIO).
2. **B.** Tanpa PVC untuk menyimpan WAL, setiap kali pod Ingester restart dengan `emptyDir`, isi memorinya langsung lenyap. Jika lebih dari 1 Ingester restart berdekatan (melampaui toleransi kuorum), data 2 jam terakhir yang belum masuk ke MinIO akan hilang.
3. **C.** Ketiga Ingester masing-masing mengunggah blok 2 jam yang berisi data yang sama ke MinIO. **Compactor**-lah yang menyatukan ketiganya menjadi satu blok tunggal yang terdeduplikasi dan menerapkan penghapusan masa retensi.

#### Solusi Soal Kasus Nyata:
1. **Perhitungan Total RAM Klaster Ingester:**
   - Replicated Series = `300.000 × 3 (RF) = 900.000 series`.
   - Memori Dasar = `900.000 × 8 KB = 7.200.000 KB ≈ 7,2 GB`.
   - Total RAM dengan 30% Headroom = `7,2 GB × 1.3 = 9,36 GB` (Dapat dibagi menjadi 3 Pod Ingester dengan masing-masing limit RAM `3.5Gi` – `4Gi`).
2. **Perhitungan Kapasitas Bucket MinIO (180 Hari):**
   - Laju sampel = `300.000 series ÷ 15 detik = 20.000 samples/detik`.
   - Durasi dalam detik = `180 hari × 86.400 detik = 15.552.000 detik`.
   - Ukuran sampel murni = `20.000 × 2 Bytes × 15.552.000 = 622.080.000.000 Bytes ≈ 622,08 GB`.
   - Total Kapasitas dengan 25% Overhead = `622,08 GB × 1.25 ≈ 777,6 GB` (Alokasikan kuota bucket sekitar **800 GB – 850 GB**).
3. **Mekanisme Pemulihan Tanpa Lubang Grafik (Alloy WAL Backpressure):**
   - Selama 20 menit jaringan ke Mimir terputus, **Grafana Alloy** di setiap node tetap melakukan scrape lokal ke pod setiap 15 detik dan menyimpannya ke dalam **Write-Ahead Log (WAL) lokal Alloy** beserta timestamp asli saat scrape terjadi.
   - Begitu switch jaringan tersambung kembali (masih di dalam batas `out_of_order_time_window: 30m` milik Mimir), Alloy mengirimkan seluruh antrean sampel yang tertunda ke Mimir, sehingga grafik terisi penuh tanpa kehilangan satu detik pun!

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai perencanaan kapasitas, HA, dan pengelolaan rules di Grafana Mimir:
- Praktikkan di [LAB-02 — Mimir Retention, Capacity, dan Recovery](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.3-mimir/lab/LAB-02-mimir-retention-capacity-recovery.md).
- Lanjutkan ke [Modul 7.4 — Grafana Loki (Logs) & Tempo (Traces)](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.4-loki-tempo/README.md) untuk melengkapi kedua pilar observabilitas lainnya dan mengkorelasikan ketiganya secara instan.
