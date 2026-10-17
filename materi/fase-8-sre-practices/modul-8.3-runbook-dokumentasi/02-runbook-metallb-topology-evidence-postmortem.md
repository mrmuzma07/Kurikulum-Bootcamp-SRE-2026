# 02 — Runbook MetalLB, Topology, Evidence, dan Postmortem

> **Ringkasan:** Menguasai panduan pemecahan masalah jaringan bare-metal (**Runbook MetalLB IP Not Responding & ARP Collision**), memelihara tiga artefak dokumentasi arsitektur sistem (**As-Built, Desired-State, dan Dependency Map**), menerapkan tata kelola bukti digital (*Evidence Chain*) dengan penyensoran data sensitif, serta menyusun laporan investigasi mendalam (**Blameless Postmortem**).

---

## 1. Runbook Produksi: MetalLB IP Not Responding & ARP Conflict

Di lingkungan bare-metal on-premises, **MetalLB** adalah jembatan vital antara jaringan fisik perusahaan dan pod Kubernetes. Ketika LoadBalancer IP tidak bisa diakses, bisnis terhenti seketika.

### 1.1 Kontrak Runbook Jaringan MetalLB

```yaml
runbook_metadata:
  runbook_id: "RB-NET-002"
  title: "MetalLB LoadBalancer VIP Unreachable & ARP Conflict"
  revision: "v1.4.0"
  last_reviewed_utc: "2026-10-01T00:00:00Z"
  expires_utc: "2027-04-01T00:00:00Z"
  sre_owner: "network-infra-team"
  escalation_channel: "#oncall-network-p1"
```

#### A. Gejala & Pemicu (Symptoms)
- **Pemicu:** Alert `BlackboxProbeFailedHTTP` atau pengguna luar tidak bisa mengakses IP LoadBalancer (`curl: (7) Failed to connect to <VIP> port 80: Connection refused` atau `No route to host`).
- **Dampak:** Seluruh lalu lintas pengguna masuk ke Ingress atau Service aplikasi terputus total.

#### B. Pemeriksaan Awal Tanpa Mutasi (Read-Only First Checks)
```bash
# 1. Periksa apakah Service Kubernetes berhasil memperoleh alokasi External-IP
kubectl get svc -n production -l app=payment-api -o wide

# 2. Periksa status kesehatan pod controller dan speaker MetalLB
kubectl get pods -n metallb-system -o wide

# 3. Periksa status pool IP dan konfigurasi L2Advertisement
kubectl get ipaddresspool,l2advertisement -n metallb-system

# 4. Uji respon ARP fisik dari mesin lain di VLAN yang sama
arping -c 3 -I eth0 <VIP_METALLB>

# 5. Periksa tabel ARP di host lokal (Cek MAC Address yang menjawab)
arp -an | grep "<VIP_METALLB>"
```

#### C. Pohon Keputusan Diagnostik MetalLB
```text
Apakah Service Mendapatkan External IP?
  ├── TIDAK (<Pending>) ──> [Masalah Alokasi IPAddressPool]
  │                           ├── Periksa apakah kapasitas IP pool sudah habis: `kubectl describe ipaddresspool`
  │                           └── Cek apakah ada kesalahan penulisan selector pada Service `spec.loadBalancerClass`.
  │
  └── YA (IP Tertera)   ──> Apakah ARP Merespons saat di-ping dari luar?
                                ├── TIDAK ──> [Masalah Speaker L2 / Switch VLAN]
                                │               ├── Cek log pod speaker MetalLB: `kubectl logs -n metallb-system -l app=metallb,component=speaker`
                                │               ├── Pastikan `strictARP: true` pada kube-proxy: `kubectl get cm -n kube-system kube-proxy -o yaml | grep strictARP`
                                │               └── Cek apakah port switch ToR berada di VLAN yang benar (VLAN 30 Workload).
                                │
                                └── YA    ──> [ARP Menjawab tapi TCP Gagal / Drop]
                                                ├── Cek apakah Pod backend aplikasi berstatus `Ready`: `kubectl get ep <SERVICE_NAME>`
                                                ├── Cek apakah NetworkPolicy memblokir trafik masuk ke namespace.
                                                └── Periksa apakah ada dua server fisik yang mengklaim IP yang sama (**IP Collision**)!
```

