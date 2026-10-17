# 02 — On-Call, Incident Response, dan Change Management

> **Ringkasan:** Membangun ekosistem operasional SRE yang manusiawi dan terstruktur melalui manajemen jadwal piket (**On-Call Engineering**), memimpin penanganan krisis sistem menggunakan struktur komando darurat (**Incident Command System / ICS**), menyusun investigasi pasca-insiden tanpa menyalahkan individu (**Blameless Postmortem**), serta menerapkan tata kelola perubahan (**Change Management**) dan pembekuan rilis (**Change Freeze**).

---

## 1. Tata Kelola On-Call yang Berkelanjutan (Sustainable On-Call)

### 1.1 Analogi Dunia Nyata: "Petugas Piket IGD Rumah Sakit & Pager Darurat Dokter Spesialis"
Bayangkan sebuah Instalasi Gawat Darurat (IGD) di rumah sakit besar:
- Jika telepon darurat berdering setiap 3 menit hanya untuk hal-hal sepele (seperti *"kertas resep di lantai 3 habis"*), dokter jaga akan mengalami kelelahan mental ekstrem (*alert fatigue*). Ketika pasien darurat dengan serangan jantung tiba-tiba datang, dokter sudah terlalu lelah untuk berpikir jernih.
- Sebaliknya, rumah sakit menerapkan **Paging Hygiene**: Telepon darurat dokter jaga **hanya boleh berdering** jika ada kondisi pasien yang mengancam nyawa (**Symptom-based Alerting**).
- Selain itu, ada **Dokter Jaga Utama (*Primary*)** dan **Dokter Jaga Cadangan (*Secondary*)**. Jika Dokter Utama sedang menangani operasi kritis di meja bedah dan ada panggilan baru, Dokter Cadangan otomatis mengambil alih panggilan agar tidak ada pasien yang terlantar.

Di dunia SRE, **On-Call** bukan hukuman lembur, melainkan tanggung jawab rekayasa bergilir untuk menjaga keandalan sistem produksi secara profesional tanpa mengorbankan kesehatan fisik dan mental engineer.

---

### 1.2 Matriks Kontrak On-Call (The On-Call Matrix)

Setiap tim SRE wajib mendokumentasikan aturan main piket ke dalam **On-Call Matrix**:

| Elemen Kontrak | Standar Produksi SRE | Rationale Operasional |
|---|---|---|
| **Peran Petugas** | - **Primary On-Call:** Penerima paging pertama, memimpin respon awal.<br>- **Secondary On-Call:** Cadangan jika Primary tidak merespons dalam 5 menit, menangani insiden kedua.<br>- **Shadow On-Call:** Engineer baru yang ikut menyimak proses on-call untuk belajar (*training mode*). | Mencegah *Single Point of Failure (SPOF)* pada manusia dan melatih anggota tim baru secara aman. |
| **Pola Rotasi** | **Weekly Rotation (1 Minggu per orang)** bergantian di dalam tim (minimal 5–8 engineer per putaran rotasi). | Rotasi di bawah 4 orang akan memicu kelelahan (*burnout*). Jika tim tersebar di 2 benua, gunakan pola **Follow-the-Sun** (piket hanya di jam siang lokal). |
| **Ambang Batas Paging** | **HANYA alert dengan `severity="critical"` yang mengancam SLO/bisnis** yang boleh membunyikan HP on-call. Alert warning hanya membuat tiket kerja atau pesan Slack biasa. | Menghindari *Pager Fatigue*. Standar Google SRE: rata-rata maksimal **2 paging per shift 12 jam**. Jika lebih dari itu, sistem dianggap tidak sehat! |
| **Batas Waktu Respon (*SLA Ack*)** | - **Jam Kerja:** Maksimal 5 menit.<br>- **Luar Jam Kerja (Malam/Hari Libur):** Maksimal 15 menit. | Jika tidak ada *Acknowledge* dalam 15 menit, sistem otomatis eskalasi ke Secondary On-Call dan Engineering Manager. |
| **Ritual Serah Terima (*Handoff*)** | Dilakukan setiap hari Senin pukul 10:00 (sinkronisasi UTC) melalui panggilan video singkat dan pencatatan tiket resmi. | Menjelaskan insiden terbuka, status Error Budget terbaru, pekerjaan pemeliharaan yang sedang berjalan, dan alert yang sempat berbunyi. |

