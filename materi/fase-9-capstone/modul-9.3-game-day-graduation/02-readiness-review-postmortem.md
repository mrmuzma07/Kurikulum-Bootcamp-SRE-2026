# 9.3.2 — Readiness Review dan Postmortem

> **Ringkasan:** Memahami proses evaluasi kelulusan akhir (**Capstone Graduation Review**), menyusun dan mempertahankan paket bukti audit (**The 8-Domain Evidence Packet**), memahami kriteria penetapan keputusan dewan penguji (**Ready, Conditional, atau Not Ready**), serta menyajikan laporan investigasi pasca-insiden (**Blameless Postmortem**) berstandar enterprise.

---

## 1. Filosofi Sidang Kelulusan SRE (Graduation Review)

### 1.1 Analogi Dunia Nyata: "Sertifikasi Kelaikan Terbang Pesawat Komersial & Sidang Ujian Rekayasa"
Bayangkan sebuah pabrik pesawat yang baru selesai merakit tipe pesawat jet baru:
- Manajemen tidak bisa datang ke kantor Otoritas Penerbangan Sipil (FAA / Kemenhub) lalu berkata: *"Pesawat kami sudah selesai dirakit dan terlihat sangat keren, tolong izinkan kami langsung terbang membawa 300 penumpang besok pagi!"*.
- Otoritas Penerbangan mewajibkan **Sidang Kelaikan Terbang (Airworthiness Certification)**:
  1. Mana rekaman uji pembebanan sayap ekstrem (*Stress Test Evidence*)?
  2. Mana bukti rekaman uji mesin mati satu saat lepas landas (*Failure Mode Evidence*)?
  3. Mana dokumen sertifikasi pilot dan buku panduan darurat (*Approved Runbooks*)?
  4. Mana bukti bahwa jika listrik pesawat mati total, instrumen cadangan mekanik masih bekerja (*Disaster Recovery*)?

Ujian **Capstone Bootcamp SRE 2026** mengadopsi standar kedisiplinan yang sama. Anda tidak dinilai dari seberapa cepat Anda mengetik perintah di terminal atau seberapa banyak tutorial yang Anda baca. Anda dinilai dari **kemampuan menyajikan Rantai Bukti Digital (*Immutable Evidence Packet*) yang membuktikan bahwa platform on-premises yang Anda bangun benar-benar andal, aman, terotomasi, dan siap dioperasikan di dunia nyata!**

---

### 1.2 Delapan Domain Penilaian Kelulusan (The 8 Evaluation Domains)

