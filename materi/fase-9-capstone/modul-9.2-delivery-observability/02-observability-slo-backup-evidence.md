# 9.2.2 — Telemetry, SLO, Alert, dan Recovery Evidence

> **Ringkasan:** Mengintegrasikan seluruh pipa observabilitas (**Grafana Alloy, Mimir, Loki, Tempo, dan Alertmanager**) di lingkungan bare-metal on-premises, menegakkan kontrak **5 Peringatan Kritis (*The 5 Mandatory Alert Contracts*)**, memvalidasi perhitungan **SLO dan Error Budget**, serta menyusun paket bukti digital (**Evidence Packet**) dari hasil uji pemulihan pencadangan (*Isolated Backup & Restore Verification*).

---

## 1. Alur Telemetri Terpadu & Standar Kardinalitas

### 1.1 Analogi Dunia Nyata: "Pusat Radar Lalu Lintas Udara & Brankas Salinan Rekaman Penerbangan"
Bayangkan sebuah menara pengawas bandara internasional (Air Traffic Control):
- **Instrumen Radar Terpadu (Observabilitas 3 Pilar):** Petugas menara tidak hanya mengandalkan suara radio pilot (Log). Mereka melihat radar ketinggian dan kecepatan secara *real-time* (Metrik Mimir), mendengar rekaman percakapan kokpit (Log Loki), serta melacak rute penerbangan 3 dimensi dari bandara asal hingga bandara tujuan (Jejak Distributed Tracing Tempo). Ketiga instrumen ini saling terhubung; jika radar menunjukkan kecepatan menurun drastis, petugas langsung melompat ke rekaman suara detik yang sama.
- **Alarm Darurat Otomatis (*Alertmanager*):** Jika dua pesawat mendekat di ketinggian yang sama, sirene menara langsung berbunyi keras (*Paging*) dan menampilkan panduan prosedur darurat (*Runbook*) di layar.
- **Brankas Salinan Kotak Hitam Luar Kota (*Isolated Recovery*):** Data rekaman penerbangan tidak hanya disimpan di hard disk lokal menara. Salinan terenkripsi dikirim setiap jam ke brankas tahan api di luar kota (**MinIO S3 On-Prem**). Setiap bulan, teknisi wajib melakukan simulasi: mengambil salinan rekaman tersebut dan membukanya di komputer cadangan terisolasi untuk membuktikan bahwa file tersebut benar-benar bisa dibaca dan tidak rusak!

---

### 1.2 Diagram Arsitektur Telemetri & Jalur Peringatan

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ARSITEKTUR LENGKAP PIPA OBSERVABILITAS CAPSTONE             │
│                                                                             │
│  [ Workload Pods ]        [ Node Exporter ]       [ k3s / MetalLB ]         │
│  (:8080/metrics)          (:9100/metrics)         (Kubelet / ARP events)    │
│         │                         │                        │                │
│         └─────────────────────────┼────────────────────────┘                │
│                                   │                                         │
│                                   ▼ (Scrape & Collect via gRPC/HTTP)        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    GRAFANA ALLOY (DAEMONSET AGENT)                    │  │
│  │  - Memory Limiter: Cegah OOMKilled                                    │  │
│  │  - Redaction: Sensor Password, JWT Token, Authorization Header & PII  │  │
│  │  - Bounded Relabeling: Buang High-Cardinality Labels                  │  │
│  └───────┬────────────────────────┼───────────────────────┬──────────────┘  │
│          │ Remote-Write           │ Log Push              │ OTLP Trace Push │
│          ▼                        ▼                       ▼                 │
│  ┌───────────────┐        ┌───────────────┐       ┌───────────────┐         │
│  │ GRAFANA MIMIR │        │ GRAFANA LOKI  │       │ GRAFANA TEMPO │         │
│  │ (Metrics TSDB)│        │ (Logs Chunks) │       │ (Traces S3)   │         │
│  └───────┬───────┘        └───────┬───────┘       └───────┬───────┘         │
│          │                        │                       │                 │
│          └────────────────────────┼───────────────────────┘                 │
│                                   │                                         │
│                                   ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │              GRAFANA DASHBOARDS (USE / RED / SLO VIEWS)               │  │
│  │  - Exemplars menghubungkan grafik latensi Mimir langsung ke Tempo!   │  │
│  │  - Derived Fields menghubungkan TraceID di Loki langsung ke Tempo!    │  │
│  └────────────────────────────────┬──────────────────────────────────────┘  │
│                                   │                                         │
│                                   ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │               MIMIR RULER & GRAFANA UNIFIED ALERTING                  │  │
│  │                 (Evaluasi Kontrak 5 Alert Wajib)                      │  │
│  └────────────────────────────────┬──────────────────────────────────────┘  │
│                                   │                                         │
│                                   ▼ (HTTP Post Webhook / Pager)             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │          ALERTMANAGER ──> ON-CALL SRE PAGER ──> RUNBOOK ACTION        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Kontrak 5 Peringatan Kritis (The 5 Mandatory Alert Contracts)

