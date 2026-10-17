# 02 — Alert Rule, Contact Point, Routing, dan Notification

> **Ringkasan:** Membedah arsitektur **Grafana Unified Alerting** dan batas kepemilikannya dengan **Mimir/Prometheus Alertmanager**, merancang kontrak **Alert Rule** dengan kebijakan **`noDataState`** dan **`execErrState`** yang aman (*fail-safe*), mengelola **Notification Policies & Contact Points as Code** tanpa mengekspos rahasia, serta menelusuri rantai bukti pengiriman notifikasi dari evaluasi kueri hingga tindakan operator.

---

## 1. Konsep & Batas Arsitektur Sistem Peringatan (Alerting Boundary)

### 1.1 Analogi Dunia Nyata: "Rantai Komando Pemadam Kebakaran 5 Tahap"
Ketika terjadi kebakaran di sebuah pabrik, keselamatan gedung bergantung pada tidak terputusnya **5 mata rantai**:
1. **Sensor Pendeteksi (*Query & Rule Evaluator*):** Kamera termal memeriksa suhu ruangan setiap 15 detik. Jika suhu > 80°C bertahan selama 2 menit (`for: 2m`), sensor mengeluarkan sinyal **FIRING**. (Jika kabel kamera putus, sensor tidak boleh melapor "Suhu Aman 25°C", melainkan harus melapor **SENSOR ERROR / NO DATA**!).
2. **Papan Wesel Operator (*Notification Policy / Routing Tree*):** Membaca label lokasi sinyal (`Gedung="Kimia"`, `Tingkat="Kritis"`) untuk menentukan tim spesialis mana yang harus dihubungi, sambil menggabungkan sinyal dari ruangan yang bersebelahan (*Grouping*).
3. **Jadwal Piket & Peredam (*Silences & Mute Timings*):** Memeriksa apakah ruangan tersebut sedang menjalani uji las terjadwal yang memiliki surat izin resmi.
4. **Pesawat Telepon / Radio Panggil (*Contact Point*):** Mengirimkan paket pesan melalui jalur komunikasi (Slack, Telegram, Webhook, PagerDuty) menggunakan kunci autentikasi yang tersimpan di brankas.
5. **Petugas Piket Merespons (*Operator Action*):** Petugas menerima bunyi sirine, menekan tombol *Acknowledge*, dan membuka buku panduan pemadaman (**Runbook**).

Jika salah satu saja dari kelima mata rantai ini putus, pabrik akan habis terbakar tanpa ada satu pun petugas yang datang!

---

### 1.2 Alur Sinyal End-to-End & Titik Pemeriksaan Bukti (Evidence Checkpoints)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│             ALUR SINYAL PERINGATAN & 5 TITIK BUKTI (EVIDENCE)               │
│                                                                             │
│  [1. Datasource (Mimir/Loki)]                                               │
│         │ (Eksekusi Kueri setiap 15s–1m)                                    │
│         ▼                                                                   │
│  [2. Rule Evaluator] ──> Status: Normal | Pending | Firing | NoData | Error │
│         │                (Bukti 1: State Transition Log & Timestamp)        │
│         ▼                                                                   │
│  [3. Notification Policy Router]                                            │
│      ├─ Cek Matchers (severity, sre_environment, sre_owner)                 │
│      ├─ Cek Silences / Mute Timings / Inhibition                            │
│      └─ Kelompokkan berdasarkan `group_by`                                  │
│         │                (Bukti 2: Route Match & Group ID)                  │
│         ▼                                                                   │
│  [4. Contact Point Integration]                                             │
│      └─ Kirim HTTP POST ke Webhook / PagerDuty / Slack API                  │
│         │                (Bukti 3: HTTP 200 Accepted / Retry Counter)       │
│         ▼                                                                   │
│  [5. On-Call Operator] ──> Buka `runbook_url` & Lakukan Mitigasi!           │
│                          (Bukti 4: Operator Ack & Incident Timeline)        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 1.3 Memilih Kepemilikan: Mimir/Prometheus Ruler vs Grafana Unified Alerting

Di dalam ekosistem observabilitas ini, Anda memiliki dua mesin yang sama-sama bisa mengevaluasi alert: **Mimir/Prometheus Ruler** dan **Grafana Unified Alerting**.