#### D. Kondisi Berhenti (Stop Condition) & Mitigasi Aman
- **STOP:** Jika terdeteksi konflik IP (*IP Collision*) dengan server database fisik lain di luar Kubernetes, **DILARANG** me-restart switch jaringan! Segera ubah alokasi IP Service ke range IP lain yang masih kosong di `IPAddressPool`.
- **Mitigasi Cepat (L2 Failover Force):** Jika pod speaker di node leader macet mengumumkan ARP, hapus pod speaker di node tersebut (`kubectl delete pod -n metallb-system -l component=speaker --field-selector spec.nodeName=<NODE_MACET>`) agar Kubernetes membuat pod baru dan memilih leader baru secara otomatis.

---

## 2. Tiga Artefak Dokumentasi Arsitektur Sistem

Dalam operasional SRE bare-metal yang matang, dokumentasi bukan sekadar gambar di papan tulis yang digambar 2 tahun lalu lalu dilupakan. SRE memelihara **3 artefak arsitektur terpisah**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 TIGA ARTEFAK DOKUMENTASI ARSITEKTUR SRE                     │
│                                                                             │
│  [1. AS-BUILT TOPOLOGY]                                                     │
│   Kondisi nyata yang benar-benar aktif dan terhubung di data center         │
│   saat ini (dihasilkan otomatis dari `lldpctl`, `kubectl get nodes`, dll).  │
│                                     │                                       │
│                                     ▼ (Deteksi Drift / Perbedaan)           │
│  [2. DESIRED-STATE TOPOLOGY]                                                │
│   Kondisi arsitektur yang seharusnya berlaku sesuai spesifikasi kode        │
│   di repositori OpenTofu, Ansible, dan GitOps.                              │
│                                     │                                       │
│                                     ▼                                       │
│  [3. DEPENDENCY MAP]                                                        │
│   Peta ketergantungan hulu ke hilir: Layanan Bisnis ➔ Ingress ➔ MetalLB     │
│   ➔ CoreDNS Lokal ➔ MinIO S3 ➔ Database PostgreSQL ➔ NTP Server Chrony.    │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Aturan Penanganan Architectural Drift:**
> Jika ditemukan perbedaan antara *As-Built* (kondisi aktual di lapangan) dan *Desired-State* (kode di Git): **JANGAN MENGUBAH SERVER FISIK SECARA DIAM-DIAM!** Investigasi mengapa perbedaan itu terjadi. Jika perubahan fisik tersebut adalah perbaikan darurat insiden, segera ajukan Merge Request di repositori OpenTofu/Ansible agar *Desired-State* kembali menjadi cerminan kenyataan.

---

## 3. Rantai Bukti Digital (Evidence Chain) & Aturan Redaksi Rahasia

Dalam investigasi insiden dan audit kepatuhan (seperti ISO 27001 / SOC 2), setiap klaim keandalan wajib memiliki **Rantai Bukti Digital (*Immutable Evidence Chain*)** yang tidak bisa dipalsukan.

### 3.1 Tujuh Titik Rantai Bukti Operasional
1. **Commit SHA & Approval:** Siapa yang menyetujui perubahan di GitOps MR.
2. **Pre-flight Telemetry:** Kondisi grafik metrik sebelum perubahan dieksekusi.
3. **Execution Timestamp (UTC):** Waktu presisi kapan perintah mitigasi dijalankan.
4. **Diagnostic Command Output:** Hasil pengecekan terminal yang relevan.
5. **Mitigation Outcome:** Kode status HTTP (`200 OK`) dan status pod (`Running/Ready`).
6. **SLO Verification Window:** Grafik metrik selama minimal 30 menit setelah perbaikan yang membuktikan bahwa error budget berhenti terbakar.
7. **Postmortem Sign-off:** Persetujuan dokumen evaluasi oleh Incident Commander dan Engineering Lead.

---

### 3.2 Matriks Penyensoran Data Sensitif (Evidence Redaction Matrix)

Saat melampirkan log terminal ke dalam dokumen insiden atau tiket publik:

| Kategori Data | Status dalam Laporan Bukti | Contoh Tindakan Penyensoran (Redaction) |
|---|---|---|
| **Alamat IP & Port Internal** | **DIPERBOLEHKAN** | Tetap tampilkan (`192.168.30.100:8080`) untuk konteks teknis. |
| **Commit SHA & Job ID** | **DIPERBOLEHKAN** | Tetap tampilkan (`commit: a8f1b2c...`) sebagai bukti jejak audit. |
| **Password & Token API** | **DILARANG KERAS (Hapus!)** | Ganti dengan string: `[REDACTED_PASSWORD]` atau `[REDACTED_API_TOKEN]`. |
| **Header `Authorization` / JWT** | **DILARANG KERAS (Hapus!)** | Sensor menjadi: `Bearer [REDACTED_JWT_TOKEN]`. |
| **Data Pribadi Nasabah (PII)** | **DILARANG KERAS (Hapus!)** | Sensor email, nomor KTP, nomor rekening: `customer_email=[REDACTED_PII]`. |
| **Private SSH Key / Kubeconfig** | **HARAM DITAMPILKAN!** | Cukup sebutkan nama file referensi: `kubeconfig_source: vault://k3s-prod-cert`. |

---

## 4. Standar Laporan Investigasi Insiden: Blameless Postmortem Contract

Setiap insiden dengan tingkat keparahan **P1 (Kritis)** atau **P2 (Mayor)** wajib menghasilkan dokumen evaluasi resmi yang disimpan di folder `docs/postmortems/` repositori Git.

### 4.1 Dokumen Postmortem Lengkap (`postmortem-INC-2026-10-18.md`)