Dalam proyek Capstone, platform wajib membuktikan berfungsinya **5 Skenario Alert Kritis** yang memantau kesehatan seluruh ekosistem:

| No | Nama Alert Resmi | Target Pemilik (*Owner*) | Tingkat Keparahan | Sinyal Metrik & Ambang Batas | Tautan Dokumen Runbook Resmi |
|---|---|---|---|---|---|
| **1** | `HighErrorRate` | `@app-payment-owner` | **Critical (Page)** | Rasio HTTP 5xx pada layanan `payment-api` melebihi toleransi SLO (> 1%) selama 5 menit berturut-turut. | `runbooks/app-error.md` |
| **2** | `HighLatencyP95` | `@app-payment-owner` | **Critical (Page)** | Latensi persentil ke-95 (`p95`) melebihi batas waktu 500ms selama 10 menit. | `runbooks/latency.md` |
| **3** | `NodeDiskPressure`| `@platform-infra` | **Warning / Page** | Kapasitas hard disk partisi Linux atau Inode terpakai > 85% pada worker node. | `runbooks/disk-full.md` |
| **4** | `KubeNodeNotReady`| `@platform-infra` | **Critical (Page)** | Node Kubernetes berstatus `NotReady` atau daemon kubelet mati selama > 3 menit. | `runbooks/worker-down.md` |
| **5** | `MetalLBAddressPoolExhausted` | `@network-platform` | **Critical (Page)** | Sisa IP LoadBalancer yang tersedia di pool MetalLB kurang dari 2 alamat IP. | `runbooks/metallb-pool.md` |

### 2.1 Standar Integritas Kontrak Alert
Setiap definisi alert di atas **WAJIB** memiliki:
- **`runbook_url`:** Tautan aktif menuju dokumen prosedur penanganan darurat.
- **`no_data_policy: "alert_and_investigate"`:** Dilarang menganggap hilangnya data telemetri sebagai kondisi normal!
- **`notification_receipt`:** Bukti bahwa pesan notifikasi benar-benar sampai ke webhook penerima (*HTTP 200 Accepted*), bukan sekadar klaim status `Firing` di layar browser.

---

## 3. Tiga Lapisan Pencadangan & Protokol Uji Pemulihan Terisolasi

### 3.1 Tiga Lapisan Pencadangan Capstone
1. **Lapisan 1: Metadata & Objek Kubernetes (Velero):** Seluruh manifes namespace, deployment, configmap, dan secret terenkripsi disimpan ke bucket `minio://velero-backups`.
2. **Lapisan 2: Volume Data Persisten (Persistent Volumes):** Data database PostgreSQL dan file upload disalin menggunakan snapshot Velero Kopia/Restic dengan status transaksi konsisten (*quiesced*).
3. **Lapisan 3: Klaster State (k3s etcd Snapshot):** File snapshot database internal k3s control-plane disimpan secara berkala menggunakan perintah `k3s etcd-snapshot`.

---

### 3.2 Alur Uji Pemulihan Terisolasi (Isolated Restore Order)

Uji coba pemulihan **DILARANG KERAS** dieksekusi langsung menimpa klaster produksi yang sedang melayani transaksi! SRE membuktikan pemulihan pada **Klaster Uji Terisolasi (Disposable Target)**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR EKSEKUSI UJI PEMULIHAN TERISOLASI                      │
│                                                                             │
│  [1. Persiapan Target Disposable]                                           │
│   Siapkan klaster k3d baru atau namespace sandbox terisolasi:               │
│   `k3d cluster create restore-drill --agents 2`                             │
│                                  │                                          │
│                                  ▼                                          │
│  [2. Pemulihan Objek & Manifes Kubernetes]                                  │
│   `velero restore create --from-backup prod-backup-daily-01`                │
│                                  │                                          │
│                                  ▼                                          │
│  [3. Pemulihan Volume Data Persisten (PV)]                                  │
│   Pastikan volume disk ter-binding dengan benar dan UID database cocok.     │
│                                  │                                          │
│                                  ▼                                          │
│  [4. Verifikasi Integritas Transaksi Database]                              │
│   Jalankan query validasi integritas: `SELECT count(*) FROM orders;`        │
│                                  │                                          │
│                                  ▼                                          │
│  [5. Pengukuran RPO dan RTO Aktual]                                         │
│   Hitung selisih transaksi yang hilang (RPO) dan durasi pemulihan (RTO).    │
│                                  │                                          │
│                                  ▼                                          │
│  [6. Pembersihan & Penguncian Bukti (Cleanup & Evidence Sign-off)]          │
│   Hancurkan klaster sandbox disposable dan rekam dokumen bukti audit!       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Skema Dokumen Rekaman Bukti Observabilitas & Pemulihan (`telemetry-recovery-evidence.yaml`)