> [!IMPORTANT]
> **Aturan Kepemilikan SRE (Anti-Duplikasi):**
> Jangan pernah mendefinisikan alert yang sama persis di Mimir Ruler DAN di Grafana Unified Alerting secara bersamaan tanpa rencana pembagian tugas!

| Kriteria | Mimir / Prometheus Ruler | Grafana Unified Alerting |
|---|---|---|
| **Lokasi Evaluasi** | Berjalan langsung di dekat penyimpanan TSDB (Mimir backend). | Berjalan di dalam server Grafana (menarik data lintas datasource). |
| **Kelebihan Utama** | **Paling andal untuk Alert Kritis & SLO!** Tetap berjalan meskipun UI Grafana sedang mati atau direstart. | Bisa menggabungkan kueri dari **banyak datasource berbeda** (misal Mimir + Loki + PostgreSQL) dalam 1 rule. |
| **Rekomendasi Penggunaan di Kurikulum** | Gunakan untuk **Core Infrastructure, Golden Signals (RED/USE), dan SLO Burn-Rate Alerts**. | Gunakan untuk **Business/Cross-Datasource Alerts** atau visualisasi status alert terpusat melalui Alertmanager eksternal. |

---

## 2. Kontrak Alert Rule & Kebijakan `NoData` / `Error` (Fail-Safe Design)

### 2.1 Bahaya Mematikan dari `noDataState: OK`
Bayangkan Anda memiliki alert: `ErrorRate > 5%`. Suatu malam, agen pengumpul metrik (Alloy) mati atau jaringan ke Mimir terputus sehingga kueri tidak mengembalikan angka sama sekali (**`NoData`**).
- Jika Anda menyetel **`noDataState: OK`**, maka ketika sistem monitoring lumpuh atau aplikasi mati total tanpa trafik, **alert yang tadinya sedang `Firing` (Merah) akan otomatis berubah menjadi `Resolved / OK` (Hijau)!** Engineer on-call akan mengira masalah sudah sembuh dan kembali tidur, padahal server sedang mati total!

### 2.2 Tabel Kebijakan `noDataState` & `execErrState` yang Benar

| Kondisi | Opsi Konfigurasi di Grafana | Perilaku Sistem | Rekomendasi Standar SRE |
|---|---|---|---|
| **Saat Kueri Kosong (`noDataState`)** | `NoData` / `Alerting` | Mengirim peringatan khusus bahwa aliran metrik terputus (`DatasourceNoData`). | **WAJIB gunakan `Alerting` atau `NoData` (yang dirutekan ke tim pemilik)** untuk layanan kritis produksi! |
| **Saat Kueri Kosong (`noDataState`)** | `KeepLast` | Mempertahankan status terakhir sebelum data hilang. | Cocok untuk metrik yang jarang muncul agar status `Firing` tidak hilang saat metrik terputus. |
| **Saat Datasource Timeout/Error (`execErrState`)** | `Error` / `Alerting` | Mengirim peringatan bahwa kueri gagal dieksekusi (misal Mimir HTTP 500/Timeout). | **WAJIB gunakan `Alerting` atau `Error`** agar kegagalan sistem monitoring langsung terdeteksi! |

---

## 3. Provisioning Alerting, Contact Points, & Routing as Code (Secret-Safe)

Sama seperti dashboard, konfigurasi Alerting, Contact Points, dan Notification Policies di Grafana wajib dikelola sebagai kode YAML di repositori GitOps (**tanpa menuliskan token/webhook URL rahasia secara plaintext**).

Berikut adalah contoh file provisioning `/etc/grafana/provisioning/alerting/alerting-prod.yaml`:

```yaml
apiVersion: 1

# ==============================================================================
# 1. CONTACT POINTS (Saluran Pengiriman Notifikasi — SECRET-SAFE!)
# ==============================================================================
contactPoints:
  - orgId: 1
    name: "sre-oncall-critical-webhook"
    receivers:
      - uid: "oncall-webhook-01"
        type: "webhook"
        settings:
          # Variabel disuntikkan dari K8s Secret milik pod Grafana!
          url: "${ONCALL_CRITICAL_WEBHOOK_URL}"
          httpMethod: "POST"
        disableResolveMessage: false

  - orgId: 1
    name: "sre-platform-warning-channel"
    receivers:
      - uid: "platform-webhook-01"
        type: "webhook"
        settings:
          url: "${PLATFORM_WARNING_WEBHOOK_URL}"
          httpMethod: "POST"
        disableResolveMessage: false

# ==============================================================================
# 2. NOTIFICATION POLICIES (Pohon Perutean & Pengelompokan)
# ==============================================================================
policies:
  - orgId: 1
    receiver: "sre-platform-warning-channel"
    group_by:
      - "grafana_folder"
      - "alertname"
      - "sre_environment"
      - "sre_owner"
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 4h
    routes:
      # Rute Prioritas 1: Alert Critical di Production langsung ke On-Call Pager
      - receiver: "sre-oncall-critical-webhook"
        object_matchers:
          - ["severity", "=", "critical"]
          - ["sre_environment", "=", "production"]
        group_wait: 10s
        repeat_interval: 1h
        continue: false

      # Rute Prioritas 2: Tangani kondisi Datasource NoData / Error secara eksplisit!
      - receiver: "sre-platform-warning-channel"
        object_matchers:
          - ["alertname", =~, "Datasource(NoData|Error)"]
        group_wait: 1m
        repeat_interval: 2h

# ==============================================================================
# 3. MUTE TIMINGS (Jendela Pemeliharaan Terjadwal Non-Produksi)
# ==============================================================================
muteTimes:
  - orgId: 1
    name: "staging-nightly-maintenance-window"
    time_intervals:
      - times:
          - start_time: "01:00"
            end_time: "03:00"
        weekdays: ["monday:friday"]
        location: "Asia/Jakarta"
```

---

## 4. Tata Kelola Silences, Maintenance Window, dan Meta-Monitoring

1. **Aturan Ketat Pembuatan Silence:**
   - Setiap Silence yang dibuat secara manual saat terjadi pemeliharaan darurat **wajib** memiliki:
     - **Author:** Email engineer yang bertanggung jawab.
     - **Comment / Ticket Reference:** Nomor tiket perubahan atau insiden (misal `CHG-2026-881: Upgrade Kernel Node Worker-02`).
     - **Bounded Matchers:** Spesifik pada `instance` atau `service` yang sedang dikerjakan (DILARANG membisukan `severity="critical"` untuk seluruh cluster!).
     - **Mandatory Expiry:** Batas waktu kedaluwarsa yang jelas (misal maksimal `2h` atau `4h`).
2. **Meta-Monitoring ("Siapa yang Memantau Sang Pemantau?"):**
   - Pasang mekanisme **Dead Man's Switch (Heartbeat Alert)**: sebuah rule bernama `Watchdog` dengan ekspresi `vector(1)` yang selalu berstatus `Firing` dan dikirim ke sistem pemantau eksternal setiap 5 menit. Jika sinyal detak jantung ini berhenti datang, sistem eksternal langsung membunyikan sirine bahwa jalur Alertmanager/Grafana Anda sedang mati!

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Titik Putus di Rantai Sinyal | Langkah Investigasi & Solusi SRE |
|---|---|---|
| **Metrik Melampaui Batas di Grafik, tapi Alert Tetap `Normal` (Tidak `Firing`)** | **Tahap 1–2 (Evaluator):**<br>1. Klausul `for: 10m` belum tercapai.<br>2. Filter label pada kueri alert berbeda dengan kueri di panel dashboard.<br>3. Ingestion delay menyebabkan sampel terbaru belum masuk saat dievaluasi. | Klik tombol **"Test Rule"** atau periksa riwayat state di halaman Alerting UI untuk melihat nilai numerik aktual yang dibaca oleh evaluator. |
| **Alert Berstatus `Firing`, tapi Tidak Ada Pesan Masuk ke Slack/Webhook** | **Tahap 3–4 (Router / Contact Point):**<br>1. Alert terkena **Silence** atau **Mute Timing** aktif.<br>2. Terkena **Inhibition** oleh alert lain.<br>3. Kredensial Webhook URL salah / diblokir firewall keluar. | 1. Cek tab **Silences** dan **Notification Policies**.<br>2. Periksa metrik kegagalan pengiriman: `grafana_alerting_result_total` dan `alertmanager_notifications_failed_total`.<br>3. Gunakan tombol **"Test"** pada Contact Point dengan penerima uji (*disposable receiver*). |
| **Engineer Menerima Ratusan Pesan Terpisah untuk 1 Masalah yang Sama** | **Tahap 3 (Grouping):**<br>Atribut `group_by` menyertakan label unik seperti `pod_name` atau `instance` pada layanan dengan puluhan replika. | Ubah `group_by` menjadi `['alertname', 'sre_environment', 'service']` agar kegagalan 20 pod dari 1 service diringkas menjadi 1 pesan notifikasi tunggal. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. MEMERIKSA STATUS EVALUASI ALERT & PENGIRIMAN NOTIFIKASI VIA API
# ==============================================================================

