# 02 — Rules, Alertmanager, Retention, dan Cardinality

> **Ringkasan:** Mempelajari cara mengotomatisasi pra-kalkulasi kueri berat menggunakan **Recording Rules**, merancang deteksi anomali presisi tinggi dengan **Alerting Rules**, mengelola perutean dan peredaman notifikasi di **Alertmanager**, serta menjaga stabilitas memori dan disk Prometheus dari ancaman **High Cardinality Explosion**.

---

## 1. Konsep & Arsitektur Evaluasi Rule dan Alerting

### 1.1 Analogi Dunia Nyata: "Buku Rekap Kasir & Pusdalops Darurat 112"
Bayangkan bagaimana sebuah kota pintar mengelola laporan transaksi dan keadaan darurat:
1. **Buku Rekapitulasi Harian (*Recording Rules*):** Jika Direktur Keuangan bertanya setiap pagi, *"Berapa total penjualan seluruh cabang selama 5 menit terakhir?"*, kasir tidak perlu menghitung ulang jutaan struk belanja dari nol setiap kali ditanya. Setiap 1 menit, mesin kasir otomatis menjumlahkan sub-total dan mencatatnya di baris rekap baru. Saat direktur membuka layar dashboard, angkanya langsung muncul dalam milidetik!
2. **Timer Verifikasi Detektor Asap (*Alerting Rules & Klausul `for:`*):** Jika seseorang menyalakan korek api di dekat sensor asap selama 2 detik, alarm gedung tidak boleh langsung membangunkan seluruh pemadam kebakaran kota. Sensor masuk ke status **Pending** dan menunggu selama 2 menit (`for: 2m`). Jika setelah 2 menit asap tebal masih terus terdeteksi, barulah status berubah menjadi **Firing** dan dikirim ke pusat komando.
3. **Pusat Komando Darurat 112 (*Alertmanager*):** Ketika satu gardu listrik meledak, 200 rumah di jalan yang sama melapor secara bersamaan. Operator 112 (**Alertmanager**) tidak mengirim 200 pesan WhatsApp terpisah ke teknisi PLN. Operator melakukan:
   - **Grouping:** Menggabungkan 200 laporan menjadi **1 pesan ringkas**: *"200 rumah di Sektor A mati listrik"*.
   - **Inhibition:** Jika sudah ada laporan *"Gardu Induk Sektor A Meledak (Critical)"*, maka 200 alarm kecil *"Lampu Teras Rumah Nomor X Mati (Warning)"* otomatis dibungkam agar teknisi fokus pada akar masalah utama.
   - **Silence:** Jika sedang ada jadwal pemeliharaan rutin pukul 01:00–03:00 pagi, alarm untuk gardu tersebut dibisukan sementara dengan batas waktu kedaluwarsa yang jelas.

---

### 1.2 Siklus Hidup Alert (Alert Lifecycle) & Alur Data

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   ALUR EVALUASI RULE HINGGA NOTIFIKASI                      │
│                                                                             │
│  [ PROMETHEUS SERVER ]                                                      │
│  Evaluasi setiap `evaluation_interval` (15s):                               │
│                                                                             │
│  ┌──────────────┐   Kondisi Terpenuhi    ┌──────────────┐   Bertahan selama │
│  │   INACTIVE   │ ─────────────────────> │   PENDING    │ ── durasi `for:` ─┐
│  │ (Normal / OK)│ <───────────────────── │ (Wait Timer) │                   │
│  └──────────────┘   Pulih sebelum `for:` └──────────────┘                   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────────┘
│  ▼
│  ┌──────────────┐
│  │    FIRING    │ ──(Kirim HTTP POST JSON setiap siklus)──┐
│  └──────────────┘                                         │
│                                                           ▼
│  [ ALERTMANAGER CLUSTER ]                                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Deduplication (Jika dikirim dari 2 server Prometheus HA)           │  │
│  │ 2. Silencing     (Cek apakah cocok dengan jadwal maintenance aktif)   │  │
│  │ 3. Inhibition    (Bungkam alert gejala jika alert root-cause Firing)  │  │
│  │ 4. Grouping      (Kumpulkan ke dalam 1 bundel berdasarkan group_by)   │  │
│  │ 5. Routing Tree  (Cocokkan label severity/owner ke Receiver tujuan)   │  │
│  └───────────────────────────────────────────┬───────────────────────────┘  │
│                                              │                              │
│                                              ▼                              │
│                    [ Webhook / Slack / Telegram / PagerDuty ]               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Recording Rules & Alerting Rules Tingkat Produksi

