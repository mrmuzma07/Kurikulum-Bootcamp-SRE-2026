# 03 — SLO, Error Budget, Burn Rate, dan Runbook

> **Ringkasan:** Menerjemahkan data telemetri teknis menjadi kontrak keandalan layanan bisnis melalui **SLI (Service Level Indicator)** dan **SLO (Service Level Objective)**, menghitung **Error Budget** secara matematis, mengimplementasikan algoritma peringatan **Multi-Window Multi-Burn-Rate (MWMBR)** standar Google SRE di PromQL, serta menyusun **Runbook Operasional** yang terstruktur untuk pemulihan insiden.

---

## 1. Konsep Dasar Keandalan SRE: SLI, SLO, SLA, dan Error Budget

### 1.1 Analogi Dunia Nyata: "Kuota Anggaran Belanja Bulanan (Error Budget) & Kecepatan Boros Uang (Burn Rate)"
Dalam dunia rekayasa perangkat lunak tradisional, sering terjadi perang dingin antara dua kubu:
- **Tim Developer** ingin merilis fitur baru setiap jam agar bisnis cepat tumbuh.
- **Tim Operasi/Infra** ingin melarang perubahan apa pun agar server tidak pernah rusak (*100% uptime*).

**Site Reliability Engineering (SRE)** mendamaikan konflik ini dengan satu kesepakatan matematis bernama **SLO dan Error Budget**:
1. **Mustahil Mencapai 100% Keandalan:** Tidak ada sistem di bumi yang 100% bebas gangguan. Mengejar 100% berarti menghentikan seluruh inovasi produk.
2. **Menetapkan Target yang Masuk Akal (SLO = 99,9%):** Bisnis, Developer, dan SRE sepakat bahwa dalam 30 hari, minimal **99,9%** transaksi pembayaran harus berhasil.
3. **Jatah Boleh Gagal (*Error Budget* = 0,1%):** Selisih `100% - 99,9% = 0,1%` adalah **"Tabungan Kuota Kegagalan"** tim selama 30 hari (setara dengan **43,2 menit downtime** atau **1.000 transaksi gagal dari 1 juta transaksi**).
   - **Selama Error Budget masih tersisa banyak:** Tim Developer bebas merilis fitur baru dan bereksperimen secepat mungkin!
   - **Jika Error Budget habis (`0% remaining`):** Seluruh rilis fitur baru **DIBEKUKAN (*Feature Freeze*)**! Seluruh waktu kerja Developer dialihkan 100% untuk memperbaiki bug keandalan, menambah automated test, dan merapikan arsitektur bersama SRE sampai kuota pulih kembali.

---

### 1.2 Perbedaan Tegas: SLI vs SLO vs SLA

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HIERARKI KONTRAK KEANDALAN LAYANAN                       │
│                                                                             │
│  1. SLI (Service Level Indicator) ──> "Angka Aktual di Alat Ukur"           │
│     Rumus: (Jumlah Kejadian Sukses ÷ Jumlah Total Kejadian Valid) × 100%    │
│     Contoh: Bulan ini terukur 99,92% request HTTP selesai < 300ms.          │
│                                      │                                      │
│                                      ▼                                      │
│  2. SLO (Service Level Objective) ──> "Target Internal Tim Engineering"     │
│     Target yang harus dijaga oleh SRE & Developer: >= 99,90% (dalam 30 hari)│
│     Jika tembus di bawah 99,90% ──> Sirene Internal Berbunyi & Freeze Fitur!│
│                                      │                                      │
│                                      ▼                                      │
│  3. SLA (Service Level Agreement) ──> "Kontrak Hukum & Denda Finansial"     │
│     Janji resmi ke pelanggan/klien eksternal: >= 99,50%                     │
│     (Selalu dibuat LEBIH LONGGAR dari SLO agar SRE sempat memperbaiki       │
│      masalah sebelum perusahaan terkena penalti denda uang!)                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Tabel Konversi Target SLO ke Jatah Waktu Downtime (Error Budget 30 Hari)