# Melihat seluruh alert rules yang sedang dievaluasi beserta state-nya (Normal/Pending/Firing)
curl -s http://localhost:3000/api/prometheus/grafana/api/v1/rules | \
  jq '.data.groups[].rules[] | {name: .name, state: .state, health: .health}'

# Memeriksa daftar alerts yang sedang aktif di Alertmanager internal Grafana
curl -s http://localhost:3000/api/alertmanager/grafana/api/v2/alerts | \
  jq '.[] | {labels: .labels, status: .status.state, startsAt: .startsAt}'

# ==============================================================================
# 2. AUDIT METRIK META-MONITORING ALERTING (PROMETHEUS FORMAT)
# ==============================================================================

# Memeriksa apakah ada notifikasi Contact Point yang gagal terkirim (Harus 0!)
curl -s http://localhost:3000/metrics | grep "alertmanager_notifications_failed_total"

# Memeriksa apakah ada rule yang mengalami error saat dievaluasi
curl -s http://localhost:3000/metrics | grep "grafana_alerting_rule_evaluations_failures_total"
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa menyetel kebijakan **`noDataState: OK`** pada sebuah Alert Rule produksi yang memantau kesehatan layanan kritis merupakan kesalahan fatal (*anti-pattern*) dalam prinsip SRE?
- A. Karena opsi `OK` membutuhkan memori RAM 10 kali lebih besar.
- B. Karena jika terjadi pemadaman total pada agen pengumpul telemetri (Alloy) atau terputusnya jaringan ke database metrik (Mimir) di tengah insiden, kueri yang menghasilkan `NoData` akan otomatis dianggap **"Sehat / Resolved"**, sehingga menutupi bencana yang sedang terjadi (*silent failure*).
- C. Karena Grafana tidak mengizinkan penggunaan huruf kapital pada konfigurasi YAML.
- D. Karena `noDataState: OK` akan menghapus seluruh dashboard di dalam folder produksi.

#### Soal 2
Manakah urutan rantai bukti (*evidence chain*) yang benar untuk membuktikan bahwa sebuah skenario peringatan darurat benar-benar siap digunakan di lingkungan produksi?
- A. Cukup memastikan file YAML alert rule tidak memiliki error garis merah di teks editor.
- B. Cukup memastikan status pod Grafana `Running`.
- C. Membuktikan secara berurutan bahwa: (1) Evaluator mendeteksi pelanggaran ambang batas dan beralih ke `Firing`, (2) Router mencocokkan label ke rute kebijakan yang tepat, (3) Contact Point berhasil mengirim HTTP payload ke saluran penerima (`200 Accepted`), dan (4) Pesan notifikasi memuat tautan `runbook_url` yang dapat ditindaklanjuti oleh operator.
- D. Memastikan tidak ada satu pun alert yang pernah berbunyi selama 5 tahun.

#### Soal 3
Bagaimana cara yang aman (*secret-safe*) untuk mendefinisikan URL Webhook Slack/Telegram yang mengandung token autentikasi di dalam file provisioning GitOps Grafana Alerting?
- A. Menuliskan URL lengkap beserta token rahasianya secara langsung di dalam file `alerting.yaml` lalu mem-push-nya ke repositori Git publik.
- B. Menyimpan URL rahasia tersebut di dalam Kubernetes Secret terenkripsi (misal via SealedSecret/Vault), menyuntikkannya sebagai variabel environment ke dalam pod Grafana, lalu mereferensikannya dengan sintaks `${WEBHOOK_URL_SECRET}` di file YAML provisioning.
- C. Menuliskan token tersebut di judul dashboard agar mudah disalin oleh semua anggota tim.
- D. Mengirimkan password melalui komentar Merge Request.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Pada hari Sabtu pukul 02:00 dini hari, tim infrastruktur berencana melakukan *reboot* terjadwal pada 1 server fisik (`worker-node-04`) selama 30 menit untuk pembaruan firmware BIOS.
Seorang engineer junior tidak ingin terganggu oleh bunyi alarm selama proses *reboot*. Ia masuk ke menu **Silences** di Alertmanager/Grafana, lalu membuat Silence dengan konfigurasi berikut:
- **Matcher:** `sre_environment = "production"`
- **Duration:** `72 hours (3 hari)`
- **Comment:** `"test"`