Dewan penguji SRE mengevaluasi kelulusan Anda berdasarkan bukti nyata pada **8 Domain Utama**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DELAPAN DOMAIN PENILAIAN CAPSTONE SRE 2026                  │
├──────────────────────────┬──────────────────────────────────────────────────┤
│ Domain Penilaian         │ Bukti Minimum (*Minimum Evidence*) yang Wajib Ada│
├──────────────────────────┼──────────────────────────────────────────────────┤
│ **1. Architecture**      │ Diagram *As-Built* vs *Desired-State*, matriks   │
│                          │ kepemilikan 8 lapisan, dan SOP *Break-Glass*.    │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ **2. Rebuild**           │ Rekaman *Cold-Start Rebuild* total dari reposi-  │
│                          │ tori Git (OpenTofu + Ansible k3s) tanpa manual.  │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ **3. Delivery**          │ Pipeline GitLab CI buildx multi-arch, immutable  │
│                          │ digest sha256, Trivy report, ArgoCD GitOps sync. │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ **4. Exposure**          │ MetalLB L2 mode aktif, pool IP terisolasi tanpa  │
│                          │ tabrakan ARP, TLS Ingress termination bekerja.   │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ **5. Observability**     │ Pipa 3 pilar: Alloy ➔ Mimir, Loki, Tempo. Kore- │
│                          │ lasi Exemplars & Trace-to-Logs terbukti aktif.   │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ **6. Reliability**       │ Kontrak SLO 99,9%, perhitungan Error Budget 30d, │
│                          │ dan evaluasi alert Multi-Window Multi-Burn-Rate. │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ **7. Recovery**          │ Pencadangan 3-lapis Velero/etcd, bukti uji restore│
│                          │ terisolasi, serta pengukuran empiris RPO & RTO.  │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ **8. Operations**        │ Minimal 4 Runbook produksi teruji, lini masa     │
│                          │ Game Day UTC, dan dokumen Blameless Postmortem.  │
└──────────────────────────┴──────────────────────────────────────────────────┘
```

---

## 2. Kriteria Keputusan Dewan Penguji (The Graduation Decision)

Setelah meninjau paket bukti dan menguji sistem kandidat secara langsung, dewan penguji mengeluarkan **satu dari tiga keputusan resmi**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│               TIGA STATUS KEPUTUSAN KELULUSAN AKHIR CAPSTONE                │
├─────────────────┬───────────────────────────────────────────────────────────┤
│ Status Keputusan│ Kriteria Evaluasi & Konsekuensi Operasional               │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ **1. READY**    │ **LULUS PENUH (Certified Production SRE).**               │
│    (Lulus)      │ Seluruh 8 domain memiliki bukti empiris lengkap, tidak ada│
│                 │ pelanggaran keamanan, rollback < 5 menit terbukti, dan    │
│                 │ platform siap diserahkan ke tim operasional produksi.     │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ **2. CONDITIONAL│ **LULUS BERSYARAT (Remediation Required).**               │
│    (Bersyarat)  │ Fondasi utama bekerja, tetapi ditemukan celah minor yang  │
│                 │ tidak fatal (misal: 1 runbook belum diperbarui atau kurva │
│                 │ retensi trace belum optimal). Kandidat diberi waktu 7–14 │
│                 │ hari untuk melengkapi bukti tanpa perlu mengulang ujian.  │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ **3. NOT READY**│ **TIDAK LULUS (Re-take Required).**                       │
│    (Gagal)      │ Ditemukan bukti palsu, kredensial rahasia bocor di Git,   │
│                 │ proses restore gagal total, pelanggaran prinsip GitOps    │
│                 │ (modifikasi manual produksi), atau klaster gagal rebuild. │
└─────────────────┴───────────────────────────────────────────────────────────┘
```

> [!WARNING]
> **Kebenaran Mutlak SRE: Status Hijau di UI Bukan Bukti Kesiapan!**
> Dewan penguji **tidak menerima klaim kelulusan** yang hanya menunjukkan:
> - Pipeline GitLab CI berwarna hijau.
> - Dashboard ArgoCD berstatus `Synced` dan `Healthy`.
> - Seluruh Pod berstatus `Running`.
> - Backup Velero berstatus `Completed`.
> Seluruh indikator visual di atas **hanya instrumen bantu**. Bukti kesiapan sejati dibuktikan melalui: **Apakah sistem berhasil dipulihkan saat sengaja dihancurkan? Apakah pesan notifikasi benar-benar sampai ke ponsel saat disuntikkan kegagalan? Dan apakah data transaksi tetap konsisten setelah di-restore?**

---

## 3. Kontrak Paket Bukti Kelulusan (`capstone-evidence-packet.yaml`)

Berikut adalah skema resmi dokumen pembuktian kelulusan yang diserahkan dan ditandatangani oleh dewan penguji:

```yaml
schema_version: "sre.bootcamp.capstone/graduation-packet/v1.0"
capstone_candidate: "Nama Peserta / Candidate ID"
evaluation_timestamp_utc: "2026-10-18T11:00:00Z"
platform_target: "MacBook Apple Silicon ARM64 (OrbStack) On-Prem Simulation"

# 1. Hasil Evaluasi 8 Domain Utama
evaluation_domains:
  architecture_boundary:
    status: "PASS"
    evidence_reference: "docs/architecture/as-built-topology.md"
  platform_rebuild:
    status: "PASS"
    evidence_reference: "artifacts/rebuild-run-evidence.json"
    cold_start_duration_minutes: 18
  delivery_gitops:
    status: "PASS"
    evidence_reference: "artifacts/delivery-evidence.yaml"
    rollback_duration_minutes: 3.5 # Lulus target < 5 menit!
  exposure_metallb:
    status: "PASS"
    vip_allocated: "192.168.30.201"
    arp_resolution_status: "VERIFIED_NO_COLLISION"
  observability_telemetry:
    status: "PASS"
    correlation_metrics_logs_traces: "VERIFIED_LIVE"
    mandatory_alerts_count_active: 5
  reliability_slo:
    status: "PASS"
    target_slo_percent: 99.9
    remaining_error_budget_percent: 38.5
  recovery_disaster:
    status: "PASS"
    measured_rpo_minutes: 15
    measured_rto_minutes: 28
    isolated_target_verified: true
  operations_runbooks:
    status: "PASS"
    runbooks_count_tested: 4
    postmortem_status: "APPROVED_BLAMELESS"

# 2. Daftar Celah yang Diketahui & Mitigasi (Known Gaps / Action Items)
known_gaps:
  - gap_id: "GAP-01"
    description: "Kapasitas disk partisi MinIO S3 di infra-vm tersisa 22% setelah pengujian restore"
    risk_level: "LOW"
    action_required: "Ekspansi virtual disk infra-vm dari 30 GiB menjadi 50 GiB di OpenTofu"
    owner: "Platform SRE"
    remediation_due_date_utc: "2026-10-25T00:00:00Z"
    verification_test: "tofu apply --target=orbstack_vm.infra_vm"

# 3. Keputusan Akhir Dewan Penguji
graduation_panel_decision:
  decision: "READY"
  panel_lead_signature: "Lead SRE Examiner (@chief-examiner)"
  signed_at_utc: "2026-10-18T11:30:00Z"
```