| Target SLO (%) | Error Budget (%) | Jatah Downtime per Bulan (30 Hari) | Jatah Downtime per Tahun (365 Hari) |
|---|---|---|---|
| **99,0%** (*Two Nines*) | `1,00%` | **7 jam 12 menit** | 3 hari 15 jam |
| **99,5%** | `0,50%` | **3 jam 36 menit** | 1 hari 19 jam |
| **99,9%** (*Three Nines*) | `0,10%` | **43 menit 12 detik** | 8 jam 45 menit |
| **99,95%** | `0,05%` | **21 menit 36 detik** | 4 jam 22 menit |
| **99,99%** (*Four Nines*) | `0,01%` | **4 menit 19 detik** | 52 menit 35 detik |

> [!IMPORTANT]
> **Aturan Kejujuran SLI (*Valid Events Denominator*):**
> Saat menghitung SLI Availability:
> $$\text{SLI} = \frac{\text{Request Sukses (HTTP 2xx, 3xx, 4xx kec. 429)}}{\text{Total Request Valid yang Masuk}}$$
> Mengapa error `HTTP 400 Bad Request` atau `404 Not Found` akibat kesalahan ketik pengguna tidak dihitung sebagai kegagalan server? Karena server bekerja dengan benar menolak input yang salah. Namun, **`HTTP 5xx` (Server Error) dan `HTTP 429` (Too Many Requests / Server Overload) WAJIB dihitung sebagai kegagalan!** Jangan pernah memanipulasi rumus penyebut untuk menyembunyikan kegagalan sistem!

---

## 2. Matematika Burn Rate & Multi-Window Alerting

### 2.1 Apa Itu Burn Rate (Laju Pembakaran Error Budget)?
**Burn Rate** mengukur seberapa cepat layanan Anda menghabiskan jatah Error Budget 30 hari jika dibandingkan dengan laju normal:

$$\text{Burn Rate} = \frac{\text{Error Rate Aktual Saat Ini}}{\text{Error Budget (} 1 - \text{SLO )}}$$

Mari kita ambil contoh layanan dengan **SLO = 99,9%** (sehingga **Error Budget = `0,001` atau `0,1%`**):
- **Burn Rate = `1x`:** Error rate saat ini adalah `0,1%`. Jika dibiarkan terus, tabungan Error Budget akan habis tepat dalam **30 hari (720 jam)**.
- **Burn Rate = `6x`:** Error rate saat ini adalah `0,6%`. Tabungan 30 hari akan ludes dalam **`720 ÷ 6 = 120 jam (5 hari)`**.
- **Burn Rate = `14.4x`:** Error rate saat ini adalah `1,44%`. Tabungan 30 hari akan ludes dalam **`720 ÷ 14.4 = 50 jam`** (artinya dalam **1 jam saja**, **2%** dari total kuota sebulan langsung hangus terbakar!).
- **Burn Rate = `1000x` (Pemadaman Total 100% Error):** Seluruh tabungan 30 hari ludes habis hanya dalam **43,2 menit**!

---

### 2.2 Mengapa Menggunakan "Multi-Window, Multi-Burn-Rate" (Jendela Panjang + Jendela Pendek)?

Jika Anda hanya memasang alert pada jendela panjang `rate(errors[1h]) > (14.4 * 0.001)`:
- **Masalah Reset Lambat:** Bayangkan terjadi insiden selama 10 menit lalu Anda sudah berhasil melakukan *rollback* dan aplikasi sudah sembuh 100%. Karena angka error 10 menit tadi masih tersimpan di dalam rata-rata `[1h]`, **sirine alert akan terus berbunyi selama 50 menit ke depan meskipun masalahnya sudah selesai!**

**Solusi Standar Google SRE:** Gabungkan **Jendela Panjang (`1h`)** DAN **Jendela Pendek (`5m` = 1/12 dari jendela panjang)** dengan operator logika **`and`**:
- Alert hanya berbunyi jika selama **1 jam terakhir** budget terbakar cepat **DAN** selama **5 menit terakhir** kebocoran tersebut **masih sedang berlangsung saat ini!** Begitu Anda melakukan *rollback*, dalam 5 menit jendela pendek langsung turun ke `0` dan alert langsung otomatis berhenti (`Resolved`)!