Sepuluh menit kemudian, pada pukul 02:10, layanan `payment-api` di node lain (`worker-node-01` dan `worker-node-02`) mengalami kegagalan koneksi database total (Error 500 mencapai 100%). Namun, **tidak ada satu pun notifikasi yang berbunyi di ponsel tim On-Call** hingga pagi hari!

**Pertanyaan & Tugas Anda:**
1. Mengapa tidak ada satu pun alert untuk `payment-api` yang terkirim ke tim On-Call pada pukul 02:10 tersebut?
2. Sebutkan **3 kesalahan fatal** dalam cara engineer junior tersebut membuat Silence!
3. Tuliskan spesifikasi Silence yang benar dan aman untuk skenario pemeliharaan `worker-node-04` selama 30 menit tersebut!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Dalam rekayasa keandalan, hilangnya sinyal pemantauan (*missing telemetry*) adalah kondisi abnormal yang wajib diinvestigasi (`Alerting` / `NoData`), bukan dianggap normal (`OK`).
2. **C.** Setiap tahapan (Evaluator -> Router -> Contact Point -> Operator) memiliki domain kegagalannya masing-masing. Status `Firing` di layar UI tidak menjamin bahwa pesan berhasil menembus firewall keluar menuju ponsel petugas piket.
3. **B.** Ekspansi variabel environment (`${VAR}`) dari Kubernetes Secret menjaga agar repositori Git tetap bersih dari rahasia sekaligus mendukung rotasi kredensial yang aman.

#### Solusi Soal Kasus Nyata:
1. **Penyebab Hilangnya Seluruh Notifikasi:**
   - Engineer junior membuat Silence dengan matcher yang terlalu luas: `sre_environment = "production"` tanpa membatasi `instance` atau `alertname`. Akibatnya, **seluruh alert dari semua layanan dan semua server di environment production dibungkam secara total!**
2. **Tiga Kesalahan Fatal Engineer Junior:**
   - **Scope Matcher Terlalu Luas (Unbounded Scope):** Membisukan seluruh environment `production` alih-alih hanya membisukan `instance="worker-node-04:9100"`.
   - **Durasi Kedaluwarsa Berlebihan:** Pekerjaan hanya butuh 30 menit, tetapi Silence dipasang selama 72 jam (3 hari) dan lupa dicabut setelah selesai.
   - **Tidak Ada Konteks Audit:** Komentar hanya berisi `"test"` tanpa mencantumkan nomor tiket perubahan resmi dan nama penanggung jawab.
3. **Spesifikasi Silence yang Benar & Aman:**
   - **Matchers:** `instance = "192.168.50.14:9100"` (atau `node = "worker-node-04"`) DAN `sre_environment = "production"`.
   - **Duration:** `45m` (30 menit waktu kerja + 15 menit buffer rekonsiliasi).
   - **Author:** `junior-sre@example.com`
   - **Comment:** `MAINT-2026-109: Reboot terjadwal BIOS worker-node-04 (Disetujui oleh Lead SRE)`.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai perancangan Alert Rule, Contact Point, dan kebijakan perutean notifikasi:
- Praktikkan di [LAB-02 — Alert Firing, Notification, dan Failure Injection](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.5-grafana-alerting-slo/lab/LAB-02-alert-firing-notification-failure-injection.md).
- Lanjutkan ke materi puncak dari Fase 7: [03 — SLO, Error Budget, Burn Rate, dan Runbook](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.5-grafana-alerting-slo/03-slo-error-budget-burn-rate-runbook.md) untuk menerjemahkan metrik teknis menjadi bahasa keandalan bisnis (**SLI, SLO, Error Budget, dan Multi-Window Burn-Rate Alerts**).
