# 9.3.1 — Game Day dan Skenario On-Call

> **Ringkasan:** Memahami metodologi pelaksanaan simulasi krisis keandalan (**SRE Game Day**), mengorganisasi peran darurat menggunakan **Incident Command Card**, mengeksekusi penanganan terarah untuk **7 Skenario Kegagalan Sistem Nyata (*The 7 Chaos Scenarios*)**, serta menegakkan disiplin pemeriksaan awal tanpa mutasi (*Read-Only First Checks*) sebelum melakukan tindakan pemulihan.

---

## 1. Konsep & Filosofi Game Day

### 1.1 Analogi Dunia Nyata: "Simulator Penerbangan Cuaca Ekstrem & Simulasi Kebakaran Gedung"
Bayangkan bagaimana pilot pesawat komersial dilatih menghadapi bahaya:
- Pilot tidak menunggu sampai mesin terbakar di dunia nyata untuk pertama kalinya. Mereka masuk ke dalam **Simulator Penerbangan (Flight Simulator)**.
- Di dalam simulator, instruktur secara diam-diam menyuntikkan kegagalan (*fault injection*): mesin mati di tengah badai petir, sistem hidrolik bocor, atau instrumen altimeter membeku.
- Pilot diuji bukan untuk menghafal tombol, melainkan untuk **tetap tenang di bawah tekanan, mematuhi buku panduan darurat (*Quick Reference Handbook*), berkomunikasi secara terstruktur dengan menara pengawas, dan mengambil keputusan pendaratan darurat yang terukur**.

Dalam rekayasa keandalan SRE, **Game Day** adalah simulator penerbangan tersebut. Kita tidak menguji keandalan sistem dengan berharap sistem tidak pernah rusak. Kita secara sengaja menyuntikkan kegagalan nyata ke lingkungan klaster on-premises untuk membuktikan bahwa:
1. Peringatan darurat (*Alerting*) benar-benar berbunyi.
2. Tim on-call tahu Runbook mana yang harus dibuka.
3. Dampak insiden dapat dihentikan dalam hitungan menit sebelum menghabiskan Error Budget.

---

### 1.2 Struktur Kartu Komando Insiden (The Incident Command Card)

Setiap sesi Game Day menunjuk peran darurat yang jelas bagi para peserta:

```yaml
incident_command_card:
  drill_id: "GD-SIM-20261018-01"
  incident_commander: "Peserta A (Menjaga makro koordinasi & keputusan stop)"
  operations_lead: "Peserta B (Mengeksekusi diagnosa & mitigasi teknis)"
  communications_lead: "Peserta C (Mengirim update status berkala ke stakeholder)"
  scribe_log_keeper: "Peserta D (Mencatat lini masa waktu UTC dan bukti perintah)"
  primary_oncall: "Peserta B"
  secondary_oncall: "Peserta A"
  severity_level: "P1 (Critical Service Outage)"
  acknowledgement_sla: "< 5 Menit"
  time_zone: "UTC"
  rollback_authority: "Incident Commander memiliki hak mutlak pembatalan"
```

> [!IMPORTANT]
> **Tiga Aturan Emas Keselamatan Game Day (Golden Rules):**
> 1. **Hanya Satu Suntikan Kegagalan per Iterasi (*One Fault at a Time*):** Dilarang menyuntikkan 2 kegagalan sekaligus agar tim dapat mengisolasi penyebab dan efeknya secara ilmiah.
> 2. **Kondisi Berhenti Tegas (*Definite Stop Condition*):** Jika kegagalan yang disuntikkan meluas ke luar batas scope laboratorium (*unintended blast radius*) atau mengancam data yang belum dibackup, IC **WAJIB MENYATAKAN ABORT** seketika!
> 3. **Dilarang Melakukan Chaos di Produksi Tanpa Jaring Pengaman:** Seluruh simulasi Game Day Capstone dijalankan pada klaster disposable atau lingkungan simulasi target terisolasi di OrbStack.