### 2.1 Standar Penamaan Recording Rules (`level:metric:operations`)
Agar kueri mudah dibaca dan dikelola lintas tim, Prometheus menetapkan konvensi nama *Recording Rule* tiga bagian yang dipisahkan oleh titik dua (`:`):
- **`level`:** Label agregasi yang tersisa pada hasil kueri (misal `service`, `instance`, atau `namespace_service`).
- **`metric`:** Nama metrik dasar tanpa akhiran `_total` atau `_bucket` (misal `http_requests` atau `http_request_duration_seconds`).
- **`operations`:** Fungsi matematika dan jendela waktu yang diterapkan (misal `rate5m`, `ratio_rate5m`, atau `p95_rate5m`).

**Contoh Valid:**
- `service:http_requests:rate5m`
- `service:http_requests_error_ratio:rate5m`
- `service:http_request_duration_seconds:p95_rate5m`

---

### 2.2 Implementasi 4 Skenario Kurikulum Wajib (`prometheus-rules.yml`)

Berikut adalah konfigurasi lengkap **Recording Rules** dan **Alerting Rules** untuk 4 skenario standar produksi SRE (Error Rate > 5%, Latency p95 > 500ms, Disk Usage > 85%, dan Node Down):

```yaml
groups:
  # ============================================================================
  # GRUP 1: RECORDING RULES (Pra-Kalkulasi Metrik Layanan)
  # ============================================================================
  - name: sre_service_golden_signals_recording
    interval: 15s
    rules:
      # 1. Total Request Rate per Service (5m window)
      - record: service:http_requests:rate5m
        expr: |
          sum by (sre_environment, service) (
            rate(http_requests_total[5m])
          )

      # 2. Rasio Error HTTP 5xx per Service (0.0 sampai 1.0)
      - record: service:http_requests_error_ratio:rate5m
        expr: |
          sum by (sre_environment, service) (
            rate(http_requests_total{status=~"5.."}[5m])
          )
          /
          (
            sum by (sre_environment, service) (
              rate(http_requests_total[5m])
            ) > 0
          )

      # 3. Latensi Persentil ke-95 (p95) dalam Detik per Service
      - record: service:http_request_duration_seconds:p95_rate5m
        expr: |
          histogram_quantile(
            0.95,
            sum by (le, sre_environment, service) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          )

  # ============================================================================
  # GRUP 2: ALERTING RULES (4 Skenario Inti SRE On-Premises)
  # ============================================================================
  - name: sre_production_core_alerts
    interval: 15s
    rules:
      # SKENARIO 1: Error Rate > 5% (0.05) selama 5 menit berturut-turut
      - alert: ServiceHighErrorRate
        expr: service:http_requests_error_ratio:rate5m{sre_environment="production"} > 0.05
        for: 5m
        labels:
          severity: critical
          sre_owner: app-platform
        annotations:
          summary: "Error rate HTTP 5xx tinggi pada service {{ $labels.service }}"
          description: "Service {{ $labels.service }} di {{ $labels.sre_environment }} mengalami error rate {{ $value | humanizePercentage }} (ambang batas > 5%) selama 5 menit."
          runbook_url: "https://git.internal.example.com/sre/runbooks/-/blob/main/alerts/ServiceHighErrorRate.md"
          no_data_policy: "investigate-scrape-path"

      # SKENARIO 2: Latensi p95 > 500ms (0.5 detik) selama 10 menit
      - alert: ServiceHighLatencyP95
        expr: service:http_request_duration_seconds:p95_rate5m{sre_environment="production"} > 0.5
        for: 10m
        labels:
          severity: warning
          sre_owner: app-platform
        annotations:
          summary: "Latensi p95 tinggi pada service {{ $labels.service }}"
          description: "Latensi p95 mencapai {{ $value | humanizeDuration }} (batas > 500ms) selama 10 menit terakhir."
          runbook_url: "https://git.internal.example.com/sre/runbooks/-/blob/main/alerts/ServiceHighLatencyP95.md"

      # SKENARIO 3: Penggunaan Disk > 85% pada Partisi Fisik/Utama
      - alert: HostDiskSpaceRunningOut
        expr: |
          100 * (
            1 - (
              node_filesystem_avail_bytes{fstype=~"ext4|xfs", mountpoint!~"^/(run|var/lib/docker|var/lib/kubelet).*"}
              /
              node_filesystem_size_bytes{fstype=~"ext4|xfs", mountpoint!~"^/(run|var/lib/docker|var/lib/kubelet).*"}
            )
          ) > 85
        for: 15m
        labels:
          severity: warning
          sre_owner: infra-onprem
        annotations:
          summary: "Kapasitas disk hampir penuh di node {{ $labels.instance }}"
          description: "Partisi {{ $labels.mountpoint }} pada {{ $labels.instance }} telah terpakai {{ printf \"%.1f\" $value }}% (> 85%)."
          runbook_url: "https://git.internal.example.com/sre/runbooks/-/blob/main/alerts/HostDiskSpaceRunningOut.md"

      # SKENARIO 4: Node Exporter / Target Down selama 3 menit
      - alert: HostNodeDown
        expr: up{job="node-exporter-baremetal"} == 0
        for: 3m
        labels:
          severity: critical
          sre_owner: infra-onprem
        annotations:
          summary: "Node atau Exporter tidak merespons pada {{ $labels.instance }}"
          description: "Prometheus gagal melakukan scrape ke {{ $labels.instance }} selama lebih dari 3 menit."
          runbook_url: "https://git.internal.example.com/sre/runbooks/-/blob/main/alerts/HostNodeDown.md"
```