---

## 4. Standar Laporan Postmortem Kelulusan (The Graduation Postmortem)

Dalam sesi Game Day, salah satu insiden nyata yang disuntikkan oleh penguji wajib dianalisis dan didokumentasikan ke dalam laporan **Blameless Postmortem** resmi:

```markdown
# POSTMORTEM: [GD-06-INCIDENT] Pemadaman Transaksi Pembayaran Akibat Tabrakan ARP MetalLB L2

| Parameter Insiden | Detail Catatan Resmi |
|---|---|
| **Incident ID** | INC-GD-20261018-06 |
| **Status Dokumen** | **APPROVED & VERIFIED** |
| **Tanggal Kejadian** | 18 Oktober 2026 (Waktu Ujian Game Day) |
| **Tingkat Keparahan** | **P1 (Critical Outage)** |
| **Durasi Pemadaman** | 14 Menit (Waktu Deteksi: 2 Menit, Waktu Mitigasi: 12 Menit) |
| **Incident Commander** | Peserta A |
| **Operations Lead** | Peserta B |

---

## 1. Ringkasan Eksekutif (Executive Summary)
Pada pukul 09:30 UTC, instruktur Game Day menyuntikkan kegagalan konflik IP pada pool MetalLB. Lalu lintas pembayaran ke `payment.internal` terputus. Tim mengidentifikasi anomali duplikasi MAC address menggunakan `arping`, mengeksekusi mitigasi pemindahan alokasi IP Service ke IP cadangan melalui GitOps, dan memulihkan trafik dalam waktu 12 menit tanpa menyentuh switch jaringan fisik.

## 2. Garis Waktu Kronologis (UTC Timeline)
- **09:30 UTC** — Instruksi chaos injection dijalankan oleh dewan penguji.
- **09:32 UTC** — Alert PagerDuty `BlackboxProbeFailedHTTP` berbunyi ke Primary On-Call.
- **09:34 UTC** — IC mendeklarasikan Insiden P1 dan membagi peran ke Ops Lead dan Comms Lead.
- **09:38 UTC** — Ops Lead menjalankan `arping -I eth0 192.168.30.201` dan menemukan respon ganda dari dua MAC address berbeda.
- **09:41 UTC** — IC menyetujui mitigasi: ubah loadBalancerIP ke `192.168.30.205` di values GitOps.
- **09:44 UTC** — ArgoCD menyelesaikan sinkronisasi commit GitOps baru.
- **09:46 UTC** — MetalLB leader speaker mengumumkan IP baru; curl probe kembali mengembalikan status HTTP 200.
- **09:55 UTC** — Metrik stabil selama 10 menit; IC menyatakan insiden selesai.

## 3. Analisis 5 Mengapa (5 Whys Root Cause Analysis)
1. *Mengapa layanan pembayaran tidak bisa diakses?* Karena paket jaringan TCP mengalami *connection refused*.
2. *Mengapa TCP refused?* Karena paket dikirim ke interface mesin lain yang tidak mendengarkan port 80/443.
3. *Mengapa paket terkirim ke mesin lain?* Karena ada dua perangkat yang mengklaim kepemilikan IP `192.168.30.201` secara bersamaan (*ARP Collision*).
4. *Mengapa ada IP ganda di jaringan?* Karena pool IP MetalLB belum diisolasi ke dalam VLAN terproteksi khusus (*VLAN Leakage*).
5. *Mengapa belum diisolasi?* Karena konfigurasi trunking switch ToR belum menerapkan filter tag VLAN untuk subnet VIP.

## 4. Pelajaran Berharga (What Went Well vs What Went Poorly)
- **Yang Berjalan Baik:**
  - Ops Lead mematuhi Runbook RB-NET-002 dan langsung menggunakan perintah `arping` tanpa menebak-nebak.
  - Komunikasi di kanal War Room sangat tertib dan bebas dari saling menyalahkan.
- **Yang Berjalan Kurang Baik:**
  - Waktu pembuatan Merge Request GitOps darurat memakan waktu 5 menit karena koneksi remote runner sempat lambat.

## 5. Rencana Tindakan Pencegahan Permanen (Action Items — SMART)
| No | Tindakan Pencegahan Sistem | Kategori | Penanggung Jawab | Tenggat Waktu | Metode Verifikasi |
|---|---|---|---|---|---|
| **1** | Kunci alokasi subnet MetalLB ke VLAN khusus (*Dedicated VIP VLAN 35*) di OpenTofu | **Arsitektur** | `@network-sre` | 24 Okt 2026 | Uji isolasi subnet via ping |
| **2** | Tambahkan alert peringatan dini *Duplicate ARP Reply Detected* di Prometheus | **Deteksi** | `@sre-platform` | 26 Okt 2026 | Alert berbunyi di lab saat diuji |
| **3** | Sediakan script otomasi darurat untuk mempercepat pembuatan Revert MR GitOps | **Mitigasi** | `@devops-lead` | 28 Okt 2026 | Waktu revert turun < 2 menit |
```