```markdown
# INCIDENT POSTMORTEM: [INC-2026-10-18] Gangguan Akses Layanan Pembayaran Akibat Konflik IP Address Pool MetalLB

| Parameter Evaluasi | Detail Catatan Resmi |
|---|---|
| **Incident ID** | INC-2026-10-18-NET |
| **Status Dokumen** | **APPROVED & CLOSED** |
| **Tanggal & Waktu (UTC)**| 18 Oktober 2026, 08:20 UTC – 09:05 UTC (45 Menit) |
| **Tingkat Keparahan** | **P1 (Critical Outage)** |
| **Layanan Terdampak** | `payment-api` dan `ingress-nginx` |
| **Dampak Finansial/User** | 3.200 transaksi pembayaran gagal (Error Connection Timeout), estimasi nominal transaksi tertunda: Rp 280.000.000 |
| **Incident Commander** | Lead SRE (@lead-sre) |
| **Operations Lead** | Network SRE (@net-sre) |
| **Communications Lead** | Product Ops (@prod-ops) |

---

## 1. Ringkasan Eksekutif (Executive Summary)
Pada pukul 08:20 UTC, seluruh permintaan pembayaran dari pengguna internet ke domain `payment.example.com` mengalami kegagalan koneksi (*Connection Refused / Timeout*). Investigasi menemukan bahwa alamat IP VIP MetalLB `192.168.30.50` mengalami konflik (*IP Collision*) karena seorang teknisi kantor memasang server printer fisik baru yang dikonfigurasi secara manual dengan alamat IP statis yang sama di VLAN 30. Switch jaringan ToR mengirim sebagian paket ke printer tersebut. Insiden terselesaikan pada pukul 09:05 UTC setelah SRE mengalihkan VIP MetalLB ke IP cadangan `192.168.30.55` melalui repositori GitOps.

## 2. Garis Waktu Kronologis (Timeline UTC)
- **08:18 UTC** — Server printer fisik baru dinyalakan di ruang operasional dengan IP statis `192.168.30.50`.
- **08:20 UTC** — PagerDuty membunyikan alarm *BlackboxProbeFailedHTTP* untuk endpoint pembayaran.
- **08:22 UTC** — On-Call SRE merespons pager dan mendeklarasikan status Insiden P1 di War Room.
- **08:28 UTC** — Ops Lead menjalankan `arping` dan menemukan ada **dua MAC Address fisik berbeda** yang merespons IP `192.168.30.50`.
- **08:35 UTC** — IC menyetujui mitigasi darurat: ubah alokasi IP `payment-ingress` ke IP cadangan `192.168.30.55`.
- **08:42 UTC** — Merge Request perbaikan IP di GitOps di-merge dan ArgoCD melakukan sinkronisasi ke klaster.
- **08:45 UTC** — MetalLB speaker mengumumkan IP baru via Gratuitous ARP; trafik mulai mengalir kembali normal.
- **09:05 UTC** — Pemantauan metrik selama 20 menit menunjukkan error rate stabil di angka 0,01%; IC menutup status darurat insiden.

## 3. Faktor Berkontribusi (5 Whys Root Cause Analysis)
1. *Mengapa transaksi pembayaran gagal?* Karena paket jaringan dari internet dikirim ke server printer kantor, bukan ke ingress Kubernetes.
2. *Mengapa printer kantor menerima paket tersebut?* Karena printer dikonfigurasi dengan alamat IP `192.168.30.50` yang sama persis dengan VIP MetalLB.
3. *Mengapa ada printer di subnet yang sama dengan MetalLB?* Karena VLAN 30 sebelumnya tidak memiliki batasan isolasi fisik dari jaringan kantor umum.
4. *Mengapa IP `192.168.30.50` dipilih untuk printer?* Teknisi kantor tidak memiliki akses ke dokumen alokasi IP dan berasumsi IP tersebut tidak terpakai karena tidak merespons ping saat itu.
5. *Mengapa tidak ada pencegahan bentrok IP otomatis?* Karena belum ada sistem IPAM (*IP Address Management*) terpusat yang mengunci pool IP MetalLB dari penetapan manual perangkat lain.

## 4. Evaluasi Respons (What Went Well vs What Went Poorly)
- **Yang Berjalan Baik:**
  - Diagnosa menggunakan `arping` langsung menemukan adanya dua MAC address dalam waktu 6 menit.
  - Alur mitigasi GitOps berjalan cepat tanpa perlu menyentuh switch fisik secara manual.
- **Yang Berjalan Kurang Baik:**
  - Tim customer support terlambat 10 menit memberikan pengumuman status page karena menunggu konfirmasi teknis yang terlalu mendalam.

## 5. Rencana Tindakan Pencegahan Permanen (Action Items — SMART)
| No | Tindakan Korektif Permanen | Kategori | Tim Penanggung Jawab | Tenggat Waktu | Verifikasi Keberhasilan |
|---|---|---|---|---|---|
| **1** | Pindahkan pool MetalLB ke VLAN khusus isolasi (*VLAN 35 VIP*) yang terpisah dari perangkat kantor umum | **Pencegahan Arsitektur** | `@net-infra` | 25 Okt 2026 | Uji coba ping dari perangkat kantor diblokir firewall |
| **2** | Pasang sistem IPAM otomatis (seperti NetBox) dengan reservasi subnet terkunci untuk Kubernetes | **Otomasi Tata Kelola** | `@sre-platform` | 05 Nov 2026 | Seluruh IP MetalLB tercatat resmi di NetBox |
| **3** | Tambahkan alert deteksi dini *Duplicate ARP MAC Address* pada sistem pemantauan Prometheus | **Deteksi** | `@sre-platform` | 30 Okt 2026 | Alert berbunyi di lab saat diuji dengan IP kembar |

## 6. Keputusan Error Budget & Kebijakan
- Insiden selama 45 menit ini menghabiskan **100% dari Error Budget bulanan layanan Payment API (Defisit -1,8 menit)**.
- **Kebijakan:** Sesuai Error Budget Policy, diberlakukan **Production Feature Freeze selama 7 hari** ke depan untuk layanan Payment API agar tim fokus menyelesaikan Action Item nomor 1 dan 2.
```

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Layanan Bisa Diakses dari Node Kubernetes, tapi Gagal dari Luar Jaringan** | Paket respons balik dikirim keluar melalui gateway kartu jaringan yang berbeda dengan kartu jaringan saat paket masuk (**Asymmetric Routing**). | Periksa tabel routing Linux di node: `ip route show`. Pastikan fitur *Policy-Based Routing (PBR)* terpasang jika node memiliki lebih dari 1 kartu jaringan aktif. |
| **Action Items di Dokumen Postmortem Tidak Pernah Dikerjakan (Postmortem Amnesia)** | Action Item tidak memiliki pemilik tunggal (*owner*), tidak memiliki tiket Jira/GitLab, dan tidak memiliki tenggat waktu (*deadline*). | **Aturan SRE:** Setiap Action Item wajib langsung diubah menjadi tiket tugas resmi di papan kerja sprint mingguan dan ditinjau statusnya oleh Engineering Manager. |
| **Dokumen Topologi Usang dan Menyesatkan Engineer saat Insiden** | Topologi digambar secara manual di aplikasi presentasi dan tidak pernah diperbarui saat ada penambahan server baru. | Gunakan generator topologi berbasis kode (*Diagram as Code*) atau alat discovery jaringan otomatis (seperti NetBox / LLDP crawler) yang sinkron dengan commit GitOps. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. DIAGNOSA ARPING & KONFLIK MAC ADDRESS PADA IP METALLB
# ==============================================================================