---

## 3. Arsitektur Konfigurasi Alertmanager (`alertmanager.yml`)

### 3.1 Tiga Parameter Waktu Grouping yang Wajib Dipahami
Saat Alertmanager mengelompokkan beberapa alert ke dalam satu grup (`group_by`), terdapat 3 parameter waktu yang mengatur kapan pesan dikirim:
1. **`group_wait` (misal `30s`):** Berapa lama Alertmanager menunggu sejak alert **pertama** masuk di grup baru sebelum mengirim notifikasi pertama. Tujuannya: memberi waktu bagi alert lain dari insiden yang sama untuk ikut berkumpul agar terkirim dalam **satu** pesan gabungan.
2. **`group_interval` (misal `5m`):** Jika grup tersebut sudah pernah mengirim notifikasi, lalu ada alert **baru** yang bergabung ke grup yang sama, berapa lama harus menunggu sebelum mengirim update grup tersebut.
3. **`repeat_interval` (misal `4h`):** Jika tidak ada perubahan tetapi alert di dalam grup tersebut **masih belum diperbaiki (tetap Firing)**, berapa lama sebelum mengirimkan pengingat ulang (*reminder*) ke kanal on-call.

---

### 3.2 Contoh Konfigurasi `alertmanager.yml` yang Aman (Secret-Safe)

```yaml
global:
  resolve_timeout: 5m

# Pohon Perutean (Routing Tree)
route:
  # Kelompokkan berdasarkan nama alert, environment, dan tim pemilik
  group_by: ['alertname', 'sre_environment', 'sre_owner']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-sre-channel'

  routes:
    # Rute 1: Alert Critical di Production langsung ke Pager/On-Call Utama
    - matchers:
        - severity = "critical"
        - sre_environment = "production"
      receiver: 'oncall-critical-webhook'
      group_wait: 10s
      repeat_interval: 1h
      continue: false

    # Rute 2: Alert Infrastruktur ke kanal tim Infra
    - matchers:
        - sre_owner = "infra-onprem"
      receiver: 'infra-team-webhook'
      continue: false

# Aturan Peredaman (Inhibition Rules)
inhibit_rules:
  # Jika HostNodeDown (Critical) sedang Firing pada suatu instance,
  # bungkam alert warning lainnya dari instance yang sama agar tidak spam!
  - source_matchers:
      - alertname = "HostNodeDown"
      - severity = "critical"
    target_matchers:
      - severity = "warning"
    equal: ['instance', 'sre_environment']

# Daftar Penerima (Receivers) - TANPA HARDCODED SECRET!
receivers:
  - name: 'default-sre-channel'
    webhook_configs:
      - url_file: '/etc/alertmanager/secrets/<DEFAULT_WEBHOOK_URL_FILE>'
        send_resolved: true

  - name: 'oncall-critical-webhook'
    webhook_configs:
      - url_file: '/etc/alertmanager/secrets/<CRITICAL_ONCALL_WEBHOOK_URL_FILE>'
        send_resolved: true

  - name: 'infra-team-webhook'
    webhook_configs:
      - url_file: '/etc/alertmanager/secrets/<INFRA_WEBHOOK_URL_FILE>'
        send_resolved: true
```