Dokumen ini adalah bukti digital wajib yang diserahkan pada dewan penguji Capstone:

```yaml
schema_version: "sre.bootcamp.capstone/telemetry-recovery-evidence/v1.0"
evidence_id: "EVID-20261018-1030"
recorded_at_utc: "2026-10-18T10:30:00Z"
evaluator_role: "Lead SRE Auditor"

# 1. Verifikasi Jendela Telemetri & Integrasi Tiga Pilar
telemetry_verification:
  observation_window_utc: "2026-10-18T09:30:00Z - 2026-10-18T10:30:00Z"
  metrics_source: "http://mimir-query-frontend.monitoring.svc.cluster.local:8080/prometheus"
  logs_source: "http://loki-gateway.monitoring.svc.cluster.local:80"
  traces_source: "http://tempo-query-frontend.monitoring.svc.cluster.local:3200"
  cross_signal_correlation_proven: true # Exemplars & Trace-to-Logs terbukti bekerja!

# 2. Rekaman Eksekusi Peringatan Darurat (The 5 Mandatory Alerts)
mandatory_alerts_status:
  - alert_name: "HighErrorRate"
    state: "VERIFIED_ACTIVE"
    test_firing_timestamp_utc: "2026-10-18T09:45:12Z"
    notification_channel: "webhook-oncall-pager"
    delivery_status_code: 200
    runbook_referenced: "runbooks/app-error.md"
  - alert_name: "HighLatencyP95"
    state: "VERIFIED_ACTIVE"
    delivery_status_code: 200
  - alert_name: "NodeDiskPressure"
    state: "VERIFIED_ACTIVE"
    delivery_status_code: 200
  - alert_name: "KubeNodeNotReady"
    state: "VERIFIED_ACTIVE"
    delivery_status_code: 200
  - alert_name: "MetalLBAddressPoolExhausted"
    state: "VERIFIED_ACTIVE"
    delivery_status_code: 200

# 3. Hasil Pengukuran SLO & Error Budget
slo_compliance_evaluation:
  target_service: "payment-api"
  monthly_objective_percentage: 99.9
  measured_30d_availability_percentage: 99.94
  remaining_error_budget_percentage: 40.0 # Masih di Zona Hijau
  burn_rate_at_eval: 0.12 # Sangat Sehat

# 4. Bukti Hasil Uji Pemulihan Bencana Terisolasi (Restore Drill Outcome)
disaster_recovery_drill:
  backup_identifier: "prod-snapshot-20261018-0800"
  backup_checksum_sha256: "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
  isolated_target: "k3d-disposable-restore-drill"
  measured_rpo_minutes: 12 # Transaksi hilang maksimal 12 menit
  measured_rto_minutes: 24 # Sistem pulih dan aktif dalam 24 menit
  database_data_integrity_test: "PASSED (All ledger transactions matched checksum)"
  drill_decision: "VERIFIED_OPERATIONAL"
```

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Pesan Notifikasi Alert Tidak Pernah Masuk ke Ponsel On-Call** | 1. URL Webhook salah atau diblokir NetworkPolicy.<br>2. Alert tertahan oleh `group_wait` yang terlalu lama.<br>3. Terkena *Silence* aktif yang lupa dicabut. | Cek tab **Silences** di Alertmanager. Periksa log Alertmanager: `kubectl logs -n monitoring -l app.kubernetes.io/name=alertmanager` untuk melihat error status HTTP pengiriman. |
| **Restore Velero Gagal dengan Pesan `VolumeSnapshotTimeout`** | CSI Driver storage lokal belum mendukung fitur snapshot native atau resource disk habis. | Gunakan mode backup berbasis file (**Velero Kopia/Restic**) yang mengunggah blok data langsung ke bucket MinIO S3 tanpa bergantung pada CSI storage plugin. |
| **Tombol Trace Exemplars di Grafana Diklik tapi Muncul `Trace Not Found`** | Tempo menghapus trace lebih cepat daripada Mimir karena durasi retensi Tempo disetel terlalu singkat (misal hanya 1 jam), atau jam antar node (*clock skew*) meleset. | Sinkronkan waktu seluruh node dengan `chronyd` dan pastikan retensi Tempo minimal 3–7 hari (`block_retention: 168h`). |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. MEMERIKSA STATUS EVALUASI RULES & ALERTS DI MIMIR / PROMETHEUS
# ==============================================================================