---

## 2. Matriks 7 Skenario Kegagalan Game Day (The 7 Chaos Scenarios)

Berikut adalah 7 skenario kegagalan operasional yang diujikan dalam Game Day Capstone:

| ID Skenario | Skenario Suntikan Kegagalan (*Fault Injection*) | Sinyal Deteksi Awal (*First Signal*) | Tindakan Mitigasi Teruji (*Safe Mitigation*) | Batas Berhenti Darurat (*Stop Condition*) |
|---|---|---|---|---|
| **GD-01** | **Worker Node Mati Listrik Mendadak** (`orb stop servers-vm` atau matikan service k3s-agent pada satu worker). | Alert `KubeNodeNotReady`, Pod berpindah ke status `Pending`. | Cordon node yang mati, periksa sisa kapasitas RAM klaster (N+1 Headroom), tunggu penjadwalan ulang Pod. | Sisa node kehabisan memori (*MemoryPressure*) atau kuorum etcd terancam. |
| **GD-02** | **Partisi Hard Disk Penuh 100%** (Mengisi file dummy di `/var/log` menggunakan `fallocate`). | Alert `NodeDiskPressure`, Pod mengalami pengusiran massal (*Evicted*). | Jalankan pembersihan log lama via journalctl, prune container image yang tidak terpakai (`crictl rmi --prune`). | Disk yang penuh adalah direktori penyimpanan database transaksi tanpa backup! |
| **GD-03** | **Rilis Kode Aplikasi Cacat (*Bad Release*)** (Mempromosikan versi aplikasi yang memuat bug koneksi database). | Lonjakan tajam alert `HighErrorRate` (HTTP 5xx > 10%) dan latensi tinggi. | **Fast-Track Rollback (< 5 Menit):** Eksekusi `kubectl argo rollouts abort` dan `undo` ke revisi sehat sebelumnya! | Rilis baru telah menjalankan migrasi database yang merusak kolom data lama. |
| **GD-04** | **Sertifikat Ingress TLS Kedaluwarsa** (Menyuntikkan sertifikat dummy dengan tanggal expired ke Secret ingress). | Alert `CertificateExpiringSoon` dan curl mengembalikan error `x509: certificate has expired`. | Eksekusi prosedur pembaruan cert-manager atau terapkan sertifikat baru yang valid via GitOps. | Kunci privat (*Private Key*) CA tidak dapat diakses atau hilang. |
| **GD-05** | **Lonjakan Latensi 10 Kali Lipat** (Menyuntikkan latensi jaringan 2 detik menggunakan perintah `tc netem`). | Alert `HighLatencyP95`, trace di Tempo menunjukkan span database memanjang drastis. | Isolasi pod backend yang lambat, alihkan trafik ke replika cadangan, atau aktifkan caching layer darurat. | Beban antrean trafik meluber ke antrean memori hingga memicu OOMKilled massal. |
| **GD-06** | **Konflik Alamat IP MetalLB (ARP Conflict)** (Menetapkan IP VIP yang sama pada interface mesin lain). | Panggilan HTTP ke VIP gagal (`Connection Refused`), `arping` mendeteksi 2 MAC Address berbeda. | Pindahkan alokasi IP Service ke IP kosong lain di dalam `IPAddressPool`, kirimkan paket Gratuitous ARP. | Konflik IP menyentuh subnet manajemen jaringan fisik kantor. |
| **GD-07** | **Kehancuran Klaster Total (Total Disaster Recovery)** (Menghapus seluruh VM klaster k3s di OrbStack). | Seluruh endpoint layanan mati total (`0% Availability`). | **Cold-Start Rebuild:** Jalankan OpenTofu apply ➔ Ansible bootstrap ➔ ArgoCD rehydrate dari repositori Git! | Repositori Git atau remote state MinIO tidak dapat diakses. |

---