> [!CAUTION]
> **Aturan Higienitas Rahasia (*Secrets Hygiene*) pada Dokumentasi On-Call:**
> Jangan pernah mencantumkan nomor telepon pribadi, kredensial akun PagerDuty/Opsgenie, atau webhook URL yang memuat token rahasia ke dalam repositori Git! Gunakan placeholder deskriptif seperti `<ONCALL_PRIMARY_SCHEDULE>` dan referensikan akun resmi perusahaan yang terhubung melalui SSO (Single Sign-On).

---

## 2. Struktur Komando Tanggap Insiden (Incident Command System / ICS)

Ketika terjadi insiden besar (P1/P2) di lingkungan produksi, pola komunikasi santai di grup obrolan harus segera ditinggalkan. SRE mengadopsi standar **Incident Command System (ICS)**—protokol penanganan bencana yang digunakan oleh pemadam kebakaran dan tim SAR internasional.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 STRUKTUR PERAN INCIDENT COMMAND SYSTEM (ICS)                │
│                                                                             │
│                        ┌─────────────────────────┐                          │
│                        │    INCIDENT COMMANDER   │                          │
│                        │          (IC)           │                          │
│                        │ "Pemimpin & Pengambil   │                          │
│                        │  Keputusan Tertinggi"   │                          │
│                        └────────────┬────────────┘                          │
│                                     │                                       │
│         ┌───────────────────────────┼───────────────────────────┐           │
│         ▼                           ▼                           ▼           │
│  ┌───────────────┐          ┌───────────────┐          ┌─────────────────┐  │
│  │   OPERATIONS  │          │ COMMUNICATIONS│          │     SCRIBE      │  │
│  │     LEAD      │          │     LEAD      │          │  (Pencatat Log) │  │
│  │ (Pelaksana    │          │ (Juru Bicara  │          │ (Mencatat Waktu │  │
│  │  Mitigasi/    │          │  Stakeholder/ │          │  UTC, Keputusan │  │
│  │  Investigasi) │          │  Status Page) │          │  & Fakta Nyata) │  │
│  └───────────────┘          └───────────────┘          └─────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Pembagian Tugas 4 Peran Kunci ICS
1. **Incident Commander (IC):**
   - **Pemimpin tertinggi selama insiden.** Memiliki wewenang absolut untuk menyetujui rollback, menghentikan deployment, atau memutuskan pemadaman sementara (*graceful degradation*).
   - **ATURAN EMAS IC: "NO HANDS ON KEYBOARD!"** IC **dilarang keras** mengetik perintah teknis atau membaca kode program sendiri! Tugas IC adalah menjaga pandangan helikopter (*macro view*), menetapkan hipotesis, menunjuk pelaksana, dan menjaga agar tim tidak panik.
2. **Operations Lead (Ops Lead):**
   - Engineer teknis senior yang mengeksekusi perintah investigasi, membaca metrik Prometheus/Mimir, menganalisis trace Tempo, dan menjalankan mitigasi sesuai Runbook.
3. **Communications Lead (Comms Lead):**
   - Bertugas sebagai juru bicara. Menyampaikan perkembangan berkala setiap 15–30 menit ke manajemen eksekutif, tim customer support, dan memperbarui halaman **Status Page Publik** menggunakan bahasa awam yang menenangkan (tanpa istilah teknis membingungkan dan tanpa membocorkan data sensitif!).
4. **Scribe (Pencatat Riwayat):**
   - Mencatat seluruh lini masa (*timeline*) insiden secara *real-time*: jam berapa anomali terdeteksi, keputusan apa yang diambil oleh IC, perintah apa yang dijalankan oleh Ops Lead, dan bagaimana hasilnya.

---

### 2.2 Siklus Hidup Insiden (Incident Lifecycle)

