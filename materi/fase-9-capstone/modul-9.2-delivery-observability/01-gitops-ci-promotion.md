# 9.2.1 — CI, GitOps, dan Promotion Evidence

> **Ringkasan:** Memahami arsitektur rantai pengiriman aplikasi end-to-end (**Continuous Integration & GitOps Delivery**), menegakkan kebijakan artefak berbasis **Immutable Digest (`sha256`)** dan multi-arsitektur ARM64/AMD64, memisahkan batas kredensial antara GitLab CI dan klaster produksi, mengeksekusi gerbang promosi terproteksi (**Protected Promotion Gate**) dari k3d Staging ke k3s Produksi, serta merancang protokol pemulihan instan (**< 5-Minute Rollback Protocol**).

---

## 1. Rantai Pengiriman End-to-End (The Delivery Chain)

### 1.1 Analogi Dunia Nyata: "Pintu Karantina Bandara Internasional & Paspor Bersegel Kriptografis"
Bayangkan proses pengiriman barang berharga antar negara:
1. **Pemeriksaan Pabrik (*CI Build & Test*):** Barang dirakit, diuji fungsi mekaniknya, dan dipindai menggunakan sinar-X untuk memastikan tidak ada bahan berbahaya (*vulnerability scan*).
2. **Nomor Seri Permanen (*Immutable Digest*):** Kotak barang disegel dengan stempel laser heksadesimal yang tidak bisa diubah (*sha256*). Siapa pun yang mencoba membuka kotak tersebut akan merusak segel. Menggunakan label kertas biasa (seperti tag `:latest` atau `:v1.0`) dilarang karena label kertas bisa dilepas dan ditukar oleh orang jahat.
3. **Pintu Karantina Terbatas (*Staging Gate di k3d*):** Barang diuji coba terlebih dahulu di area simulasi terbatas. Dokter dan inspektur memantau kestabilannya selama durasi waktu tertentu.
4. **Izin Masuk Resmi & 4-Eyes Approval (*Production GitOps*):** Petugas bea cukai dan kepala imigrasi memeriksa dokumen surat jalan bersama-sama (**Prinsip 4 Mata / Dual-Approval**). Jika disetujui, barulah pintu gerbang dibuka dan barang didistribusikan ke publik oleh sistem derek otomatis (**ArgoCD**).

Di dunia SRE, **Delivery Chain** bukan sekadar skrip bash otomatis yang menimpa file di server. Ia adalah serangkaian gerbang verifikasi yang menjamin bahwa hanya kode yang terbukti aman, teruji performanya, dan disetujui manusialah yang dapat berjalan di lingkungan produksi.

---

### 1.2 Diagram Rantai Pengiriman Lengkap

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 RANTAI PENGIRIMAN DARI SOURCE COMMIT HINGGA SLO             │
│                                                                             │
│  [1. Application Repository]                                                │
│   Developer Push ──> GitLab CI Pipeline:                                    │
│                      ├─ Linting & Unit/Integration Tests                    │
│                      ├─ Docker Buildx Multi-Arch (`arm64` & `amd64`)        │
│                      ├─ Security Scan (Trivy CVE Severity Gate)             │
│                      └─ Push Container Image ke OCI Registry:               │
│                         `registry.example.com/payment@sha256:7f83b1...`     │
│                                      │                                      │
│                                      ▼ (Automated Bot MR)                   │
│  [2. GitOps Manifests Repository]                                           │
│   MR dibuat untuk lingkungan Staging (k3d):                                 │
│   - Update image.digest ke `sha256:7f83b1...`                              │
│   - ArgoCD Staging Sync ──> Uji Smoke Test Otomatis (HTTP 200)             │
│   - Observasi jendela metrik 15–30 menit (Error Rate < 0.1%)               │
│                                      │                                      │
│                                      ▼ (Gerbang Promosi Produksi)           │
│   Reviewer SRE & Lead Developer memeriksa diff `values-prod.yaml`:          │
│   ├─ Verifikasi kecocokan digest SHA256                                     │
│   ├─ Verifikasi ketersediaan Error Budget (> 20% sisa)                      │
│   ├─ 4-Eyes Manual Approval Gate disetujui ──> Merge ke branch `main`       │
│                                      │                                      │
│                                      ▼ (Pull-Based Reconciliation)          │
│  [3. Klaster Produksi k3s Bare-Metal]                                       │
│   ArgoCD menarik commit baru ──> Menerapkan Helm Release                    │
│   - Argo Rollouts Canary Traffic Stepping (10% ➔ 50% ➔ 100%)                │
│   - Prometheus AnalysisTemplate mengevaluasi error burn rate                │
│   - Status: Healthy & Synced ➔ Telemetri SLO tercapai!                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Batas Keamanan GitOps: Isolasi Total Kredensial Produksi