## 3. Alur Siklus Penanganan Insiden Game Day (Incident Lifecycle)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 SIKLUS 8 LANGKAH TANGGAP INSIDEN GAME DAY                   │
│                                                                             │
│  [1. DETECTION]     ──> Alarm Alertmanager berbunyi / Pengguna melapor      │
│  [2. DECLARATION]   ──> Deklarasikan Insiden P1 & Tunjuk Peran ICS          │
│  [3. READ-ONLY]     ──> Periksa Dashboard RED & Event Pod (Dilarang Mutasi!)│
│  [4. RUNBOOK SELECT]──> Buka Runbook yang cocok (misal: RB-APP-001)         │
│  [5. MITIGATION]    ──> Eksekusi tindakan aman (Rollback / Failover / Prune)│
│  [6. VERIFICATION]  ──> Pantau Error Rate & Latensi kembali normal > 15m    │
│  [7. CLOSURE]       ──> IC menutup status darurat & membubarkan War Room    │
│  [8. POSTMORTEM]    ──> Susun dokumen Blameless Postmortem dalam 24 jam!    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Protokol Pemeriksaan Awal Tanpa Mutasi (Read-Only First Checks)
Sebelum engineer diizinkan mengetik perintah perbaikan apa pun, mereka **WAJIB** membuktikan 3 fakta di War Room:
1. **Verifikasi Sasaran (*Target Verification*):** Apakah klaster dan namespace yang sedang dilihat benar-benar target yang bermasalah? (`kubectl config current-context`).
2. **Korelasi Waktu (*Timeline Correlation*):** Kapan lonjakan error pertama kali terjadi pada grafik Grafana Mimir? Apakah bertepatan dengan waktu sinkronisasi ArgoCD?
3. **Penyelidikan Trace & Event (*Deep Signal*):** Buka contoh trace yang gagal di Grafana Tempo atau periksa riwayat event klaster (`kubectl get events --sort-by=.metadata.creationTimestamp`).

---

## 4. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah di Game Day | Penyebab Utama | Langkah Penyelamatan SRE |
|---|---|---|
| **Target Rollback GD-03 Melebihi Batas Waktu 5 Menit** | Engineer terjebak dalam perdebatan membaca kode aplikasi dan mencoba memperbaiki bug di baris kode alih-alih langsung membatalkan deployment. | **Disiplin IC:** Ingatkan tim bahwa tujuan pertama adalah **MENGHENTIKAN DAMPAK KE PENGGUNA**. Segera eksekusi `kubectl argo rollouts abort` dan selidiki bug setelah sistem stabil! |
| **Peserta Menjalankan Perintah Berbahaya di Terminal yang Salah** | Tab terminal produksi dan terminal lab tidak memiliki pembeda visual, memicu eksekusi perintah destruktif di luar scope. | Terapkan proteksi prompt terminal (beri warna merah terang untuk produksi) dan batasi izin akun pengguna dengan RBAC terpisah. |
| **Skenario GD-06 (MetalLB) Macet karena Tabel ARP Router Tidak Update** | Switch jaringan mengabaikan paket Gratuitous ARP dari speaker MetalLB baru. | Jalankan perintah `arping -U -c 5 -I eth0 <VIP_IP>` secara manual dari node leader untuk memaksa pembaruan tabel MAC address switch. |

---

## 5. Cheatsheet & Ringkasan Perintah Game Day

```bash
# ==============================================================================
# 1. PERINTAH SUNTIKAN KEGAGALAN TERKONTROL (CHAOS INJECTION)
# ==============================================================================

# Simulasi GD-01: Mematikan service agent k3s pada salah satu node worker
ssh worker-01 "sudo systemctl stop k3s-agent"

# Simulasi GD-02: Mengisi partisi disk secara terkontrol (Hapus setelah selesai!)
ssh worker-01 "sudo fallocate -l 10G /var/log/dummy-fill.log"

# Simulasi GD-05: Menyuntikkan latensi jaringan buatan 1.5 detik pada interface eth0
ssh worker-01 "sudo tc qdisc add dev eth0 root netem delay 1500ms"
# Menghapus suntikan latensi jaringan (Pembersihan / Cleanup)
ssh worker-01 "sudo tc qdisc del dev eth0 root"

# ==============================================================================
# 2. EKSEKUSI PEMULIHAN CEPAT DALAM GAME DAY (< 5 MENIT)
# ==============================================================================

# Membatalkan rilis aplikasi yang cacat seketika
kubectl argo rollouts abort payment-api -n production
kubectl argo rollouts undo payment-api -n production

# Mengisolasi worker node yang bermasalah agar pod tidak dijadwalkan ke sana
kubectl cordon worker-01
```