```text
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 1. DETECTION │ ──> │2. DECLARATION│ ──> │  3. TRIAGE   │ ──> │4. MITIGATION │
│ (Alarm/User) │     │ (Buat WarRoom│     │ (Inspeksi    │     │  & ROLLBACK  │
│              │     │  & Tetapkan) │     │  Read-Only)  │     │ (Stop Dampak)│
└──────────────┘     └──────────────┘     └──────────────┘     └──────┬───────┘
                                                                      │
┌──────────────┐     ┌──────────────┐     ┌──────────────┐            │
│ 7. POSTMORTEM│ <── │  6. CLOSURE  │ <── │5. RESOLUTION │ <──────────┘
│ (Belajar     │     │ (Kunci Bukti │     │(SLO & Metrik │
│  Bersama)    │     │  Audit Trail)│     │ Normal >30m) │
└──────────────┘     └──────────────┘     └──────────────┘
```

1. **Detection (Deteksi):** PagerDuty berbunyi dari alert *Burn Rate Fast Burn* atau laporan kegagalan transaksi dari payment gateway.
2. **Declaration (Deklarasi Resmi):** Primary On-Call secara resmi menyatakan: *"Saya mendeklarasikan insiden P1 pada Payment API, saya bertindak sebagai Incident Commander sementara, silakan bergabung ke War Room!"*.
3. **Triage (Diagnosis Aman):** Lakukan pemeriksaan **Read-Only** terlebih dahulu! Inspeksi Dashboard RED di Grafana, periksa status Pod di ArgoCD, dan bandingkan commit Git terakhir.
4. **Mitigation & Rollback:** **Fokus utama adalah MENGHENTIKAN DAMPAK BISNIS, bukan mencari tahu bug di baris kode berapa!** Jika masalah terjadi setelah rilis baru, segera lakukan `kubectl argo rollouts abort` atau `git revert` tanpa ragu.
5. **Resolution (Pemulihan):** Trafik kembali normal, error rate turun di bawah ambang batas SLO, dan sistem dipantau selama minimal 30 menit (*soak time*).
6. **Closure (Penutupan):** IC menyatakan insiden selesai, mengumumkan ke stakeholder, dan mengunci kanal War Room.
7. **Postmortem:** Tim berkumpul dalam 24–48 jam untuk menyusun evaluasi insiden secara terbuka.

---

## 3. Investigasi Pasca-Insiden Tanpa Menyalahkan (Blameless Postmortem)

### 3.1 Mengapa Budaya "Blameless" Sangat Penting bagi Keandalan?
Di perusahaan dengan budaya buruk (*Blame Culture*), ketika seorang engineer salah mengetik perintah dan server mati, ia akan dimarahi, dipermalukan, atau dipecat. Akibatnya:
- Engineer lain akan menyembunyikan kesalahan mereka di masa depan.
- Tidak ada yang berani menyentuh atau memperbaiki sistem produksi.
- Masalah yang sama akan terulang kembali karena sistem dasarnya tetap rapuh.

Dalam **Budaya Blameless SRE (Diadopsi dari Industri Penerbangan & Bedah Medis)**:
- **Asumsi Dasar:** Kita berasumsi bahwa seluruh karyawan adalah orang pintar, memiliki niat baik, dan berusaha melakukan pekerjaan terbaik dengan informasi yang mereka miliki saat itu.
- **Jika seseorang bisa salah mengetik perintah dan merusak seluruh klaster, MAKA SISTEM KITALAH YANG RUSAK!** Mengapa sistem mengizinkan 1 kesalahan manusia (*human error*) merusak segalanya? Mengapa tidak ada validasi otomatis, pembatasan hak akses (*least privilege*), atau pengujian dry-run?

---

### 3.2 Template Laporan Postmortem Standar SRE