> [!IMPORTANT]
> **Mengapa Menggunakan `url_file` atau `http_config.authorization.credentials_file`?**
> Webhook URL (seperti Slack Incoming Webhook, Discord Webhook, atau Telegram Bot URL) sering kali mengandung token rahasia di dalam path URL-nya. Menyimpan URL tersebut langsung di `alertmanager.yml` akan mengekspos token di repositori Git dan di halaman web `/#/status` Alertmanager UI! Selalu gunakan `url_file` yang merujuk pada file yang di-mount dari Kubernetes Secret.

---

## 4. Ledakan Kardinalitas (Cardinality Explosion) & Perencanaan Kapasitas

### 4.1 Apa Itu High Cardinality dan Mengapa Bisa Mematikan Server?
Di Prometheus, **setiap kombinasi unik dari nama metrik dan nilai label akan menciptakan satu Time-Series baru di dalam indeks RAM (Head Block)**.

Mari kita hitung perkalian kardinalitasnya:
- Metrik dasar: `http_requests_total`
- Label `method`: 4 nilai (`GET`, `POST`, `PUT`, `DELETE`)
- Label `status`: 5 nilai (`200`, `201`, `400`, `404`, `500`)
- Label `instance`: 10 replika pod
- **Total Series Aman:** `4 × 5 × 10 = 200 active series` (Sangat ringan, hanya butuh beberapa Kilobyte RAM).

**Sekarang bayangkan seorang developer menambahkan label `user_id` (ada 500.000 pengguna aktif) dan `endpoint_path` mentah tanpa normalisasi:**
- **Total Series Meledak:** `200 × 500.000 = 100.000.000 (100 Juta) active series!`
- Akibatnya: Penggunaan RAM Prometheus melonjak puluhan Gigabyte dalam hitungan menit hingga pod Prometheus mati terkena **`OOMKilled` (Exit Code 137)** dan seluruh sistem pemantauan perusahaan lumpuh total!

### 4.2 Daftar Hitam Label (Forbidden Labels di Prometheus)
**DILARANG KERAS** memasukkan data dengan nilai tak terbatas (*unbounded*) berikut ke dalam label metrik Prometheus:
1. `user_id`, `customer_id`, `email`, `phone_number`, atau `session_token` (Selain merusak kardinalitas, ini juga melanggar hukum privasi PII!).
2. `request_id`, `trace_id`, atau `transaction_uuid` (Gunakan fitur **Exemplars** untuk menghubungkan metrik ke trace, bukan label!).
3. `ip_address` klien publik atau `user_agent` browser.
4. `url_path` mentah yang memuat ID dinamis (misal `/api/v1/orders/98237489` harus dinormalisasi menjadi `/api/v1/orders/:id`).
5. Pesan error mentah (`exception_message` atau `sql_query`).

---

### 4.3 Rumus Perencanaan Kapasitas Penyimpanan (TSDB Disk Sizing)

Sebelum menaikkan durasi retensi (`--storage.tsdb.retention.time=30d`), SRE wajib menghitung kebutuhan disk secara matematis:

```text
Rumus Kapasitas Disk TSDB:
Total Disk (Bytes) = (Active Series ÷ Scrape Interval) × Bytes per Sample × Retention Seconds × Safety Factor (1.3)
```

**Contoh Perhitungan Nyata di Cluster On-Premises:**
- Jumlah Active Series: `150.000 series`
- Scrape Interval: `15 detik` ➔ Laju sampel masuk = `150.000 ÷ 15 = 10.000 samples/detik`
- Ukuran rata-rata per sampel setelah kompresi Gorilla/XOR Prometheus: `~2 Bytes`
- Durasi Retensi: `30 hari` = `30 × 86.400 detik = 2.592.000 detik`
- Faktor Keamanan (Overhead Index + WAL + Compaction ruang kerja 30%): `1.3`
- **Total Kebutuhan Disk:**
  `10.000 samples/s × 2 Bytes × 2.592.000 s × 1.3 = 67.392.000.000 Bytes ≈ 67.4 GB`