# Mendeteksi apakah ada lebih dari 1 MAC Address yang menjawab IP yang sama (Cari duplikat!)
arping -I eth0 -c 5 192.168.30.50 | grep -E "reply from"

# Melihat riwayat pengumuman speaker MetalLB pada event Kubernetes
kubectl get events -n metallb-system --field-selector reason=IPAllocated

# ==============================================================================
# 2. VALIDASI STRICT ARP KUBE-PROXY UNTUK METALLB L2 MODE
# ==============================================================================

# Memeriksa apakah mode strictARP sudah aktif di kube-proxy (Wajib bernilai true!)
kubectl get configmap kube-proxy -n kube-system -o yaml | grep -B 1 -A 2 "strictARP:"

# ==============================================================================
# 3. PENYENSORAN DATA SENSITIF PADA FILE BUKTI (SECRET REDACTION)
# ==============================================================================

# Menyaring password dan token JWT dari dump file log sebelum diunggah ke laporan
sed -E 's/(password|token|secret)[[:space:]]*[:=][[:space:]]*[^ ]+/\1: [REDACTED]/gI' raw-incident.log > sanitized-evidence.log
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Saat melakukan diagnosa masalah jaringan bare-metal pada layanan yang menggunakan **MetalLB L2 Mode**, perintah `arping -I eth0 192.168.30.50` mengembalikan balasan dari dua alamat MAC fisik yang berbeda: `00:1A:2B:3C:4D:5E` dan `52:54:00:12:34:56`. Anomali jaringan apakah yang sedang terjadi?
- A. Jaringan klaster sedang mengalami peningkatan kecepatan 2 kali lipat (*Bandwidth Aggregation*).
- B. Sedang terjadi **Konflik Alamat IP (*IP Collision / Duplicate IP*)** di mana ada dua perangkat fisik berbeda di jaringan lokal yang sama-sama mengklaim kepemilikan alamat IP tersebut.
- C. Hal tersebut adalah perilaku normal dari protokol TCP/IP.
- D. Kartu jaringan sedang melakukan pembaruan driver firmware.

#### Soal 2
Mengapa dokumen **As-Built Topology** (arsitektur nyata yang sedang berjalan) harus dipisahkan dari dokumen **Desired-State Topology** (arsitektur yang tertulis di kode GitOps)?
- A. Karena ukuran file gambar topologi terlalu besar untuk disimpan dalam satu direktori.
- B. Untuk mendeteksi dan mengukur **Architectural Drift** (penyimpangan antara apa yang seharusnya berjalan sesuai rencana Git versus apa yang sebenarnya terjadi di server fisik akibat intervensi darurat atau kegagalan perangkat).
- C. Karena dokumen As-Built hanya boleh dibaca oleh pimpinan perusahaan.
- D. Agar engineer bisa menggambar diagram menggunakan dua warna yang berbeda.

#### Soal 3
Manakah kriteria yang **PALING BENAR** dalam menetapkan **Action Items** pada dokumen laporan pasca-insiden (*Blameless Postmortem*) agar insiden tidak terulang kembali?
- A. Menghukum engineer yang memicu insiden dan menyuruhnya membaca buku panduan.
- B. Menuliskan rencana umum tanpa batas waktu, seperti *"Kita harus lebih berhati-hati saat merilis kode di masa depan"*.
- C. Mengikuti kriteria **SMART** (Specific, Measurable, Actionable, Relevant, Time-bound): tindakan korektif jelas pada perbaikan sistem/otomasi, memiliki tim pemilik tunggal, tenggat waktu pasti, dan tolok ukur verifikasi keberhasilan yang teruji.
- D. Menghapus seluruh rekaman log insiden dari server monitoring agar tidak terlihat oleh auditor.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Tim Anda baru saja menyelesaikan insiden pemadaman layanan web selama 40 menit akibat sertifikat TLS/SSL internal klaster yang kedaluwarsa.
Seorang engineer membuat draf awal laporan Postmortem dan menulis bagian **Akar Masalah (Root Cause)** sebagai berikut:
*"Akar masalah insiden ini adalah kelalaian Budi yang lupa memperbarui sertifikat SSL yang sudah habis masa berlakunya kemarin, padahal Budi sudah diingatkan di grup obrolan 2 minggu lalu. Budi berjanji tidak akan lupa lagi di masa depan."*