```markdown
# POSTMORTEM: [INC-2026-10-17] Pemadaman Transaksi Payment API Akibat Salah Konfigurasi Database Connection Pool

| Parameter Insiden | Detail Catatan Resmi |
|---|---|
| **Tanggal Kejadian** | 17 Oktober 2026 |
| **Tingkat Keparahan** | **P1 (Critical Outage)** |
| **Durasi Pemadaman** | 38 Menit (14:12 UTC – 14:50 UTC) |
| **Dampak Bisnis** | 4.820 transaksi gagal (Error 504), perkiraan kerugian transaksi: Rp 420.000.000 |
| **Incident Commander** | SRE Lead (Team Platform) |
| **Status Dokumen** | **APPROVED & ACTION ITEMS ASSIGNED** |

---

## 1. Ringkasan Eksekutif (Executive Summary)
Pada pukul 14:12 UTC, layanan `payment-api` mengalami degradasi tajam setelah sinkronisasi otomatis GitOps menerapkan perubahan nilai `max_connections` dari 100 menjadi 5. Hal ini menyebabkan antrean koneksi jenuh seketika dan memicu lonjakan error HTTP 504 hingga 85%. Insiden termitigasi pada pukul 14:50 UTC setelah tim melakukan pembatalan (*emergency revert MR*) di repositori GitOps.

## 2. Garis Waktu Kronologis (Timeline dalam Format Waktu UTC)
- **14:05 UTC** — Merge Request `#412` di-merge ke branch `main` repositori GitOps oleh developer payment.
- **14:12 UTC** — ArgoCD menyelesaikan sinkronisasi ke klaster produksi.
- **14:14 UTC** — PagerDuty membunyikan alarm *PaymentApiSLOErrorBudgetFastBurn* ke Primary On-Call.
- **14:16 UTC** — Primary On-Call mendeklarasikan Insiden P1 dan membuka kanal War Room.
- **14:22 UTC** — Ops Lead mengidentifikasi lewat grafik metode USE bahwa pod PostgreSQL kehabisan pool koneksi.
- **14:35 UTC** — IC menginstruksikan Emergency Revert commit pada repositori GitOps.
- **14:45 UTC** — ArgoCD melakukan rekonsiliasi ulang konfigurasi lama yang stabil.
- **14:50 UTC** — Error rate turun kembali ke 0,01%; IC menyatakan sistem kembali sehat.

## 3. Faktor Berkontribusi (5 Whys / Contributing Factors)
1. *Mengapa payment-api mengalami error 504?* Karena antrean koneksi database jenuh.
2. *Mengapa pool koneksi jenuh?* Karena nilai `max_connections` diturunkan menjadi 5.
3. *Mengapa nilai 5 diterapkan di produksi?* Developer salah menyalin file `values-dev.yaml` ke `values-prod.yaml`.
4. *Mengapa kesalahan salin file tersebut tidak terdeteksi sebelum rilis?* Pipeline CI repositori GitOps belum memiliki uji validasi batas nilai minimum (*schema validation*).
5. *Mengapa reviewer menyetujui MR tersebut?* Reviewer mengira nilai tersebut adalah konfigurasi timeout, bukan ukuran pool koneksi.

## 4. Pelajaran Berharga (What Went Well vs What Went Poorly)
- **Yang Berjalan Baik:**
  - Alert burn-rate Mimir berbunyi sangat cepat (kurang dari 2 menit setelah anomali terjadi).
  - Alur eskalasi ICS berjalan tertib tanpa perdebatan personal.
- **Yang Berjalan Kurang Baik:**
  - Proses revert GitOps memakan waktu 13 menit karena engineer sempat bingung membedakan branch target.