# Menampilkan seluruh aturan peringatan yang sedang aktif dievaluasi
curl -s http://localhost:8080/prometheus/api/v1/rules | \
  jq '.data.groups[].rules[] | select(.type=="alerting") | {alert: .name, state: .state}'

# Menguji pengiriman notifikasi test ke Alertmanager
amtool alert add alertname="TestAlertManual" severity="critical" instance="test-node" \
  --alertmanager.url=http://localhost:9093

# ==============================================================================
# 2. OPERASI PENCADANGAN & PEMULIHAN DENGAN VELERO
# ==============================================================================

# Mengambil status detail proses restore Velero terakhir
velero restore get
velero restore describe <RESTORE_NAME> --details

# Memeriksa log proses backup untuk memastikan tidak ada file database yang terlewati
velero backup logs <BACKUP_NAME> | grep -iE "(error|warning|completed)"
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa seorang auditor SRE menolak klaim kesiapan sistem jika kandidat hanya menunjukkan bahwa alert di Prometheus berstatus **`Firing`**, tanpa menyertakan bukti **Notification Receipt** dari Alertmanager?
- A. Karena warna merah pada status `Firing` dianggap kurang kontras.
- B. Karena status `Firing` hanya membuktikan bahwa kueri metrik melewati ambang batas di dalam evaluator; status tersebut **belum membuktikan** bahwa pesan peringatan berhasil melewati pohon perutean (*routing tree*), tidak terhalang oleh *Silence* atau *Inhibition*, serta berhasil dikirimkan menembus firewall ke ponsel petugas piket (*Contact Point HTTP 200*).
- C. Karena Prometheus tidak memiliki fitur pengiriman notifikasi sama sekali.
- D. Agar engineer terpaksa membeli lisensi software berbayar.

#### Soal 2
Dalam perancangan sistem pemantauan keandalan layanan bisnis, apa perbedaan peran mendasar antara **Alert Kritis (*Critical Paging Alert*)** dengan **Dashboard Visualisasi Grafana**?
- A. Dashboard Grafana berbunyi keras di malam hari, sedangkan alert hanya menampilkan gambar.
- B. Alert bertindak secara proaktif untuk membangunkan manusia hanya saat terjadi ancaman nyata terhadap SLO bisnis (*Actionable & Symptom-based*), sedangkan Dashboard bertindak secara reaktif sebagai instrumen penyelidikan visual (*Triage & Root Cause Investigation*) saat manusia sudah berada di depan layar.
- C. Dashboard Grafana tidak bisa membaca data dari database Mimir.
- D. Alert hanya boleh dibuat oleh manajer perusahaan.

#### Soal 3
Saat melakukan simulasi pemulihan bencana (*Disaster Recovery Drill*), mengapa pengujian pemulihan data dari backup Velero/etcd **WAJIB** dieksekusi pada lingkungan terisolasi (*Isolated Disposable Target*)?
- A. Karena Velero otomatis menghapus database sumber saat proses restore dimulai.
- B. Untuk mencegah terjadinya tabrakan jaringan (*IP conflict*), penimpaan data transaksi yang sedang aktif berjalan di produksi, serta memastikan pengujian berlangsung objektif dari kondisi nol (*Cold-Start*).
- C. Karena sistem operasi Mac melarang pembukaan dua tab terminal sekaligus.
- D. Agar kuota penyimpanan hard disk bertambah dua kali lipat secara otomatis.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Dalam sesi verifikasi Capstone, seorang kandidat membanggakan sistem pencadangan yang dibuatnya:
*"Saya sudah mengonfigurasi cron backup otomatis setiap malam pukul 00:00 menggunakan Velero ke MinIO. Status backup setiap hari selalu `Completed` 100%. RPO kami adalah 0 menit dan RTO kami adalah 5 menit!"*

Dewan penguji meminta kandidat untuk membuktikannya secara langsung:
*"Bagus. Sekarang bayangkan server produksi kita terserang ransomware pada pukul 15:30 sore ini. Tolong pulihkan data transaksi database nasabah ke klaster sandbox ini sekarang juga dan tunjukkan kepada kami hasilnya!"*