---

## 5. Lima Gerbang Peninjauan Kelulusan (The 5 Review Gates)

Sebelum menandatangani kelulusan, dewan penguji memvalidasi **5 Gerbang Pengaman Terakhir**:
1. **Gerbang Integritas Bukti (Zero Secret Leakage Gate):** Seluruh file log, dokumen postmortem, dan rekaman evidence wajib bersih 100% dari kredensial plaintext, private key, token, dan PII nasabah.
2. **Gerbang Kemandirian Rekonstruksi (Deterministic Rebuild Gate):** Kandidat mampu menghancurkan seluruh lingkungan dan membangunnya kembali dari repositori Git tanpa campur tangan instruktur.
3. **Gerbang Kecepatan Mitigasi (Rollback Under 5-Minutes Gate):** Kandidat membuktikan pembatalan rilis cacat berlangsung di bawah 5 menit.
4. **Gerbang Integritas Data (Data Consistency Gate):** Data yang dipulihkan dari snapshot Velero/MinIO terbukti konsisten secara transaksional di database PostgreSQL.
5. **Gerbang Kepemimpinan Insiden (ICS Leadership Gate):** Kandidat mampu memimpin komunikasi darurat dengan tenang, terstruktur, dan berbudaya *Blameless*.

---

## 6. Cheatsheet & Ringkasan Perintah Penutupan Capstone