- **Rekomendasi Provisioning PVC:** Alokasikan minimal **80 GB – 100 GB** SSD/NVMe.

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Masalah Operasional | Indikasi & Gejala | Langkah Mitigasi & Perbaikan SRE |
|---|---|---|
| **Alert Flood (Badai Notifikasi)** | Ratusan pesan masuk ke kanal on-call dalam waktu bersamaan saat 1 switch/node mati. | 1. Periksa `group_by` di `alertmanager.yml` (pastikan tidak mengelompokkan berdasarkan label unik seperti `pod_ip`).<br>2. Tambahkan `inhibit_rules` agar alert level node membungkam alert level pod di node tersebut. |
| **Memory Prometheus Naik Terus hingga `OOMKilled`** | Penggunaan RAM pod Prometheus melonjak tajam setelah rilis aplikasi baru. | 1. Buka halaman **Status -> TSDB Status** di UI Prometheus (`/tsdb-status`).<br>2. Lihat tabel **"Top 10 label names with high memory usage"** dan **"Top 10 series count by metric names"**.<br>3. Gunakan `metric_relabel_configs` dengan `action: labeldrop` untuk membuang label penyebab ledakan kardinalitas sebagai pertolongan pertama. |
| **Alert Tidak Pernah Firing Padahal Layanan Mati (Silent Failure)** | Aplikasi rusak total dan tidak menerima trafik (`0 req/s`), namun alert error rate tidak berbunyi. | Kueri `rate(errors[5m]) / rate(requests[5m])` menghasilkan `0 / 0 = NaN` (Kosong) saat trafik mati total! Pasang alert pendamping **Absent / DeadManSwitch** atau gunakan `absent_over_time()` untuk mendeteksi hilangnya trafik secara tiba-tiba. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. VALIDASI & UNIT TEST RECORDING/ALERTING RULES DENGAN PROMTOOL
# ==============================================================================

# Memeriksa sintaks file rules YAML sebelum di-commit ke GitOps
promtool check rules /etc/prometheus/rules/prometheus-rules.yml

# Menjalankan automated unit test terhadap logika perhitungan rule
promtool test rules /etc/prometheus/tests/rules-test.yml

# ==============================================================================
# 2. OPERASI ALERTMANAGER DENGAN AMTOOL (CLI)
# ==============================================================================

# Memvalidasi file konfigurasi alertmanager.yml
amtool check-config /etc/alertmanager/alertmanager.yml

# Menguji ke receiver mana sebuah alert dengan label tertentu akan dirutekan
amtool config routes test --config.file=/etc/alertmanager/alertmanager.yml \
  severity=critical sre_environment=production sre_owner=infra-onprem

# Melihat daftar alert yang sedang aktif (Firing) di Alertmanager
amtool alert query --alertmanager.url=http://localhost:9093

# Membuat Silence resmi dengan alasan tiket dan durasi kedaluwarsa 2 jam
amtool silence add alertname="HostNodeDown" instance="192.168.50.12:9100" \
  --author="sre-oncall@example.com" \
  --comment="MAINT-402: Penggantian RAM fisik server worker-02" \
  --duration="2h" \
  --alertmanager.url=http://localhost:9093

# ==============================================================================
# 3. AUDIT KARDINALITAS PROMETHEUS VIA HTTP API
# ==============================================================================

# Mengambil statistik TSDB (Top 10 metrik dan label dengan kardinalitas tertinggi)
curl -s http://localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName[:10]'
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Apa tujuan utama menambahkan klausa `for: 5m` pada sebuah definisi **Alerting Rule** di Prometheus?
- A. Agar Prometheus hanya melakukan scrape ke target tersebut setiap 5 menit sekali.
- B. Agar alert hanya akan dihapus dari database setelah 5 menit selesai diperbaiki.
- C. Untuk mencegah *flapping/noisy alert* akibat lonjakan sesaat; kondisi ambang batas harus terus terpenuhi tanpa putus selama 5 menit (status `Pending`) sebelum berubah menjadi `Firing` dan dikirim ke Alertmanager.
- D. Untuk membatasi durasi bunyi sirine di ponsel engineer maksimal selama 5 menit.

#### Soal 2
Manakah dari penambahan label berikut pada metrik `http_requests_total` yang akan memicu bencana **High Cardinality Explosion** di dalam memori TSDB Prometheus?
- A. `sre_environment="production"` (3 kemungkinan nilai: dev, staging, production).
- B. `method="POST"` (5 kemungkinan nilai metode HTTP).
- C. `customer_email` dan `request_uuid` dari setiap transaksi pengguna yang masuk.
- D. `cluster="k3s-onprem-01"` (2 kemungkinan nilai cluster).

