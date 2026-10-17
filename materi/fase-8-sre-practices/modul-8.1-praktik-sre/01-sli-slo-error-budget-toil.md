# 01 — SLI, SLO, Error Budget, Burn Rate, dan Toil

> **Ringkasan:** Memahami pilar fundamental rekayasa keandalan Google SRE: mendefinisikan indikator keandalan (**SLI**), menetapkan target layanan (**SLO**), menegakkan kebijakan **Error Budget**, mengukur laju konsumsi anggaran (**Burn Rate**), serta mengidentifikasi dan membasmi kerja repetitif tanpa nilai permanen (**Toil**) melalui otomasi berpagar (*guarded automation*).

---

## 1. Konsep & Arsitektur Keandalan Berbasis Data

### 1.1 Analogi Dunia Nyata: "Kartu Garansi Mesin Pabrik, Jatah Cuti Sakit, dan Pekerjaan Robotik Sia-Sia"
Bayangkan sebuah pabrik manufaktur modern:
1. **Target Kualitas Produk (*SLO = 99,9%*):** Direktur pabrik dan kepala teknisi bersepakat: dari setiap 1.000 botol minuman yang diproduksi, minimal **999 botol** harus tersegel sempurna dan higienis (**SLO**).
2. **Jatah Cacat yang Ditoleransi (*Error Budget = 0,1%*):** Teknisi menyadari bahwa mengejar kesempurnaan 100% (0 botol cacat) membutuhkan biaya sensor jutaan dolar dan membuat kecepatan mesin melambat hingga 90%. Jatah **1 botol cacat per 1.000 produksi** adalah kompromi bisnis yang sehat agar pabrik tetap bisa berinovasi dan bekerja cepat.
3. **Pembasmian Kerja Manual yang Melelahkan (*Toil*):** Setiap kali ada 1 botol yang macet di ban berjalan, seorang operator pabrik harus berdiri, menekan tuas darurat dengan tangan, dan menarik botol keluar. Pekerjaan ini manual, berulang setiap 2 jam, tidak membutuhkan pemikiran intelektual, dan menyita waktu operator dari tugas merancang mesin yang lebih baik. Dalam prinsip SRE, tugas seperti ini disebut **Toil**, dan tugas utama engineer adalah **membuat lengan robotik otomatis (*automation*)** agar masalah tersebut terselesaikan secara mandiri tanpa campur tangan manusia!

---

### 1.2 Hirarki Metrik Keandalan: Hubungan SLI, SLO, SLA, dan Error Budget

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 HUBUNGAN SLI, SLO, ERROR BUDGET, DAN SLA                    │
│                                                                             │
│  [1. SLI (Service Level Indicator)]                                         │
│   Pengukuran aktual di runtime:                                             │
│   SLI = (Permintaan Sukses yang Valid ÷ Total Permintaan yang Valid) × 100% │
│                                  │                                          │
│                                  ▼                                          │
│  [2. SLO (Service Level Objective)] ──> Sasaran Internal Engineering        │
│   Kesepakatan tim: SLI >= 99,9% dalam jendela 30 hari kalender.             │
│                                  │                                          │
│         ┌────────────────────────┴────────────────────────┐                 │
│         ▼                                                 ▼                 │
│  [3. ERROR BUDGET]                                 [4. SLA KONTRAK]         │
│   Jatah Kegagalan: 100% - SLO                      Janji Hukum ke Klien:    │
│   = 0,1% (43,2 menit downtime / 30 hari)           Availability >= 99,5%    │
│   ├─ Budget Masih Ada ──> Rilis Fitur Cepat!       (Denda penalti finansial │
│   └─ Budget Habis ──> Freeze Fitur & Perbaiki!     jika melanggar batas ini)│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Kontrak Spesifikasi SLO (The SLO Contract)