---

## 6. Evaluasi Pemahaman Mandiri

### 6.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa dalam aturan baku pelaksanaan **SRE Game Day**, setiap iterasi pengujian **HANYA BOLEH menyuntikkan satu kegagalan (*One Fault at a Time*)** ke dalam sistem?
- A. Karena baterai laptop instruktur tidak cukup untuk menguji dua hal sekaligus.
- B. Untuk menjaga kejelasan observasi ilmiah; menyuntikkan dua kegagalan sekaligus akan mengaburkan hubungan sebab-akibat (*cause-and-effect*), menyulitkan pembacaan metrik telemetri, dan memicu risiko kerusakan di luar batas kendali yang tidak bisa dipulihkan.
- C. Karena Kubernetes secara otomatis mematikan server jika mendeteksi dua error.
- D. Agar waktu ujian Game Day bisa selesai lebih cepat.

#### Soal 2
Pada skenario **GD-03 (Bad Release)**, sebuah versi aplikasi baru memicu lonjakan error HTTP 5xx hingga 60%. Berapa batas waktu maksimal (**Target Waktu Pemulihan / Rollback SLA**) yang harus dicapai oleh tim SRE untuk menghentikan dampak bisnis sejak deklarasi insiden?
- A. Kurang dari 1 jam.
- B. Kurang dari 30 menit.
- C. **Kurang dari 5 menit**, dieksekusi melalui perintah pembatalan controller instan (*Fast-Track Rollback*) atau Emergency Git Revert.
- D. Tepat pada jam kerja keesokan harinya.

#### Soal 3
Apa peran seorang **Scribe (Pencatat Riwayat)** di dalam struktur *Incident Command System (ICS)* saat Game Day berlangsung?
- A. Menyajikan makanan ringan untuk seluruh peserta ujian.
- B. Mencatat seluruh lini masa kronologis secara *real-time* menggunakan stempel waktu resmi (UTC), mendokumentasikan setiap keputusan yang diambil oleh Incident Commander, mencatat tindakan yang dijalankan oleh Operations Lead, serta merekam bukti metrik sebelum dan sesudah mitigasi.
- C. Memperbaiki kode bug aplikasi di repositori pengembang.
- D. Mematikan sambungan internet seluruh peserta yang gagal menjawab pertanyaan.

---

### 6.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Saat sesi simulasi Game Day skenario **GD-01 (Worker Node Down)**, instruktur mematikan server fisik `k3s-worker-01` secara mendadak.
Alarm `KubeNodeNotReady` berbunyi di PagerDuty.
Peserta yang bertindak sebagai **Operations Lead** langsung terburu-buru membuka terminal dan mengetik:
`kubectl drain k3s-worker-01 --force --delete-emptydir-data`
Perintah tersebut macet (*hanging*) selama 10 menit tanpa hasil.

Sementara itu, peserta yang bertindak sebagai **Incident Commander** hanya diam menonton layar terminal Operations Lead tanpa mengambil keputusan apa pun, dan **Communications Lead** tidak mengirimkan satu pun update ke saluran obrolan pemangku kepentingan.

Dewan penguji memutuskan sesi drill tersebut **GAGAL (*DRILL FAILED*)**.