Kandidat mulai panik dan membutuhkan waktu 45 menit untuk mencari file dokumentasi restore. Ketika restore selesai dijalankan, database PostgreSQL menolak menyala karena file kunci enkripsi (`encryption_key`) tidak ikut tersimpan di dalam backup Velero. Selain itu, seluruh data transaksi antara pukul 00:01 hingga 15:30 sore itu hilang total.

**Pertanyaan & Tugas Anda sebagai Lead SRE Evaluator:**
1. Mengapa klaim kandidat bahwa **"RPO = 0 menit"** adalah kebohongan teknis yang fatal berdasarkan jadwal backup harian pukul 00:00 tersebut? Berapa nilai **RPO aktual** pada insiden pukul 15:30 sore itu?
2. Mengapa klaim **"RTO = 5 menit"** terbukti tidak valid dalam pengujian nyata di atas?
3. Sebutkan **2 kesalahan fatal dalam strategi pencadangan kandidat** (terkait kunci enkripsi dan uji coba restore) dan bagaimana cara memperbaikinya!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** *Alerting delivery chain* memiliki banyak titik kegagalan di hilir (router, silence, webhook network). Status `Firing` di hulu tidak menjamin sinyal sampai ke telinga manusia.
2. **B.** SRE memegang prinsip: *Alerts wake people up; Dashboards help them fix the problem*. Jangan pernah memaksa engineer menatap dashboard 24 jam sehari untuk mencari anomali secara manual.
3. **B.** Menjalankan restore ke klaster aktif tanpa isolasi dapat merusak data transaksi yang sedang berjalan dan memicu konflik alokasi Service IP di jaringan produksi.

#### Solusi Soal Kasus Nyata:
1. **Analisis RPO Aktual:**
   - **RPO (Recovery Point Objective)** mengukur jarak waktu maksimal transaksi yang hilang sejak titik backup terakhir.
   - Karena backup terakhir diambil pukul 00:00 dan insiden terjadi pukul 15:30, maka seluruh transaksi nasabah selama **15 jam 30 menit HILANG TOTAL!**
   - Klaim "RPO = 0 menit" hanya mungkin tercapai jika menggunakan replikasi sinkronus (*synchronous multi-datacenter replication*) atau pengarsipan stream WAL secara *real-time*. Nilai RPO aktual kandidat adalah **15,5 Jam**.
2. **Analisis RTO Aktual:**
   - **RTO (Recovery Time Objective)** dihitung dari saat insiden dideklarasikan hingga aplikasi kembali melayani transaksi secara sehat.
   - Kandidat membutuhkan 45 menit hanya untuk mencari dokumen, dan proses restore akhirnya gagal total (*system never recovered*). Klaim RTO 5 menit hanyalah asumsi tanpa dasar bukti empiris.
3. **Dua Kesalahan Fatal & Perbaikannya:**
   - **Kesalahan 1 (Kunci Enkripsi Terpisah & Hilang):** Menyimpan database terenkripsi tanpa menyertakan prosedur pemulihan kunci enkripsi (*key custody boundary*) membuat data yang di-restore menjadi tumpukan biner yang tidak bisa dibaca.
     *Perbaikan:* Kelola kunci enkripsi secara terpusat di HashiCorp Vault / KMS eksternal dengan prosedur *Break-Glass Key Recovery* tertulis.
   - **Kesalahan 2 (Tidak Pernah Melakukan Restore Drill):** Kandidat menganggap status `Completed` sama dengan kesiapan pemulihan (*untested backup is no backup*).
     *Perbaikan:* Jadwalkan latihan pemulihan berkala (*Disaster Recovery Drill*) secara otomatis setiap minggu ke klaster disposable untuk memvalidasi integritas data secara berkelanjutan.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai validasi telemetri penuh, kontrak 5 alert kritis, dan pembuktian pemulihan bencana:
- Lanjutkan ke modul puncak kelulusan: [Modul 9.3 — Game Day & Graduation Readiness Review](../modul-9.3-game-day-graduation/README.md) untuk menghadapi simulasi insiden nyata tanpa pemberitahuan (*Chaos Injection*) dan mempertahankan dokumen Postmortem di hadapan dewan penguji!
- Praktikkan simulasi di [LAB-02 — Telemetry, Alert, dan Backup Evidence Lab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-9-capstone/modul-9.2-delivery-observability/lab/LAB-02-telemetry-alert-backup-evidence.md).