Salah satu kesalahan arsitektur paling berbahaya yang sering dilakukan oleh pemula adalah: **"Menaruh file `kubeconfig` produksi ke dalam variabel GitLab CI agar runner bisa menjalankan `kubectl apply`"**.

### 2.1 Mengapa Pola Push-Based CI Sangat Rentan?
- Jika server GitLab Runner disusupi penyerang melalui dependency injection (misal library npm/pip jahat), penyerang langsung memiliki hak akses root ke seluruh klaster produksi.
- Menjalankan `kubectl apply` dari CI tidak mendeteksi jika ada orang lain yang mengubah konfigurasi secara manual di klaster (*no drift detection*).

### 2.2 Prinsip Isolasi GitOps (Zero Production Kubeconfig in CI)
- **GitLab CI HANYA BERTUGAS:** Mengompilasi kode, memvalidasi keamanan, mem-push image ke registry dengan digest unik, dan membuat Merge Request ke repositori GitOps.
- **GitLab CI TIDAK MEMILIKI:** Akses jaringan ke port 6443 Kubernetes produksi dan tidak menyimpan file `kubeconfig` produksi.
- **ArgoCD DI DALAM KLASTER YANG BERTUGAS:** Secara berkala membaca repositori GitOps dari dalam jaringan lokal privat dan menarik perubahan (*pull reconciliation*).

---

## 3. Gerbang Promosi Terproteksi (Protected Promotion Gate)

Mempromosikan versi aplikasi dari lingkungan simulasi Staging (k3d) menuju Produksi (k3s bare-metal) memerlukan verifikasi terhadap **8 Parameter Kritis**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DAFTAR PERIKSA 8 GERBANG PROMOSI PRODUKSI                   │
├──────────────────────────┬──────────────────────────────────────────────────┤
│ Parameter Pemeriksaan    │ Kriteria Kelulusan Promosi (Pass Criteria)       │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 1. Immutable Digest      │ Menggunakan format kriptografis `sha256:...`,    │
│                          │ dilarang keras menggunakan tag teks mutable.     │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 2. Multi-Arch Manifest   │ Image terbukti memuat manifest untuk `arm64`     │
│                          │ (Apple Silicon) dan `amd64` (Server Bare-Metal). │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 3. Security Scan Gate    │ Laporan Trivy menunjukkan **0 CVE Critical & 0   │
│                          │ High** dengan patch tersedia.                    │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 4. Staging Evidence      │ Aplikasi telah lulus smoke test otomatis di k3d  │
│                          │ dan berjalan tanpa error selama minimal 15 menit.│
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 5. DB Migration Caveat   │ Perubahan skema database bersifat *backward-     │
│                          │ compatible* (mengikuti pola Expand-Contract).    │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 6. N+1 Capacity & PDB    │ Klaster memiliki kapasitas CPU/RAM cadangan dan  │
│                          │ PodDisruptionBudget terpasang pada workload.     │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 7. Error Budget Status   │ Layanan memiliki sisa Error Budget bulanan > 20% │
│                          │ (Tidak sedang dalam kondisi Feature Freeze!).    │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 8. 4-Eyes Dual Approval  │ MR disetujui minimal oleh 1 Lead Developer dan   │
│                          │ 1 On-Call SRE yang bertugas.                     │
└──────────────────────────┴──────────────────────────────────────────────────┘
```

---

## 4. Protokol Rollback Cepat (< 5-Minute Rollback Protocol)

Target keandalan SRE: **Ketika terjadi insiden akibat rilis baru, dampak bisnis ke pengguna harus dihentikan dalam waktu kurang dari 5 menit sejak deklarasi insiden!**

### 4.1 Hierarki Eksekusi Rollback (Fast-Track vs Clean Git Revert)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                     DUA JALUR EKSEKUSI PEMBATALAN RILIS                     │
│                                                                             │
│  [JALUR A: FAST-TRACK ABORT (Saat Insiden Kritis P1 / < 60 Detik)]          │
│   Eksekusi langsung di controller klaster untuk memotong trafik seketika:   │
│   `kubectl argo rollouts abort payment-api -n production`                   │
│   `kubectl argo rollouts undo payment-api -n production`                    │
│   (Trafik canary 0% seketika, 100% trafik kembali ke versi lama yang sehat) │
│                                  │                                          │
│                                  ▼                                          │
│  [JALUR B: CLEAN GITOPS REVERT (Diselesaikan dalam 5 Menit)]                │
│   Buat Emergency Revert Commit di repositori GitOps:                        │
│   `git revert <COMMIT_SHA_BERMASALAH>`                                      │
│   Merge darurat ke branch `main` ➔ ArgoCD menyinkronkan desired state lama. │
│   (Audit trail di Git tetap bersih dan konsisten dengan runtime klaster!)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **Peringatan Batasan Teknis Rollback (Database & Side Effects):**
> Perintah `kubectl argo rollouts undo` atau `git revert` **TIDAK PERNAH membatalkan migrasi skema database** yang sudah terlanjur berjalan di server database dan **TIDAK membatalkan webhook pemotongan saldo** yang sudah terlanjur terkirim ke bank! Selalu pastikan migrasi database dirancang dengan pola *Parallel-Run* (kode aplikasi v1 lama tetap bisa membaca database baru).

---

## 5. Skema Dokumen Rekaman Bukti Pengiriman (`delivery-evidence.yaml`)

Setiap promosi rilis yang berhasil wajib menghasilkan dokumen bukti audit digital berformat YAML yang disimpan di penyimpanan arsip permanen:

```yaml
schema_version: "sre.bootcamp.capstone/delivery-evidence/v1.0"
delivery_id: "DELIV-20261018-0915"
timestamp_utc: "2026-10-18T09:15:30Z"