Menetapkan target SLO tidak boleh dilakukan secara sepihak atau dipilih secara acak setelah insiden terjadi. Tim SRE, Pengembang (*Developer*), dan Manajer Produk (*Product Owner*) wajib menandatangani **SLO Contract** tertulis yang disimpan di repositori Git:

### 2.1 Anatomi SLO Contract (`slo-contract-payment.yaml`)

```yaml
apiVersion: sre.bootcamp.internal/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: payment-api-availability
  namespace: production
spec:
  service: payment-api
  owner: payment-squad
  business_criticality: Tier-1

  # Jendela waktu evaluasi resmi (Rolling window 30 hari kalender)
  window: 30d
  objective: 99.9 # Target 99.9% availability

  # Definisi Indikator Kuantitatif (SLI)
  sli:
    type: availability
    description: "Persentase transaksi pembayaran yang selesai dengan sukses tanpa error 5xx atau timeout"
    
    # Kueri Numerator: Permintaan yang berhasil
    numerator_query: |
      sum(rate(http_requests_total{service="payment-api", status=~"2..|3..|400|404", sre_environment="production"}[5m]))
    
    # Kueri Denominator: Seluruh permintaan valid yang masuk
    denominator_query: |
      sum(rate(http_requests_total{service="payment-api", sre_environment="production"}[5m]))

    # Pengecualian yang Disetujui (Exclusion Policy)
    exclusions:
      - reason: "Jendela pemeliharaan database terjadwal dengan izin CAB"
        ticket_reference: "CHG-MAINT-POLICY"
      - reason: "Trafik uji penetrasi keamanan (Pen-test) bersertifikat"
        header_filter: 'http_request_header_x_test == "penetration-testing"'

  # Kebijakan Penanganan Ketiadaan Data Telemetri (Missing Data Policy)
  missing_data_policy:
    action: "investigate_and_alert" # Jangan pernah menganggap NoData sebagai sukses!
    grace_period: 2m

  # Jadwal Tinjauan Rutin (Cadence)
  review_cadence: "bi-weekly" # Ditinjau setiap 2 minggu dalam sprint planning
  runbook_reference: "https://git.internal.example.com/sre/runbooks/-/blob/main/payment/PaymentApiFastBurn.md"
```

---

### 2.2 Aturan Integritas Numerator & Denominator
1. **Denominator Harus Adil (*Eligible Traffic Only*):**
   - Hanya hitung trafik yang memang merupakan tanggung jawab layanan `payment-api`.
   - **Trafik yang Dikecualikan (*Excluded*):** Probe health check internal (`/healthz` dan `/readyz`), trafik uji beban (*load test*) sintetis dari tim QA, dan panggilan dari IP yang diblokir oleh firewall (*WAF blacklist*).
2. **Kapan Kode Status HTTP Dianggap Berhasil vs Gagal?**
   - **Sukses (Numerator):** `2xx` (Sukses), `3xx` (Redirect), serta `400 Bad Request` atau `404 Not Found` (karena kesalahan input dari sisi klien/pengguna, bukan kesalahan sistem server).
   - **Gagal (Bukan Numerator):** `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout`, dan **`429 Too Many Requests`** (karena server menolak pengguna akibat kapasitas server jenuh).

---

## 3. Matematika Error Budget & Penegakan Kebijakan (Enforcement Policy)

### 3.1 Perhitungan Kuota Kegagalan (Error Budget Math)

Jika sebuah layanan memproses **2.000.000 request valid** dalam sebulan dengan **SLO 99,9%**:
- **Fraksi Error Budget:** $1 - 0{,}999 = 0{,}001$ ($0{,}1\%$).
- **Jumlah Maksimal Request Gagal yang Ditoleransi:**
  $$2.000.000 \times 0{,}001 = 2.000 \text{ transaksi gagal/bulan}$$
- **Jatah Waktu Downtime Total dalam 30 Hari (43.200 menit):**
  $$43.200 \times 0{,}001 = 43{,}2 \text{ menit/bulan}$$

---