| Tingkat Keparahan (Severity) | Burn Rate | Persen Budget Hangus | Jendela Panjang (Long Window) | Jendela Pendek (Short Window) | Tindakan Operasional |
|---|---|---|---|---|---|
| **Critical (Page On-Call Cepat)** | **`14.4x`** | 2% budget habis dalam 1 jam | `1h` | `5m` | Bangunkan On-Call Engineer sekarang juga! |
| **Critical (Page On-Call Menengah)**| **`6.0x`** | 5% budget habis dalam 6 jam | `6h` | `30m` | Panggil On-Call Engineer (kebocoran signifikan). |
| **Warning (Buat Tiket Kerja)** | **`1.0x`** | 10% budget habis dalam 3 hari| `3d` | `6h` | Kirim notifikasi ke jam kerja esok pagi (tidak perlu membangunkan orang tidur). |

---

### 2.3 Implementasi Lengkap PromQL Recording & Alerting Rules SLO (`slo-rules.yaml`)

Berikut adalah konfigurasi siap pakai untuk **SLO 99,9% (Error Budget = `0.001`)** pada `payment-api`:

```yaml
groups:
  # ============================================================================
  # 1. RECORDING RULES RASIO ERROR MULTI-WINDOW (5m, 30m, 1h, 6h)
  # ============================================================================
  - name: sre_slo_payment_api_error_ratios
    interval: 30s
    rules:
      - record: service:http_requests_error_ratio:rate5m
        expr: |
          sum by (sre_environment, service) (rate(http_requests_total{service="payment-api", status=~"5.."}[5m]))
          /
          sum by (sre_environment, service) (rate(http_requests_total{service="payment-api"}[5m]))

      - record: service:http_requests_error_ratio:rate30m
        expr: |
          sum by (sre_environment, service) (rate(http_requests_total{service="payment-api", status=~"5.."}[30m]))
          /
          sum by (sre_environment, service) (rate(http_requests_total{service="payment-api"}[30m]))

      - record: service:http_requests_error_ratio:rate1h
        expr: |
          sum by (sre_environment, service) (rate(http_requests_total{service="payment-api", status=~"5.."}[1h]))
          /
          sum by (sre_environment, service) (rate(http_requests_total{service="payment-api"}[1h]))

      - record: service:http_requests_error_ratio:rate6h
        expr: |
          sum by (sre_environment, service) (rate(http_requests_total{service="payment-api", status=~"5.."}[6h]))
          /
          sum by (sre_environment, service) (rate(http_requests_total{service="payment-api"}[6h]))

  # ============================================================================
  # 2. MULTI-WINDOW MULTI-BURN-RATE ALERTING RULES (Target SLO: 99.9% / Budget: 0.001)
  # ============================================================================
  - name: sre_slo_payment_api_burn_alerts
    interval: 30s
    rules:
      # TIER 1: Fast Burn (14.4x pada 1h & 5m) OR Medium-Fast Burn (6x pada 6h & 30m)
      - alert: PaymentApiSLOErrorBudgetFastBurn
        expr: |
          (
            service:http_requests_error_ratio:rate1h{sre_environment="production", service="payment-api"} > (14.4 * 0.001)
            and
            service:http_requests_error_ratio:rate5m{sre_environment="production", service="payment-api"} > (14.4 * 0.001)
          )
          or
          (
            service:http_requests_error_ratio:rate6h{sre_environment="production", service="payment-api"} > (6.0 * 0.001)
            and
            service:http_requests_error_ratio:rate30m{sre_environment="production", service="payment-api"} > (6.0 * 0.001)
          )
        for: 2m
        labels:
          severity: critical
          sre_owner: payment-squad
          slo_tier: tier-1
        annotations:
          summary: "CRITICAL: Error Budget payment-api terbakar >14.4x / >6x kecepatan normal!"
          description: "Layanan {{ $labels.service }} di {{ $labels.sre_environment }} mengonsumsi Error Budget 99.9% dengan laju kritis. Segera lakukan triase dan rollback jika berkaitan dengan rilis baru!"
          dashboard_url: "https://grafana.internal.example.com/d/sre-red-overview-v1?var-env=production"
          runbook_url: "https://git.internal.example.com/sre/runbooks/-/blob/main/slo/PaymentApiFastBurn.md"
```