```bash
# ==============================================================================
# 1. PENGUMPULAN & SANITASI BUKTI AKHIR SEBELUM SUBMIT CAPSTONE
# ==============================================================================

# Memeriksa apakah ada file di repositori yang tidak sengaja memuat kata sandi/token
grep -rnE "(password|secret|bearer|token|private_key)" docs/ artifacts/ || echo "[OK] Clean"

# Mengompresi seluruh dokumen bukti dan laporan postmortem ke dalam paket zip kelulusan
tar -czvf capstone-graduation-evidence-$(date +%Y%m%d).tar.gz docs/ artifacts/

# ==============================================================================
# 2. PEMBERSIHAN AKHIR LINGKUNGAN DISPOSABLE PASCA-KELULUSAN (CLEANUP)
# ==============================================================================

# Menghapus klaster sandbox uji coba k3d
k3d cluster delete restore-drill

# Menghentikan VM disposable lab di OrbStack setelah seluruh bukti diverifikasi
orb stop infra-vm servers-vm
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Apa perbedaan mendasar antara hasil evaluasi **READY** dan **CONDITIONAL** pada sidang keputusan kelulusan akhir Capstone Bootcamp SRE?
- A. READY berarti peserta mendapatkan hadiah uang tunai, sedangkan CONDITIONAL tidak.
- B. **READY** berarti seluruh 8 domain telah terbukti memenuhi standar operasional produksi tanpa celah kritis, sedangkan **CONDITIONAL** berarti fondasi utama sistem sudah bekerja namun masih ada celah minor (*known gap*) yang harus diselesaikan dalam batas waktu 7–14 hari disertai komitmen tindakan perbaikan (*remediation plan*).
- C. CONDITIONAL berarti peserta dinyatakan gugur dan harus mengulang bootcamp dari minggu pertama.
- D. READY hanya diberikan kepada peserta yang memiliki komputer termahal.

#### Soal 2
Dalam penyusunan **Action Items** pada dokumen laporan pasca-insiden (*Blameless Postmortem*), kriteria apakah yang membedakan rekomendasi yang berkualitas profesional dari sekadar janji kosong?
- A. Action Item ditulis menggunakan huruf kapital semua.
- B. Action Item memenuhi kriteria **SMART** (Specific, Measurable, Actionable, Relevant, Time-bound): tindakan terarah pada perbaikan sistem/otomasi, memiliki tim penanggung jawab tunggal (*owner*), batas waktu penyelesaian pasti (*due date UTC*), dan metode pengujian verifikasi yang terukur.
- C. Action Item berisi daftar nama orang-orang yang harus dihukum potong gaji.
- D. Action Item dibuat sebanyak minimal 100 butir.

#### Soal 3
Mengapa seorang penguji menolak meluluskan kandidat yang membuktikan keberhasilan pengujian pemulihan bencana (*Disaster Recovery*) dengan cara me-restore database langsung menimpa klaster produksi yang sedang aktif?
- A. Karena proses restore membutuhkan kuota internet yang terlalu besar.
- B. Karena tindakan tersebut melanggar prinsip keselamatan operasional (*Safety First*); pengujian restore wajib dilakukan pada lingkungan terisolasi (*Isolated Disposable Target*) untuk membuktikan bahwa prosedur pemulihan dapat berjalan dari kondisi nol (*Cold-Start*) tanpa mengancam data transaksi nasabah yang sedang aktif berjalan.
- C. Karena Kubernetes tidak mengizinkan pengetikan perintah restore di siang hari.
- D. Karena database PostgreSQL otomatis rusak jika di-restore lebih dari satu kali.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Anda adalah Ketua Dewan Penguji (*Lead SRE Examiner*) pada sidang Capstone Bootcamp SRE 2026.
Seorang kandidat mempresentasikan paket bukti (*Evidence Packet*) miliknya:
- **Domain Rebuild:** Klaster k3s berhasil dibangun kembali dari OpenTofu dan Ansible dalam 15 menit.
- **Domain Delivery:** Pipeline GitLab CI multi-arch dan ArgoCD sinkron tanpa kendala.
- **Domain Observability:** Mimir, Loki, dan Tempo aktif mengumpulkan metrik, log, dan trace.
- **Domain Operations:** Runbook tersedia lengkap.

Namun, saat meninjau **Domain Recovery & Security**, tim penguji menemukan:
1. File pencadangan Velero tersimpan di bucket MinIO lokal di dalam VM yang sama dengan klaster k3s (bukan di server terpisah). Jika server fisik terbakar, data backup ikut musnah terbakar.
2. Kandidat belum sempat melakukan uji coba restore nyata (*Restore drill*) ke klaster terisolasi karena kehabisan waktu sebelum ujian.
3. Dokumen *Blameless Postmortem* Game Day kandidat sangat rapi dan lengkap dengan analisa 5 Whys.

**Pertanyaan & Tugas Anda sebagai Ketua Dewan Penguji:**
1. Berdasarkan kriteria evaluasi Capstone, apakah keputusan yang Anda berikan kepada kandidat tersebut: **READY, CONDITIONAL, atau NOT READY**? Jelaskan argumen teknis Anda!
2. Jika Anda memberikan keputusan **CONDITIONAL**, rumuskan **1 Known Gap & Action Item Remediation resmi** yang wajib diselesaikan kandidat dalam waktu 7 hari ke depan!
3. Bukti (*evidence*) spesifik apakah yang wajib diserahkan kandidat tersebut pada sesi verifikasi ulang minggu depan agar statusnya berubah menjadi **READY**?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Keputusan CONDITIONAL memberi ruang profesional bagi perbaikan celah non-fatal (*remediation grace period*) dengan pertanggungjawaban risiko yang jelas (*Risk Acceptance*).
2. **B.** Standar SMART menjamin akuntabilitas tanpa menyalahkan individu (*accountability without blame*), mengubah insiden menjadi investasi perbaikan sistem jangka panjang.
3. **B.** Menguji pemulihan langsung di klaster aktif berisiko merusak integritas transaksi yang sedang berjalan (*data corruption risk*). Lingkungan sandbox terisolasi adalah standar baku pengujian DR.

#### Solusi Soal Kasus Nyata:
1. **Keputusan Dewan Penguji: CONDITIONAL (Lulus Bersyarat)**
   - **Alasan Teknis:**
     - Seluruh fondasi utama (Rebuild, Delivery GitOps, Observabilitas, Exposure, dan Penanganan Insiden) telah terbukti berjalan dengan sangat baik dan memenuhi standar kompetensi tinggi.
     - Kelemahan pada penyimpanan backup di VM yang sama (*Single Failure Domain*) dan absennya bukti uji restore terisolasi adalah celah keamanan (*gap*), namun bukan berarti kandidat gagal total. Celah ini dapat dimitigasi dalam waktu singkat dengan memindahkan lokasi bucket MinIO ke VM terpisah (`infra-vm`) dan menjalankan uji restore terisolasi.
2. **Rumusan Known Gap & Action Item Remediation (7 Hari):**
   - **Gap ID:** `GAP-CAPSTONE-DR-01`
   - **Deskripsi:** *Penyimpanan backup Velero berada dalam satu failure domain dengan klaster k3s dan belum diverifikasi melalui uji pemulihan terisolasi.*
   - **Tingkat Risiko:** *MEDIUM*
   - **Action Item:** Pindahkan konfigurasi `BackupStorageLocation` Velero ke bucket MinIO di VM infrastruktur terpisah (`infra-vm`), jalankan 1 kali simulasi restore penuh ke klaster sandbox disposable k3d, dan catat hasil pengukuran RPO dan RTO aktual.
   - **Owner:** Kandidat SRE | **Tenggat:** 25 Oktober 2026 (7 Hari).
3. **Bukti Spesifik yang Wajib Diserahkan Minggu Depan:**
   - **Bukti 1:** File manifes konfigurasi Velero yang menunjukkan endpoint MinIO berada di IP `infra-vm` (`192.168.10.x`), bukan `servers-vm`.
   - **Bukti 2:** Dokumen rekaman `telemetry-recovery-evidence.yaml` yang mencantumkan: `isolated_target: k3d-restore-drill`, `measured_rpo_minutes: <angka>`, `measured_rto_minutes: <angka>`, dan bukti kueri `SELECT count(*) FROM orders;` pada database hasil restore yang cocok dengan data sumber!

---

## 8. Penutup Kurikulum Bootcamp SRE 2026

Selamat! Anda telah menyelesaikan seluruh rangkaian materi teori kurikulum **Bootcamp SRE 2026**:
- Dari **Fase 0 (Fondasi Linux, Jaringan, dan Git)**,
- **Fase 1–2 (Container, OrbStack ARM64, k3s, dan MetalLB)**,
- **Fase 3–5 (OpenTofu IaC, Ansible Configuration as Code, dan Helm Packaging)**,
- **Fase 6–7 (GitLab CI, ArgoCD GitOps, Progressive Delivery, dan Full-Stack Observability Alloy/Mimir/Loki/Tempo/SLO)**,
- Hingga **Fase 8–9 (Praktik Operasional SRE, Incident Command, Runbook, dan Capstone Graduation Rebuild)**.

Anda kini memiliki pola pikir, disiplin rekayasa, dan refleks otot (*muscle memory*) seorang **Site Reliability Engineer tingkat produksi** yang siap mengoperasikan, mengamankan, dan mempertahankan sistem infrastruktur fisik berskala besar dengan penuh percaya diri!