## 5. Rencana Tindakan Pencegahan Permanen (Action Items — Wajib SMART!)
| No | Tindakan Korektif Permanen | Kategori | Tim Penanggung Jawab | Tenggat Waktu | Verifikasi Keberhasilan |
|---|---|---|---|---|---|
| **1** | Pasang validator `kube-linter` / `datree` di CI GitOps untuk melarang `max_connections < 50` di produksi | **Pencegahan** | `@sre-platform` | 24 Okt 2026 | Uji coba MR gagal jika diset ke nilai 5 |
| **2** | Perbaiki Runbook darurat Payment API dengan perintah 1-baris fast rollback | **Mitigasi** | `@payment-squad` | 21 Okt 2026 | Drill uji coba di Staging lab |
| **3** | Tambahkan metrik saturasi pool koneksi database ke Dashboard RED Payment | **Deteksi** | `@sre-platform` | 28 Okt 2026 | Grafik pool tampil di Grafana |
```

---

## 4. Manajemen Perubahan & Pembekuan Rilis (Change Management & Change Freeze)

Perubahan adalah penyebab nomor satu dari seluruh pemadaman sistem di dunia teknologi (*"Changes cause outages"*). SRE menerapkan klasifikasi perubahan untuk menyeimbangkan antara kecepatan rilis dan keamanan:

### 4.1 Empat Kelas Perubahan Sistem (Change Classification)

| Kelas Perubahan | Kriteria & Contoh Nyata | Jalur Persetujuan & Mitigasi |
|---|---|---|
| **1. Standard Change** | Perubahan rutin, berulang, risiko sangat rendah, dan terotomasi penuh. *(Contoh: Update tag image patch aplikasi mingguan via pipeline CI/CD yang sudah lulus tes unit)*. | **Pre-Approved.** Cukup *Peer Review* 1 rekan tim di MR GitOps, tidak perlu rapat izin khusus. |
| **2. Normal Change** | Perubahan pada komponen penting dengan risiko sedang. *(Contoh: Penyesuaian resource CPU/RAM pod, penambahan ingress route baru, upgrade versi Helm chart minor)*. | Membutuhkan **Tinjauan SRE & Developer**, wajib memiliki rencana pengujian dan *Rollback Plan* tertulis di deskripsi MR. |
| **3. High-Risk Change** | Perubahan fundamental pada fondasi infrastruktur. *(Contoh: Migrasi skema database besar, upgrade versi kernel k3s, perubahan subnet VLAN/MetalLB, penggantian sertifikat root CA)*. | Wajib melalui **Lightweight CAB (Change Advisory Board)**, dilakukan di luar jam sibuk (*Maintenance Window*), dan wajib diuji coba di lingkungan replika (*Staging/Lab rehearsal*). |
| **4. Emergency Change** | Perubahan darurat yang dilakukan saat insiden P1 sedang berlangsung untuk memulihkan layanan. *(Contoh: Hotfix darurat membatalkan fitur yang crash, penambahan kapasitas darurat)*. | **Persetujuan Instan oleh Incident Commander (IC)**. Dokumen analisis dan pencatatan audit diselesaikan dalam waktu 24 jam setelah insiden selesai (*retrospective review*). |

---

### 4.2 Kapan Kebijakan Change Freeze Diberlakukan?

**Change Freeze** adalah periode di mana **DILARANG KERAS** melakukan perubahan apa pun ke lingkungan produksi (kecuali *Emergency Fix* yang disetujui pimpinan).

Tiga Kondisi Pemicu Change Freeze:
1. **Error Budget Terkuras Habis:** Sisa Error Budget bulanan mencapai `<= 0%` (Sesuai kesepakatan tata kelola SRE).
2. **Peristiwa Bisnis Puncak (*Peak Business Events*):** Momen krusial bagi perusahaan di mana sistem dilarang mengalami gangguan (misal: Hari Penggajian/Payroll tanggal 25–28, Flash Sale Promo Akhir Tahun 12.12, atau Pemilu Nasional).
3. **Krisis Operasional:** Terjadi bencana alam di data center utama atau separuh tim teknis sedang cuti bersama.

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **On-Call Engineer Mengabaikan Alarm Paging (Missed Page)** | PagerDuty disetel ke nada dering yang sama dengan notifikasi pesan biasa, atau engineer mengalami *alert fatigue* akibat ratusan pesan palsu. | 1. Gunakan aplikasi pager khusus dengan izin *Critical Alert* yang mampu menembus mode *Silent / Do Not Disturb* ponsel.<br>2. Evaluasi aturan alert: hapus atau naikkan ambang batas alert yang tidak memerlukan tindakan segera. |
| **War Room Insiden Kacau & Banyak Orang Bicara Bersamaan** | Tidak ada penunjukan **Incident Commander (IC)** yang jelas, atau pimpinan eksekutif masuk dan langsung memberi instruksi teknis tanpa koordinasi. | Terapkan disiplin ICS: Hanya IC yang berhak membagi giliran bicara. Jika pimpinan eksekutif hadir, arahkan mereka untuk berkomunikasi dengan **Communications Lead** agar Ops Lead tidak terganggu. |
| **Tindakan Perbaikan Darurat Justru Menghapus Data Pengguna** | Engineer panik saat insiden dan menjalankan perintah destruktif tanpa rencana rollback. | Terapkan aturan: **Setiap tindakan mutasi darurat wajib disebutkan ke IC dan diverifikasi dampaknya sebelum tombol Enter ditekan!** |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. PENGENDALIAN INSIDEN DARURAT DI KUBERNETES & ARGO ROLLOUTS
# ==============================================================================

# Menghentikan proses rilis canary seketika dan mengembalikan trafik ke versi stabil
kubectl argo rollouts abort payment-api -n production

# Mengembalikan spesifikasi Rollout ke revisi sebelumnya yang terbukti sehat
kubectl argo rollouts undo payment-api -n production

# Mengunci sinkronisasi otomatis ArgoCD agar tidak menimpa perbaikan darurat
argocd app set prod-payment-api --sync-policy manual

# ==============================================================================
# 2. PENGUMPULAN BUKTI KRONOLOGIS UNTUK DOKUMEN POSTMORTEM (UTC)
# ==============================================================================

# Menampilkan riwayat event cluster terurut berdasarkan waktu pembuatan
kubectl get events -n production --sort-by='.metadata.creationTimestamp'

# Mengambil log container sebelum pod mengalami restart terakhir kali (--previous)
kubectl logs -n production payment-api-xyz --previous --tail=100
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Dalam struktur **Incident Command System (ICS)**, apa peran dan batasan utama seorang **Incident Commander (IC)** selama menangani insiden besar (P1)?
- A. Mengetik kode program perbaikan dan menulis kueri database secara langsung di terminal produksi.
- B. Memimpin proses penanganan, menetapkan prioritas, menjaga pandangan makro, mengoordinasikan peran, dan **dilarang mengetik perintah teknis sendiri (*No Hands on Keyboard*)** agar tidak kehilangan fokus kepemimpinan.
- C. Menghubungi keluarga seluruh engineer yang sedang bertugas.
- D. Mengganti kata sandi seluruh server secara acak.

#### Soal 2
Mengapa laporan analisis pasca-insiden SRE **wajib berprinsip Blameless (tanpa menyalahkan individu)**?
- A. Karena di dunia teknologi tidak ada orang yang pernah melakukan kesalahan.
- B. Agar manajemen tidak perlu membayar asuransi kecelakaan kerja.
- C. Karena menyalahkan individu akan membuat tim menyembunyikan kesalahan dan takut bereksperimen, sedangkan budaya *Blameless* berfokus memperbaiki kelemahan sistem, proses validasi, dan otomasi agar kesalahan manusia yang sama mustahil merusak sistem di masa depan.
- D. Karena dokumen postmortem hanya dibaca oleh mesin pencari.

#### Soal 3
Kondisi manakah di bawah ini yang menjadi pemicu resmi diberlakukannya **Change Freeze** di lingkungan produksi menurut tata kelola SRE?
- A. Setiap kali hari Jumat sore tiba.
- B. Sisa **Error Budget bulanan telah habis (0% remaining)**, sedang berlangsung pemadaman insiden besar P1, atau sedang berlangsung periode bisnis puncak perusahaan (*Peak Business Events*).
- C. Ketika kuota internet kantor sedang lambat.
- D. Setiap kali ada karyawan baru yang masuk kerja.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Pada pukul 11:00 siang di hari kerja biasa, seorang engineer junior bernama Andi diminta untuk mengosongkan antrean cache pada Redis staging. Namun, Andi salah memilih jendela terminal dan tidak sengaja menjalankan perintah pembersihan `FLUSHALL` di cluster **Redis Production**. Akibatnya, sesi login 50.000 pengguna aktif terputus seketika dan gateway otentikasi lumpuh selama 15 menit.

Manajer Divisi marah besar dan mengirim email ke seluruh kantor:
*"Ini kelalaian fatal dari Andi! Andi harus diskors selama 2 minggu, dan mulai hari ini tidak ada engineer junior yang boleh menyentuh terminal produksi!"*

**Pertanyaan & Tugas Anda sebagai Senior SRE:**
1. Jelaskan mengapa reaksi Manajer Divisi tersebut bertentangan dengan prinsip **Blameless Culture** dan justru akan merusak keandalan jangka panjang perusahaan!
2. Mengapa insiden tersebut pada dasarnya adalah **kegagalan sistem dan desain arsitektur**, bukan kesalahan pribadi Andi?
3. Rancanglah **3 Action Items (Tindakan Pencegahan)** konkret yang harus diusulkan di dalam dokumen Postmortem agar insiden salah ketik perintah seperti ini tidak akan pernah bisa terjadi lagi oleh siapa pun!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Peran IC adalah pengambil keputusan dan konduktor orkestrasi darurat. Begitu seorang IC mulai mengetik perintah di shell, ia akan terjebak dalam *tunnel vision* dan kehilangan kendali atas koordinasi tim.
2. **C.** Budaya menyalahkan individu menciptakan rasa takut (*culture of fear*) yang berujung pada penutupan informasi, padahal transparansi adalah bahan bakar utama perbaikan keandalan sistem (*Just Culture*).
3. **B.** *Change Freeze* ditegakkan sebagai rem darurat bisnis untuk melindungi stabilitas sistem saat risiko kegagalan sudah tidak memiliki toleransi anggaran (*zero budget*).

#### Solusi Soal Kasus Nyata:
1. **Bahaya Reaksi Manajer:**
   - Menghukum Andi tidak menghilangkan risiko insiden serupa di masa depan karena orang lain (termasuk engineer senior yang sedang lelah) tetap bisa salah memilih tab terminal. Reaksi tersebut memicu budaya menutup-nutupi kesalahan (*culture of secrecy*), menurunkan moral tim, dan melarang junior belajar justru menghambat regenerasi organisasi.
2. **Bukti Bahwa Ini Adalah Kegagalan Sistem:**
   - Sistem membiarkan terminal Staging dan Production memiliki tampilan visual yang sama persis tanpa pembeda warna yang tegas.
   - Perintah destruktif `FLUSHALL` diizinkan berjalan di Redis produksi tanpa proteksi *command renaming* atau otorisasi bertingkat.
   - Hak akses (*privilege*) Andi tidak dibatasi: mengapa seorang engineer junior memiliki akses langsung ke port eksekusi Redis produksi padahal tugasnya hanya di staging (*pelanggaran prinsip Least Privilege*)?
3. **Tiga Usulan Action Items yang Konstruktif:**
   - **Tindakan 1 (Konfigurasi Keamanan Redis):** Ubah konfigurasi Redis produksi melalui Ansible: nonaktifkan atau ganti nama perintah berbahaya dengan direktif `rename-command FLUSHALL ""` dan `rename-command FLUSHDB ""` di `redis.conf`.
   - **Tindakan 2 (Visual Boundary di Terminal):** Konfigurasi terminal jump-host/bastion agar otomatis menampilkan latar belakang merah menyala dan teks peringatan besar ketika terhubung ke server produksi.
   - **Tindakan 3 (Access Control & Bastion Hardening):** Pisahkan kredensial jump-host antara staging dan production menggunakan sistem akses berbasis peran (RBAC / Teleport) dengan persetujuan ganda (*dual-authorization approval*) untuk akses tingkat administrator.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai manajemen On-Call, respons insiden, dan tata kelola perubahan:
- Praktikkan simulasi di [LAB-02 — Incident Response dan Change Freeze Drill](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.1-praktik-sre/lab/LAB-02-incident-response-change-freeze.md).
- Lanjutkan ke modul berikutnya: [Modul 8.2 — Operasi Produksi On-Premises](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.2-production-onprem/README.md) untuk mempelajari realitas operasional jaringan fisik bare-metal, storage class, perangkat keras, dan akses bastion.