#### Soal 3
Ketika satu node fisik bare-metal (`worker-03`) mati listrik, terdapat 15 pod aplikasi di dalamnya yang ikut mati. Fitur Alertmanager apakah yang bertugas **membungkam** 15 alert level pod/aplikasi selama alert `HostNodeDown` untuk `worker-03` sedang berstatus `Firing`?
- A. `repeat_interval`
- B. `inhibit_rules` (Inhibition)
- C. Recording Rules
- D. TSDB Compaction

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Pada hari Senin pagi pukul 09:00, tim developer baru saja merilis fitur pelacakan pesanan di `shipping-service`. Pada pukul 09:20, pod Prometheus di cluster produksi mengalami restart berulang kali karena `OOMKilled` (kehabisan memori RAM 8 GB).

Saat Anda berhasil mengakses endpoint `/api/v1/status/tsdb` sesaat setelah pod menyala, Anda melihat output berikut:
- `http_client_requests_total`: **2.450.000 series**
- Label dengan pasangan nilai terbanyak: `target_url` (contoh nilai: `https://partner-courier.com/track?resi=JNE9981231`, `https://partner-courier.com/track?resi=JNE9981232`, dst.).

**Pertanyaan & Tugas Anda:**
1. Jelaskan mengapa label `target_url` tersebut membuat Prometheus kehabisan memori RAM 8 GB dalam waktu 20 menit!
2. Tanpa menunggu tim developer mengubah kode aplikasi mereka dan melakukan build ulang (yang memakan waktu 1 jam), konfigurasi darurat apa yang bisa langsung Anda terapkan di `prometheus.yml` untuk menyelamatkan server Prometheus sekarang juga?
3. Bagaimana seharusnya tim developer mendesain label metrik tersebut dengan benar?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **C.** Klausul `for:` bertindak sebagai filter peredam kejut waktu (*debounce timer*). Jika lonjakan CPU/error hanya terjadi selama 30 detik lalu normal kembali, alert yang berada di fase `Pending` akan langsung kembali ke `Inactive` tanpa membangunkan engineer on-call.
2. **C.** `customer_email` dan `request_uuid` memiliki kardinalitas tak terbatas (*unbounded cardinality*). Setiap transaksi baru menciptakan *time-series* baru di indeks RAM Prometheus dan memuat data pribadi (PII) yang dilarang masuk ke sistem metrik.
3. **B.** **Inhibition (`inhibit_rules`)** memungkinkan kita menekan notifikasi untuk sekumpulan alert hilir (*target matchers*) apabila sudah ada alert hulu/akar masalah (*source matchers*) yang sedang aktif dan berbagi label identitas yang sama (seperti `node` atau `instance`).

#### Solusi Soal Kasus Nyata:
1. **Penyebab Kehabisan RAM:**
   - Developer memasukkan parameter query dinamis (`?resi=JNE...`) ke dalam label `target_url`. Karena setiap nomor resi pengiriman paket berbeda untuk setiap pelanggan, jutaan resi unik menghasilkan 2,45 juta *time-series* terpisah di dalam Head Block RAM Prometheus.
2. **Mitigasi Darurat di `prometheus.yml` (Tanpa Menunggu Build Aplikasi):**
   - Tambahkan blok **`metric_relabel_configs`** pada job scrape tersebut dengan aksi **`labeldrop`** untuk membuang label `target_url` sebelum disimpan ke dalam TSDB:
     ```yaml
     metric_relabel_configs:
       - regex: "target_url"
         action: labeldrop
     ```
   - Lakukan `curl -X POST http://localhost:9090/-/reload`.
3. **Perbaikan Desain di Kode Aplikasi:**
   - Buang parameter query dinamis dan ID unik dari label metrik. Gunakan label statis/terbatas seperti `partner="jne"` dan `operation="track_shipment"`. Jika detail nomor resi dibutuhkan untuk investigasi, catat di sistem **Log (Loki)** atau **Trace Attribute (Tempo)** setelah melalui kebijakan redaksi PII yang sesuai.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai pembuatan Rules, manajemen Alertmanager, dan pengendalian kardinalitas di Prometheus:
- Praktikkan di [LAB-02 — PromQL, Rules, dan Alertmanager](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.1-prometheus/lab/LAB-02-promql-rules-alertmanager.md).
- Lanjutkan ke [Modul 7.2 — Grafana Alloy Telemetry Pipeline](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.2-alloy-telemetry-pipeline/README.md) untuk membangun pipeline pengumpul terpadu untuk Metrics, Logs, dan Traces.