**Pertanyaan & Tugas Anda sebagai SRE Coach:**
1. Mengapa perintah `kubectl drain` macet saat dijalankan pada node yang komputernya sudah mati secara fisik? Perintah apa yang seharusnya dijalankan terlebih dahulu untuk mengisolasi node tersebut?
2. Sebutkan **2 kegagalan kepemimpinan & proses ICS** yang terjadi pada simulasi di atas (terkait Incident Commander dan Communications Lead)!
3. Tuliskan urutan penanganan yang benar untuk skenario Worker Node Down ini sesuai panduan Runbook standar SRE!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Metodologi *Chaos Engineering* menuntut eksperimen terisolasi dengan batas ledakan (*blast radius*) yang terkontrol ketat.
2. **C.** Dalam skenario Game Day Capstone, kemampuan membatalkan rilis buruk (*rollback*) di bawah 5 menit adalah tolok ukur kesiapan operasional utama.
3. **B.** Data pencatatan Scribe adalah bahan baku utama untuk menyusun dokumen evaluasi *Blameless Postmortem* yang akurat dan berbasis fakta.

#### Solusi Soal Kasus Nyata:
1. **Penyebab `kubectl drain` Macet:**
   - Perintah `kubectl drain` mencoba menghubungi agen Kubelet di node target untuk meminta penghapusan Pod secara tertib (*graceful eviction*). Karena server fisiknya sudah mati total, Kubelet tidak merespons panggilan API server sehingga perintah drain mengalami *timeout* tanpa henti.
   - **Tindakan Awal yang Benar:** Cukup tandai node dengan **`kubectl cordon k3s-worker-01`** agar scheduler tidak mencoba menjadwalkan pod baru ke server yang mati tersebut, lalu periksa sisa kapasitas memori klaster.
2. **Dua Kegagalan Proses ICS:**
   - **Kegagalan Incident Commander (IC):** Melanggar aturan *"No Hands on Keyboard / Keep Macro View"*. IC terjebak dalam kebingungan teknis Ops Lead, tidak menetapkan batas waktu tunggu (*timeout stop condition*), dan tidak mengarahkan tindakan alternatif.
   - **Kegagalan Communications Lead:** Membiarkan pemangku kepentingan berada dalam kegelapan (*silent failure*) tanpa informasi dampak dan perkiraan pemulihan (*ETA*), melanggar protokol komunikasi insiden.
3. **Urutan Penanganan yang Benar:**
   - **Langkah 1:** Deklarasikan insiden dan tentukan tingkat keparahan (*P1*).
   - **Langkah 2 (Cordon):** Jalankan `kubectl cordon k3s-worker-01`.
   - **Langkah 3 (Capacity Check):** Periksa apakah sisa node worker (`worker-02`) sanggup menampung pod yang terdampak (*N+1 Headroom Check*).
   - **Langkah 4 (Hardware Triage):** Periksa daya fisik via IPMI / konsol virtualisasi OrbStack (`orb list` / `orb start`).
   - **Langkah 5 (Comms Update):** Communications Lead mem-posting ringkasan: *"Terdeteksi worker-01 NotReady, beban kerja sedang dialihkan, kapasitas klaster masih aman, estimasi pemulihan 15 menit"*.
   - **Langkah 6 (Uncordon):** Setelah server menyala dan berstatus `Ready`, buka kembali node: `kubectl uncordon k3s-worker-01`.

---

## 7. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai eksekusi skenario Game Day dan manajemen peran ICS:
- Lanjutkan ke materi pamungkas penentuan kelulusan: [9.3.2 — Readiness Review dan Postmortem](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-9-capstone/modul-9.3-game-day-graduation/02-readiness-review-postmortem.md) untuk menyusun paket bukti audit kelulusan (*Evidence Packet*) dan mempresentasikan laporan Postmortem di hadapan dewan penguji Capstone!
- Praktikkan simulasi di [LAB-01 — Game Day Drill Lab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-9-capstone/modul-9.3-game-day-graduation/lab/LAB-01-game-day-drill.md).