### 3.2 Matriks Keputusan Error Budget (Error Budget Policy)

Ketika Error Budget dikonsumsi oleh insiden, tim tidak boleh berdebat secara emosional. Organisasi wajib mengikuti **Aturan Main Penegakan Error Budget** yang disepakati bersama manajemen:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ MATRIKS KONSEKUENSI PENGGUNAAN ERROR BUDGET SRE                             │
├─────────────────┬────────────────────────────┬──────────────────────────────┤
│ Sisa Budget     │ Status Operasional Layanan │ Kebijakan Delivery & Rilis   │
├─────────────────┼────────────────────────────┼──────────────────────────────┤
│ 100% – 50%      │ **ZONA HIJAU (Normal)**    │ Developer bebas merilis      │
│                 │ Sistem stabil dan sehat.   │ fitur baru dengan alur GitOps│
│                 │                            │ standar tanpa pembatasan.    │
├─────────────────┼────────────────────────────┼──────────────────────────────┤
│ 49% – 20%       │ **ZONA KUNING (Waspada)**  │ Rilis fitur tetap berjalan,  │
│                 │ Terjadi kebocoran budget.  │ tetapi wajib menyertakan     │
│                 │                            │ mitigasi risiko ekstra dan   │
│                 │                            │ tinjauan ulang arsitektur.   │
├─────────────────┼────────────────────────────┼──────────────────────────────┤
│ 19% – 1%        │ **ZONA ORANYE (Kritis)**   │ Hanya perubahan berisiko     │
│                 │ Ambang batas hampir jebol. │ rendah yang diizinkan;       │
│                 │                            │ freeze rilis arsitektur baru.│
├─────────────────┼────────────────────────────┼──────────────────────────────┤
│ <= 0% (HABIS)   │ **ZONA MERAH (Defisit)**   │ **PRODUCTION FEATURE FREEZE!**│
│                 │ Keandalan telah gagal      │ Seluruh sprint dialihkan     │
│                 │ melampaui komitmen bisnis! │ 100% untuk stabilitas, bug,  │
│                 │                            │ observabilitas & runbook!    │
└─────────────────┴────────────────────────────┴───────────────────────────────┘
```

> [!CAUTION]
> **Pengecualian Freeze (Waiver Exception Process):**
> Jika ada kebutuhan bisnis yang sangat mendesak untuk merilis fitur di tengah kondisi *Feature Freeze*, pengecualian tersebut **wajib mendapatkan persetujuan tertulis dari VP of Engineering dan Head of Product**. Mereka harus secara sadar menerima risiko bahwa kegagalan berikutnya akan melanggar komitmen kontrak legal kepada pelanggan (**SLA Breach**)!

---

## 4. Memahami dan Membasmi Toil (Toil Reduction)

Salah satu prinsip paling revolusioner dari Google SRE adalah: **"SRE bukan tim Helpdesk atau Sysadmin operator tiket manual"**.

### 4.1 Definisi 6 Ciri Khas Toil
Menurut buku panduan resmi Google SRE, pekerjaan diklasifikasikan sebagai **Toil** jika memenuhi 6 karakteristik:
1. **Manual:** Membutuhkan manusia untuk mengetik perintah di terminal atau mengklik tombol di antarmuka web.
2. **Repetitif:** Pekerjaan yang sama dilakukan berulang kali (misal: setiap Senin pagi, atau setiap kali ada deploy).
3. **Dapat Diotomasi (*Automatable*):** Pekerjaan tersebut mengikuti langkah baku yang sebenarnya bisa dikerjakan oleh skrip shell, Ansible playbook, atau controller Kubernetes.
4. **Taktis (*Tactical / Non-Enduring Value*):** Menyelesaikan masalah saat itu saja, tetapi tidak membuat sistem menjadi lebih andal secara permanen (misal: merestart pod yang bocor memori).
5. **Tumbuh Linier dengan Skala Layanan ($O(n)$):** Jika server bertambah dari 5 menjadi 50, pekerjaan tersebut menyita waktu 10 kali lebih banyak.
6. **Tidak Membutuhkan Pemikiran Rekayasa Kreatif:** Pekerjaan mekanis yang membosankan dan memicu kelelahan mental (*fatigue*).

---

### 4.2 Aturan Emas 50% Google SRE (The 50% Rule)
- **Maksimal 50% Waktu Kerja SRE** boleh dihabiskan untuk tugas operasional harian (menangani tiket on-call, respons insiden, rapat koordinasi, dan mitigasi toil).
- **Minimal 50% Waktu Kerja SRE** **WAJIB** dialokasikan untuk pekerjaan rekayasa (*engineering projects*): membangun otomasi, memperbaiki arsitektur IaC OpenTofu/Ansible, memperkuat pipeline GitOps, dan menciptakan alat bantu yang membasmi toil secara permanen!

---

### 4.3 Inventarisasi Toil & Batasan Keamanan Otomasi (Automation Guardrails)

Membangun skrip otomasi untuk membasmi toil harus dibekali pagar pengaman (*guardrails*) agar skrip tersebut tidak menjadi bumerang yang merusak klaster:

| Elemen Pagar Pengaman | Risiko jika Tidak Dibatasi | Implementasi Standar SRE |
|---|---|---|
| **1. Blast Radius Control** | Skrip membersihkan disk yang salah di seluruh klaster produksi. | Batasi eksekusi hanya pada namespace atau node target tertentu yang terverifikasi (`--namespace production --selector app=payment`). |
| **2. Dry-Run / Check Mode** | Perubahan diterapkan langsung tanpa verifikasi awal. | Wajib mendukung opsi `--dry-run` atau `ansible --check` untuk memeriksa dampak sebelum mutasi nyata dieksekusi. |
| **3. Rate Limiting & Concurrency** | Skrip me-restart 100 pod sekaligus, menyebabkan pemadaman total (*self-inflicted outage*). | Batasi konkurensi: restart maksimal 1 pod atau 1 node pada satu waktu (`maxUnavailable: 1`). |
| **4. Circuit Breaker / Kill Switch** | Bot otomasi terus me-restart pod yang rusak tanpa henti (*infinite restart loop*). | Jika proses otomatis gagal 3 kali berturut-turut, sistem otomatis berhenti (*abort*), membunyikan alarm ke manusia, dan mengunci eksekusi. |
| **5. Audit Trail Permanen** | Tidak ada catatan siapa atau skrip apa yang mengubah konfigurasi server. | Setiap tindakan mutatif wajib mencatat log terstruktur ke Loki berisi: ID tindakan, nama skrip/bot, timestamp UTC, dan daftar resource yang diubah. |

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **SLO Ditetapkan 100% ("Sistem Tidak Boleh Pernah Mati")** | Manajemen bisnis tidak memahami konsep kompromi keandalan; tim dilarang merilis kode baru dan biaya infrastruktur membengkak tidak wajar. | Edukasi pemangku kepentingan menggunakan data historis: tunjukkan bahwa pihak ketiga (jaringan ISP, cloud, OS kernel) memiliki batas SLA < 100%. Sepakati SLO rasional (misal 99,9%). |
| **Error Budget Selalu Habis di Minggu Pertama Tiap Bulan** | Kualitas kode buruk, tidak ada automated smoke test di staging, atau promosi rilis dilakukan tanpa melalui *Canary Analysis*. | Terapkan **Production Feature Freeze** secara disiplin. Wajibkan *Canary Deployment* via Argo Rollouts (Fase 6) sebelum promosi penuh. |
| **Skrip Pembersih Log Otomatis Menghapus Data Database Produksi** | Skrip otomasi (*toil killer*) dibuat terburu-buru dengan perintah mentah `rm -rf /var/data/*` tanpa validasi path dan tanpa izin review. | 1. Pulihkan data dari snapshot Velero/MinIO segera!<br>2. Hapus skrip berbahaya tersebut dan terapkan aturan: seluruh skrip otomasi wajib melalui *Code Review* dan diuji di lab disposable terlebih dahulu. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. EVALUASI SLI & SISA ERROR BUDGET DI TERMINAL (PROMETHEUS API)
# ==============================================================================

# Menghitung persentase SLI Availability aktual selama 30 hari terakhir
curl -sG http://localhost:9090/api/v1/query \
  --data-urlencode 'query=100 * (sum(rate(http_requests_total{service="payment-api", status=~"2..|3..|400|404"}[30d])) / sum(rate(http_requests_total{service="payment-api"}[30d])))' | jq .

# Menghitung sisa Error Budget 30 hari dalam bentuk persentase (Target SLO: 99.9%)
curl -sG http://localhost:9090/api/v1/query \
  --data-urlencode 'query=100 * (1 - ((1 - (sum(rate(http_requests_total{service="payment-api", status=~"2..|3..|400|404"}[30d])) / sum(rate(http_requests_total{service="payment-api"}[30d])))) / 0.001))' | jq .

# ==============================================================================
# 2. AUDIT EKSEKUSI TUGAS TOIL DENGAN DRY-RUN AMAN
# ==============================================================================

# Menjalankan Ansible playbook pemeliharaan node dengan mode simulasi aman
ansible-playbook -i inventory/hosts.ini playbooks/cleanup-containers.yml --check --diff

# Membersihkan pod yang gagal (Failed/Evicted) dengan pembatasan namespace
kubectl delete pods -n production --field-selector status.phase=Failed --dry-run=client
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Sebuah layanan microservice perbankan memiliki target **SLO Availability sebesar 99,9%** dalam periode **30 hari**. Berapakah sisa waktu downtime maksimal (**Error Budget**) yang diizinkan sebelum seluruh rilis fitur baru wajib dibekukan (*Feature Freeze*)?
- A. 7 jam 12 menit
- B. **43 menit 12 detik**
- C. 0 detik (sistem dilarang mati sedikit pun)
- D. 24 jam penuh

#### Soal 2
Manakah dari aktivitas operasional harian berikut yang **PALING TEPAT** diklasifikasikan sebagai **Toil** menurut standar Google SRE?
- A. Melakukan riset arsitektur dan menulis modul OpenTofu baru untuk mendukung multi-cluster bare-metal.
- B. Menulis dokumen analisis akar masalah (*Blameless Postmortem*) setelah terjadinya insiden P1.
- C. Setiap pagi membuka terminal SSH ke 10 server secara manual untuk menghapus file log sementara yang penuh, pekerjaan yang sama diulang setiap hari dan tidak memberi nilai tambah permanen pada sistem.
- D. Merancang dashboard observabilitas baru menggunakan metode USE dan RED di Grafana.

#### Soal 3
Berdasarkan aturan emas **Google SRE 50% Rule**, bagaimana seorang SRE seharusnya mengalokasikan waktu kerjanya?
- A. 100% waktu untuk membalas tiket keluhan pengguna dan berjaga di depan layar monitor.
- B. Maksimal 50% waktu untuk pekerjaan operasional/on-call/toil, dan minimal 50% waktu dialokasikan untuk pekerjaan rekayasa (*engineering*) seperti membangun otomasi, perbaikan arsitektur, dan pengembangan sistem.
- C. 90% waktu untuk coding aplikasi bisnis, dan 10% untuk infrastruktur.
- D. Waktu kerja sepenuhnya ditentukan oleh jumlah insiden yang terjadi setiap hari.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Tim SRE di perusahaan Anda mengeluh kelelahan (*burnout*) karena setiap malam mereka menerima rata-rata 30 notifikasi alarm yang meminta mereka merestart pod `catalog-service` yang kehabisan memori (*OOMKilled*). 
Seorang engineer senior mengusulkan solusi:
*"Saya akan buat CronJob yang mengeksekusi `kubectl delete pod -l app=catalog-service` secara otomatis setiap 30 menit agar pod-nya selalu fresh dan kita tidak perlu bangun malam lagi!"*

**Pertanyaan & Tugas Anda:**
1. Mengapa solusi engineer senior tersebut (membuat CronJob restart setiap 30 menit) adalah bentuk **penanganan semu yang melanggengkan Toil**, bukan solusi rekayasa SRE yang sejati?
2. Sebutkan risiko teknis yang bisa terjadi jika CronJob restart paksa setiap 30 menit itu diterapkan pada aplikasi yang sedang melayani transaksi checkout pelanggan di jam sibuk!
3. Bagaimana pendekatan rekayasa SRE yang benar untuk menyelesaikan akar masalah kebocoran memori pada `catalog-service` tersebut secara permanen?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Jatah kegagalan untuk SLO 99,9% adalah $43.200 \times 0{,}001 = 43{,}2 \text{ menit}$ (43 menit 12 detik).
2. **C.** Menghapus file secara manual setiap pagi adalah contoh murni Toil: manual, repetitif, automatable, taktis, dan tidak menyelesaikan akar masalah mengapa log menumpuk.
3. **B.** Prinsip 50% menjamin bahwa peran SRE tetap sebagai tim rekayasa perangkat lunak (*software engineering*) yang memperbaiki sistem, bukan sekadar tim pemadam kebakaran operasional.

#### Solusi Soal Kasus Nyata:
1. **Mengapa Restart Periodik Bukan Solusi SRE Sejati:**
   - Tindakan tersebut hanya menyembunyikan gejala (*treating the symptom*) dan tidak menyembuhkan penyakit utamanya (*root cause*).
   - Tindakan itu menormalisasi kerusakan (*normalizing deviance*) dan membuat tim developer merasa tidak perlu memperbaiki kebocoran memori (*memory leak*) di kode program mereka karena "sudah ada skrip restart". Ini melanggar filosofi keandalan SRE.
2. **Risiko Restart Paksa Periodik:**
   - Memutus koneksi TCP/HTTP secara mendadak saat pengguna sedang melakukan pembayaran (*abrupt connection drop*).
   - Membanjiri database dengan ratusan koneksi baru secara bersamaan setiap 30 menit saat pod baru menyala (*thundering herd / connection storm*).
3. **Pendekatan Rekayasa SRE yang Benar:**
   - **Langkah 1 (Observabilitas & Bukti):** Ambil *heap profile* memori Go/JVM menggunakan *pprof* saat memori mulai naik untuk mengidentifikasi fungsi kode mana yang membocorkan objek memori.
   - **Langkah 2 (Kolaborasi dengan Developer):** Buat tiket insiden prioritas tinggi, sajikan bukti profil memori, dan gunakan Error Budget Policy: tunda rilis fitur baru pada `catalog-service` sampai tim developer menambal kebocoran memori tersebut.
   - **Langkah 3 (Pencegahan di Kubernetes):** Pasang `HorizontalPodAutoscaler (HPA)` berbasis utilisasi CPU/Memory dan konfigurasi `terminationGracePeriodSeconds` serta `preStop` hook agar pod dapat menyelesaikan transaksi aktif sebelum dihentikan dengan aman.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah memahami kontrak SLO, Error Budget, dan pembasmian Toil:
- Lanjutkan ke [02 — On-Call, Incident Response, dan Change Management](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.1-praktik-sre/02-oncall-incident-response-change-management.md) untuk mempelajari struktur tim tanggap darurat (*Incident Command System*), tata kelola rotasi on-call, manajemen perubahan, serta penulisan laporan *Blameless Postmortem*.
- Praktikkan di [LAB-01 — SLO, Error Budget, dan On-Call Simulation](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.1-praktik-sre/lab/LAB-01-slo-error-budget-oncall.md).