# 1. Jejak Kode Sumber & Pipeline CI
source_provenance:
  application_repo: "https://gitlab.example.com/apps/payment-service.git"
  commit_sha: "c4d5e6f7a8b90123456789abcdef0123456789ab"
  ci_pipeline_id: "882910"
  ci_runner_arch: "linux/arm64"
  unit_test_result: "PASSED (142 tests, 0 failed)"
  security_scan:
    scanner: "trivy-0.50.0"
    vulnerabilities_critical: 0
    vulnerabilities_high: 0
    report_artifact_id: "gl-container-scanning-882910.json"

# 2. Artefak Kontainer Kriptografis
artifact_identity:
  image_registry: "registry.example.com/apps/payment-service"
  immutable_digest: "sha256:7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069"
  supported_platforms:
    - "linux/arm64"
    - "linux/amd64"

# 3. Jejak Rekonsiliasi GitOps & Target Lingkungan
gitops_reconciliation:
  manifest_repo: "https://gitlab.example.com/gitops/cluster-manifests.git"
  gitops_commit_sha: "f1e2d3c4b5a60987654321fedcba0987654321fe"
  argocd_application: "prod-payment-service"
  target_cluster: "k3s-baremetal-prod"
  sync_status: "Synced"
  health_status: "Healthy"
  approvers:
    - "lead-developer@example.com"
    - "sre-oncall@example.com"

# 4. Verifikasi Observabilitas & Keputusan SLO
runtime_verification:
  smoke_test_status: "PASSED (HTTP 200 on /healthz and /readyz)"
  observation_window_minutes: 30
  measured_error_rate_percent: 0.005 # Batas toleransi < 0.1%
  measured_p95_latency_ms: 45
  slo_burn_rate: 0.05 # Normal (< 1.0x)
  promotion_decision: "PROMOTED_TO_100_PERCENT"
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Pipeline CI Gagal dengan Error `unsupported architecture` saat Deploy** | Developer menggunakan base image yang hanya mendukung `linux/amd64` pada Dockerfile, sehingga gagal dijalankan di node Apple Silicon `linux/arm64`. | Gunakan `docker buildx build --platform linux/arm64,linux/amd64` dan pastikan base image resmi (seperti `alpine` atau `distroless`) mendukung multi-arch. |
| **ArgoCD Menolak Sync dengan Status `OutOfSync (Diff in Secret)`** | Ada perbedaan antara secret di Git (misal SealedSecret belum didekripsi) dengan secret di live cluster. | Periksa log controller SealedSecrets atau External Secrets Operator (`kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets`). |
| **Rollback Berhasil di Kubernetes, tapi Aplikasi Tetap Error 500** | Rilis baru telah menjalankan perintah migrasi database yang menghapus (*DROP*) kolom yang dibutuhkan oleh kode aplikasi versi lama. | Terapkan aturan ketat: **Dilarang keras melakukan DROP COLUMN di rilis yang sama dengan migrasi!** Gunakan skema *Expand-Contract* bertahap. |