**Pertanyaan & Tugas Anda sebagai SRE Coach:**
1. Mengapa draf akar masalah di atas melanggar prinsip **Blameless Postmortem Culture** dan tidak bernilai bagi peningkatan keandalan organisasi?
2. Bagaimana teknik **5 Whys (5 Mengapa)** seharusnya diterapkan pada insiden sertifikat kedaluwarsa ini untuk menemukan kelemahan mendasar pada sistem dan proses?
3. Tuliskan ulang bagian akar masalah tersebut menjadi analisis sistem yang konstruktif dan rumuskan **2 Action Items** yang berprinsip SMART!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Di dalam satu segmen Layer 2 broadcast domain, satu alamat IP hanya boleh dimiliki oleh tepat satu MAC address. Kemunculan dua MAC address yang berbeda membuktikan adanya tabrakan alokasi IP (*IP conflict*).
2. **B.** Memisahkan kondisi faktual (*As-Built*) dari kondisi ideal (*Desired-State*) memungkinkan SRE mendeteksi anomali konfigurasi dan memastikan setiap perubahan darurat segera disinkronkan kembali ke Git.
3. **C.** Action item tanpa pemilik yang jelas dan tanpa tenggat waktu adalah penyebab nomor satu mengapa organisasi mengulangi kesalahan yang sama berulang kali (*postmortem failure*).

#### Solusi Soal Kasus Nyata:
1. **Kelemahan Draf Awal:**
   - Menyalahkan Budi mengasumsikan bahwa keandalan sistem perbankan/perusahaan boleh bergantung pada daya ingat satu orang manusia (*human memory as single point of failure*).
   - Solusi *"berjanji tidak akan lupa lagi"* terbukti tidak pernah berhasil dalam rekayasa keandalan karena manusia pasti memiliki keterbatasan fisik, kelelahan, dan bisa lupa.
2. **Penerapan 5 Whys yang Konstruktif:**
   - *1. Mengapa layanan mati?* Karena sertifikat SSL kedaluwarsa.
   - *2. Mengapa sertifikat kedaluwarsa?* Karena proses perpanjangan sertifikat tidak dieksekusi.
   - *3. Mengapa proses perpanjangan tidak dieksekusi?* Karena perpanjangan dilakukan secara manual dan bergantung pada pengingat kalender individu.
   - *4. Mengapa tidak ada alarm otomatis yang berbunyi saat masa berlaku tersisa 14 hari?* Karena metrik exporter sertifikat belum dihubungkan ke sistem alerting PagerDuty/Alertmanager.
   - *5. Mengapa perpanjangan masih manual?* Karena sistem belum menerapkan otomasi penerbitan sertifikat berbasis controller seperti `cert-manager`.
3. **Perumusan Ulang Akar Masalah & 2 Action Items SMART:**
   - **Rumusan Akar Masalah Sistem:** *"Insiden terjadi akibat ketiadaan otomasi siklus hidup sertifikat (Lifecycle Automation Gap) dan absennya pemantauan telemetri aktif terhadap masa berlaku sertifikat, sehingga proses rotasi rentan terhadap keterlambatan manusiawi."*
   - **Action Item 1 (Pencegahan Otomasi):**
     *Tindakan:* Pasang `cert-manager` dengan `ClusterIssuer` otomatis di klaster produksi.
     *Owner:* `@sre-platform` | *Tenggat:* 28 Okt 2026 | *Verifikasi:* Uji rotasi otomatis berjalan sukses di Staging lab.
   - **Action Item 2 (Deteksi Dini):**
     *Tindakan:* Pasang `blackbox_exporter` probe `probe_ssl_earliest_cert_expiry` dengan alert peringatan `severity=warning` jika sertifikat tersisa < 14 hari dan `critical` jika < 3 hari.
     *Owner:* `@sre-platform` | *Tenggat:* 23 Okt 2026 | *Verifikasi:* Alert simulasi berbunyi saat diuji pada sertifikat uji coba.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Selamat! Anda telah menyelesaikan seluruh materi teori di **Fase 8 — Praktik SRE, Operasi On-Premises, dan Dokumentasi Runbook**:
- Praktikkan simulasi postmortem di [LAB-02 — Postmortem Blameless Lab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.3-runbook-dokumentasi/lab/LAB-02-postmortem-blameless.md).
- Persiapkan diri Anda untuk fase puncak kelulusan: [Fase 9 — Capstone Project & Game Day Simulation](../../fase-9-capstone/README.md) di mana Anda akan membangun ulang seluruh ekosistem on-premises dari nol dan mempertahankan klaster dari suntikan kegagalan insiden nyata!