---

## 3. Standar Penyusunan Runbook Operasional SRE

Sebuah alert yang berbunyi pada pukul 03:00 pagi **tidak ada gunanya** jika engineer yang terbangun dari tidur harus menebak-nebak perintah apa yang harus diketik. Setiap alert produksi wajib memiliki dokumen **Runbook** yang mengikuti struktur kontrak 7 bagian berikut:

### 3.1 Template Runbook Produksi (`runbooks/slo/PaymentApiFastBurn.md`)

```markdown
# RUNBOOK: PaymentApiSLOErrorBudgetFastBurn

## 1. Metadata & Eskalasi
- **Pemilik Layanan (Owner):** `payment-squad` (`#oncall-payment`)
- **Dampak Bisnis (Impact):** Pelanggan gagal menyelesaikan pembayaran (Lebih dari 1,44% transaksi gagal). Jika dibiarkan 50 jam, kuota SLO bulanan habis 100%.
- **Jalur Eskalasi:** Jika dalam 15 menit belum termitigasi, hubungi Incident Commander & Database Lead.

## 2. Langkah Triase 3 Menit Pertama (Segitiga Observabilitas)
1. **Periksa Dashboard RED:** Buka [Dashboard RED Payment](https://grafana.internal.example.com/d/sre-red-overview-v1). Apakah lonjakan error dimulai tepat setelah waktu deployment baru?
2. **Periksa Riwayat Deployment ArgoCD / Rollouts:**
   ```bash
   kubectl argo rollouts status payment-api -n production
   argocd app history prod-payment-api
   ```
3. **Inspeksi Trace & Log Error (Tempo & Loki):**
   - Klik titik *Exemplar* pada grafik latensi/error untuk membuka span yang gagal di **Tempo**.
   - Atau jalankan kueri LogQL cepat:
     ```logql
     {namespace="production", service="payment-api"} |= "error" | json | status >= 500
     ```

## 3. Pohon Keputusan Mitigasi (Mitigation vs Rollback)
- **Kondisi A — Masalah muncul setelah rilis kode/konfigurasi baru (< 60 menit lalu):**
  - **JANGAN MENDIAGNOSA KODE LAMA-LAMA!** Segera batalkan rollout atau lakukan rollback:
    ```bash
    kubectl argo rollouts abort payment-api -n production
    kubectl argo rollouts undo payment-api -n production
    ```
  - Lanjutkan dengan membuat *Emergency Revert MR* di repositori GitOps.
- **Kondisi B — Tidak ada rilis baru, tetapi terjadi saturasi resource (OOMKilled / CPU Throttling):**
  - Cek status pod: `kubectl get pods -n production -l app=payment-api`
  - Cek event: `kubectl describe pod -n production -l app=payment-api`
- **Kondisi C — Dependency Eksternal (Payment Gateway Bank / Database) Down:**
  - Aktifkan *Circuit Breaker* atau alihkan rute pembayaran ke gateway cadangan melalui Feature Flag GitOps.

## 4. Peringatan Kritis Data & Side Effects (Data Recovery Caveat)
- **PERHATIAN:** Melakukan rollback pod `payment-api` **TIDAK** membatalkan migrasi skema database yang sudah terlanjur berjalan dan **TIDAK** melakukan *refund* otomatis untuk transaksi yang menggantung (*pending settlement*). Koordinasikan rekonsiliasi buku kas dengan tim Finance setelah trafik stabil.

## 5. Stop Condition (Kapan Harus Berhenti & Eskalasi)
- **BERHENTI** melakukan perubahan manual jika rollback pertama gagal menurunkan error rate dalam 5 menit. Jangan pernah menghapus PVC atau memodifikasi tabel database produksi secara manual tanpa persetujuan Incident Commander!

## 6. Verifikasi Pasca-Mitigasi (Post-Check Evidence)
- [ ] Rasio error 5 menit (`service:http_requests_error_ratio:rate5m`) turun kembali di bawah `0.001` (< 0,1%).
- [ ] Status alert `PaymentApiSLOErrorBudgetFastBurn` kembali ke `Normal / Resolved`.
- [ ] Buat dokumen kerangka **Blameless Postmortem** dalam waktu maksimal 24 jam.
```

---

## 4. Integrasi SLO sebagai Gerbang Promosi Rilis (SLO Promotion Gate)

Menghubungkan kembali ke **Fase 6 (GitOps & Progressive Delivery)**:
- Sebuah rilis aplikasi di Staging atau Canary **tidak boleh dipromosikan ke 100% Production** hanya karena status ArgoCD `Synced` dan `Healthy`!
- **Gerbang Promosi Berbasis SLO (*SLO Gate*):**
  1. Apakah selama jendela observasi canary (misal 30 menit), *Burn Rate* berada di bawah `1.0x`?
  2. Apakah sisa **Error Budget 30 hari** layanan tersebut saat ini masih di atas `0%`?
  - Jika kedua syarat terpenuhi disertai bukti rekaman kueri Mimir, barulah promosi rilis disetujui!

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Alert Burn-Rate Terlalu Sering Berbunyi di Malam Hari Saat Trafik Sepi (False Positive)** | Pada pukul 03:00 pagi hanya ada **2 request** yang masuk dalam 5 menit, dan kebetulan **1 request gagal** ➔ Error rate terhitung `1 / 2 = 50%` (Burn rate `500x`!). | Tambahkan syarat **Minimum Traffic Threshold** pada ekspresi alert (misal `and sum(rate(http_requests_total[5m])) > 1.0`), atau gunakan generator trafik sintetis (*blackbox/synthetic probe*) agar jumlah sampel minimum selalu terpenuhi. |
| **Error Budget Terkuras Habis di Akhir Bulan Tanpa Pernah Ada Alert Critical yang Berbunyi** | Terjadi kebocoran error kecil tapi konstan (misal Burn Rate `1.5x` atau `2x` terus-menerus selama 20 hari) yang berada di bawah ambang batas Fast Burn (`14.4x` dan `6x`). | Pastikan Anda memasang **Tier-2 / Slow Burn Alert (Warning Ticket)** pada Burn Rate `1.0x` – `2.0x` dengan jendela evaluasi panjang (`1d` / `3d`) agar kebocoran lambat terdeteksi sejak dini. |
| **Runbook Tidak Bisa Diikuti Saat Insiden Nyata** | Perintah di dalam runbook sudah usang (nama namespace/deployment berubah) atau tidak pernah diuji. | Uji coba setiap runbook secara berkala pada sesi **Game Day / Failure Injection** di lingkungan lab/staging (Fase 8 & Fase 9). |

---

## 6. Cheatsheet & Ringkasan Rumus Cepat SLO SRE

```text
# ==============================================================================
# RUMUS CEPAT KALKULASI ERROR BUDGET & BURN RATE
# ==============================================================================

1. Error Budget Ratio  = 1 - (SLO_Target_Percentage / 100)
   Contoh SLO 99.9%    = 1 - 0.999 = 0.001 (0.1%)

2. Jatah Waktu Downtime (30 Hari / 43.200 Menit)
   = 43.200 menit × Error Budget Ratio
   Contoh SLO 99.9%    = 43.200 × 0.001 = 43,2 menit / bulan

3. Error Rate Threshold untuk Burn Rate N kali lipat:
   Threshold = Burn_Rate × Error_Budget_Ratio
   Contoh Burn Rate 14.4x pada SLO 99.9% = 14.4 × 0.001 = 0.0144 (1,44% error rate)

4. Waktu Habisnya Seluruh Budget 30 Hari (720 Jam) pada Burn Rate N:
   Waktu Habis (Jam) = 720 jam ÷ Burn_Rate
   Contoh pada Burn Rate 14.4x = 720 ÷ 14.4 = 50 jam!

5. Rumus Sisa Error Budget 30 Hari (%) di PromQL:
   100 * (
     1 - (
       service:http_requests_error_ratio:rate30d
       /
       0.001
     )
   )
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Sebuah layanan pembayaran memiliki target **SLO Availability sebesar 99,9%** dalam jendela waktu **30 hari (43.200 menit)**. Berapakah total jatah **Error Budget** layanan tersebut, dan berapa ambang batas rasio error (*error rate threshold*) yang memicu **Burn Rate 14.4x**?
- A. Error Budget = 10% (4.320 menit); Ambang batas Burn Rate 14.4x = 50% error.
- B. Error Budget = **0,1% (43,2 menit)**; Ambang batas Burn Rate 14.4x = **`14.4 × 0,1% = 1,44%` error rate**.
- C. Error Budget = 0 menit (tidak boleh ada error sama sekali); Ambang batas = 0%.
- D. Error Budget = 7 hari penuh.

#### Soal 2
Apa alasan teknis utama penggunaan pendekatan **Multi-Window** (menggabungkan jendela panjang `1h` **DAN** jendela pendek `5m`) pada aturan peringatan Burn Rate dibandingkan hanya menggunakan jendela `1h` saja?
- A. Agar kueri PromQL terlihat lebih panjang dan rumit di layar monitor.
- B. Jendela `1h` memastikan kebocoran error budget benar-benar signifikan secara statistik, sedangkan jendela pendek `5m` memastikan bahwa **insiden masih sedang berlangsung saat ini**—sehingga begitu masalah berhasil diperbaiki (*rollback*), alert akan berhenti berbunyi dalam waktu 5 menit tanpa harus menunggu 1 jam penuh untuk reset!
- C. Untuk menghapus data yang berusia lebih dari 5 menit dari Mimir.
- D. Karena Prometheus tidak bisa menghitung jendela waktu 1 jam tanpa bantuan jendela 5 menit.

#### Soal 3
Mengapa nilai target **SLA (Service Level Agreement)** yang dijanjikan secara hukum kepada pelanggan eksternal hampir selalu ditetapkan **lebih longgar** (misal `99,5%`) dibandingkan target **SLO (Service Level Objective)** internal tim engineering (misal `99,9%`)?
- A. Karena tim hukum tidak menyukai angka desimal.
- B. Untuk memberikan **sabuk pengaman peringatan dini (*safety buffer*)** bagi tim SRE; ketika keandalan turun menyentuh `99,85%` (melanggar SLO internal `99,9%`), tim SRE langsung bergerak memulihkan sistem dan membekukan rilis fitur berisiko **jauh sebelum** angka keandalan merosot menyentuh batas penalti finansial/hukum SLA (`99,5%`).
- C. Agar tim developer tidak perlu menulis unit test.
- D. Karena SLA hanya berlaku untuk server yang dimatikan.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Tim Anda mengelola layanan `ticket-booking-api` dengan kontrak **SLO Availability = 99,95%** per 30 hari.
Pada tanggal 10 bulan berjalan, terjadi insiden kegagalan migrasi database selama **25 menit penuh (100% transaksi gagal total selama 25 menit)** sebelum tim berhasil melakukan pemulihan.

Keesokan paginya (tanggal 11), Manajer Produk meminta tim untuk segera melakukan *deploy* fitur eksperimental baru yang belum diuji beban (*load test*) langsung ke cluster produksi karena ingin mengejar target promosi pemasaran.

**Pertanyaan & Tugas Anda sebagai Lead SRE:**
1. Hitunglah secara matematis: berapa menitkah total jatah **Error Budget 30 hari** untuk layanan dengan **SLO 99,95%** tersebut?
2. Setelah mengalami pemadaman total (100% gagal) selama **25 menit** pada tanggal 10, berapa sisa Error Budget layanan tersebut untuk sisa 20 hari ke depan?
3. Berdasarkan kebijakan **Error Budget Policy SRE**, keputusan apa yang harus Anda ambil terhadap permintaan Manajer Produk untuk merilis fitur eksperimental yang belum diuji tersebut, dan apa langkah konstruktif yang harus dikerjakan tim terlebih dahulu?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Dengan SLO `99,9%`, budget kegagalan adalah `100% - 99,9% = 0,1%` (`0.001`). Dari 43.200 menit dalam sebulan, `0.001 × 43.200 = 43,2 menit`. Laju pembakaran `14.4x` berarti laju error saat ini mencapai `14.4 × 0.001 = 0.0144` (`1,44%`).
2. **B.** Tanpa jendela pendek (`5m`), memori dari insiden singkat namun parah akan terus bertahan di dalam rata-rata jendela `1h` selama hampir satu jam setelah sistem pulih (*long reset latency*).
3. **B.** SLO internal berfungsi sebagai pagar pengaman awal agar organisasi sempat bereaksi dan memperbaiki sistem sebelum melanggar janji komersial SLA kepada pelanggan.

#### Solusi Soal Kasus Nyata:
1. **Perhitungan Jatah Error Budget 30 Hari (SLO 99,95%):**
   - Persentase Error Budget = `100% - 99,95% = 0,05%` (`0.0005`).
   - Total menit dalam 30 hari = `30 × 24 × 60 = 43.200 menit`.
   - Jatah Waktu Downtime Maksimal = `43.200 menit × 0.0005 = 21,6 menit` (21 menit 36 detik) untuk 1 bulan penuh!
2. **Sisa Error Budget Setelah Insiden 25 Menit:**
   - Insiden 25 menit downtime telah mengonsumsi **`25 ÷ 21,6 = 115,7%`** dari total anggaran sebulan!
   - Artinya, **Error Budget bulan ini sudah HABIS TOTAL (Defisit `-3,4 menit` / `0% tersisa`)** padahal bulan baru berjalan 10 hari!
3. **Keputusan Kebijakan Error Budget (Error Budget Governance):**
   - **Tunda / Tolak Rilis Fitur Eksperimental Tanpa Uji:** Sesuai kesepakatan Error Budget Policy, ketika Error Budget terkuras habis (`<= 0%`), diberlakukan **Production Feature Freeze** untuk perubahan berisiko tinggi (kecuali *patch* keamanan P0 atau perbaikan keandalan).
   - **Langkah Konstruktif:** Selesaikan dokumen **Blameless Postmortem** dari insiden migrasi database kemarin, implementasikan *Action Items* pencegahannya (seperti pengujian migrasi otomatis di CI Staging dan penerapan *Canary Analysis Gate*), serta jalankan *Load Testing* di environment Staging terlebih dahulu sebelum meminta pengecualian (*waiver*) kepada pimpinan eksekutif bisnis.

---

## 8. Langkah Selanjutnya & Kaitan Antar Fase

Selamat! Anda telah menyelesaikan seluruh rangkaian teori **Fase 7 — Full-Stack Observability** (Prometheus, Grafana Alloy, Mimir, Loki, Tempo, Grafana USE/RED, Unified Alerting, hingga SLO & Multi-Window Burn Rate).

Langkah Anda berikutnya:
- Selesaikan praktik laboratorium di [LAB-02 — Alert Firing, Notification, dan Failure Injection](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.5-grafana-alerting-slo/lab/LAB-02-alert-firing-notification-failure-injection.md).
- Lanjutkan perjalanan Anda ke [Fase 8 — Praktik Operasional SRE, Incident Command, & Disaster Recovery On-Premises](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/README.md) untuk mempraktikkan manajemen insiden nyata, penulisan Blameless Postmortem, serta pencadangan klaster penuh menggunakan Velero dan snapshot etcd!