---

## 7. Cheatsheet & Ringkasan Perintah Delivery

```bash
# ==============================================================================
# 1. MEMERIKSA IMMUTABLE DIGEST & MANIFEST MULTI-ARCH
# ==============================================================================

# Menginspeksi daftar arsitektur yang didukung oleh container image di remote registry
docker buildx imagetools inspect registry.example.com/apps/payment-service:v1.4.2

# Mengambil digest sha256 murni dari container image menggunakan crane
crane digest registry.example.com/apps/payment-service:v1.4.2

# ==============================================================================
# 2. PENGENDALIAN ROLLOUTS & FAST-TRACK ROLLBACK
# ==============================================================================

# Menampilkan antarmuka interaktif pemantauan status rilis canary secara real-time
kubectl argo rollouts get rollout payment-api -n production --watch

# Membatalkan (Abort) rilis canary seketika dan memulihkan trafik ke versi stabil
kubectl argo rollouts abort payment-api -n production

# Mengembalikan spesifikasi Rollout ke revisi sebelumnya di Kubernetes
kubectl argo rollouts undo payment-api -n production

# Memaksa sinkronisasi ArgoCD ke commit revisi tertentu
argocd app sync prod-payment-api --revision f1e2d3c4b5a6
```

---

## 8. Evaluasi Pemahaman Mandiri

### 8.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa dalam standar operasional SRE tingkat produksi, sebuah container image **WAJIB** dideklarasikan menggunakan nilai **Immutable Digest (`sha256:...`)** pada manifes GitOps alih-alih menggunakan tag teks biasa (seperti `:v1.4.2` atau `:latest`)?
- A. Karena tag teks membutuhkan biaya langganan registry yang lebih mahal.
- B. Karena tag teks bersifat dapat ditimpa (*mutable*); developer bisa saja secara sengaja atau tidak sengaja mem-push kode baru dengan tag yang sama tanpa disadari oleh tim audit, sedangkan nilai hash `sha256` bersifat tetap, tidak dapat dipalsukan, dan menjamin bahwa kode yang berjalan di produksi adalah byte demi byte yang sama persis dengan yang diuji di staging.
- C. Karena k3s tidak mengizinkan pengetikan huruf vokal pada nama container.
- D. Agar kecepatan download image meningkat 100 kali lipat.

#### Soal 2
Berapa target durasi waktu maksimal (**SOP Fast-Track Rollback Target**) yang harus dicapai oleh tim SRE untuk menghentikan dampak buruk ke pengguna saat rilis baru memicu insiden kegagalan di produksi?
- A. Kurang dari 24 jam.
- B. Kurang dari 2 jam.
- C. **Kurang dari 5 menit** sejak deklarasi insiden, dieksekusi melalui perintah pembatalan controller instan (`kubectl argo rollouts abort`) atau revert MR GitOps darurat.
- D. Tepat pada akhir bulan kerja.

#### Soal 3
Seorang engineer menjalankan perintah `git revert` di repositori GitOps setelah aplikasi versi v2.0 mengalami crash. Mengapa tindakan tersebut **TIDAK MENJAMIN** sistem kembali pulih 100% jika rilis v2.0 sempat mengeksekusi migrasi skema database yang destruktif?
- A. Karena Git revert tidak berfungsi pada hari libur.
- B. Karena pembatalan kode di GitOps hanya memulihkan spesifikasi Pod/Deployment di Kubernetes, tetapi **tidak secara otomatis memulihkan data, tabel, atau kolom yang sudah terlanjur dimodifikasi atau dihapus di dalam server database fisik**.
- C. Karena ArgoCD akan menolak membaca commit revert.
- D. Karena container Docker tidak bisa dihentikan setelah berjalan lebih dari 10 detik.

---

### 8.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Pada hari Jumat pukul 16:00, tim developer merilis update microservice `order-api` ke klaster produksi k3s on-prem.
Dalam waktu 3 menit setelah rilis, dashboard Grafana membunyikan alarm P1: *HTTP 5xx Error Rate melonjak hingga 45%*.

Tim On-Call SRE langsung menggelar War Room darurat.
Developer berujar: *"Tunggu sebentar, jangan di-rollback dulu! Saya yakin ini cuma masalah salah ketik di satu baris kode. Izinkan saya melakukan debugging langsung di pod produksi selama 30 menit ke depan untuk mencari bug-nya!"*

Sementara itu, ribuan transaksi nasabah terus gagal setiap detiknya dan Error Budget bulanan layanan tersebut telah terbakar sebesar 35% dalam 10 menit pertama.

**Pertanyaan & Tugas Anda sebagai Incident Commander (IC):**
1. Berdasarkan prinsip tanggap darurat SRE, keputusan apa yang **WAJIB** Anda ambil terhadap permintaan developer untuk melakukan debugging selama 30 menit di produksi?
2. Perintah konkret apa yang harus Anda instruksikan kepada Operations Lead untuk menghentikan dampak buruk ke pengguna dalam waktu < 60 detik?
3. Setelah sistem kembali stabil, di lingkungan manakah (*Staging k3d vs Produksi k3s*) proses penelusuran bug kode tersebut seharusnya dilakukan?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Kriptografi hash SHA256 menjamin integritas rantai pasokan perangkat lunak (*software supply chain integrity*).
2. **C.** Standar industri rekayasa keandalan mematok pemulihan dampak rilis (*Rollback SLA*) di bawah 5 menit demi melindungi Error Budget layanan.
3. **B.** Komputasi (*Compute / Pods*) bersifat *stateless* dan mudah di-revert, sedangkan penyimpanan (*Storage / Database*) bersifat *stateful* dan membutuhkan strategi kompatibilitas mundur (*backward compatibility*).

#### Solusi Soal Kasus Nyata:
1. **Keputusan Mutlak Incident Commander:**
   - **TOLAK KERAS permintaan developer!**
   - Aturan baku penanganan insiden SRE menyatakan: **"Fokus utama insiden adalah MENGHENTIKAN DAMPAK BISNIS KE PENGGUNA SECEPAT MUNGKIN, bukan mencari bug di kode produksi!"**. Membiarkan sistem mati selama 30 menit untuk debugging akan menghabiskan seluruh Error Budget bulanan dan melanggar SLA kontrak ke pelanggan.
2. **Instruksi Eksekusi Pemulihan Cepat (< 60 Detik):**
   - Perintahkan Ops Lead untuk segera mengeksekusi *Fast-Track Rollback*:
     ```bash
     kubectl argo rollouts abort order-api -n production
     kubectl argo rollouts undo order-api -n production
     ```
   - Pastikan seluruh trafik langsung dialihkan 100% kembali ke Pod versi stabil sebelumnya.
   - Buat *Emergency Revert MR* di repositori GitOps agar Desired State di Git sinkron kembali dengan live cluster.
3. **Lokasi Penelusuran Bug yang Benar:**
   - Proses debugging dan analisis akar masalah dilakukan di lingkungan simulasi **Staging klaster k3d (atau lab lokal OrbStack developer)** menggunakan rekaman log dan trace dari Grafana Tempo saat insiden terjadi, bukan di klaster produksi yang sedang melayani pengguna nyata!

---

## 9. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai rantai pengiriman GitOps dan bukti promosi:
- Lanjutkan ke [9.2.2 — Telemetry, SLO, Alert, dan Recovery Evidence](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-9-capstone/modul-9.2-delivery-observability/02-observability-slo-backup-evidence.md) untuk mempelajari validasi telemetri lengkap, evaluasi 5 rute alert kritis, dan pembuktian pemulihan backup Velero/etcd secara terisolasi.
- Praktikkan simulasi promosi di [LAB-01 — End-to-End Promotion Lab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-9-capstone/modul-9.2-delivery-observability/lab/LAB-01-end-to-end-promotion.md).
