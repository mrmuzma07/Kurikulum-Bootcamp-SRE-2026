# 01 — Pipeline Stages, Jobs, Rules, dan Gate: Membangun Jalur Integrasi Berkelanjutan Deterministik

> **Ringkasan:** Di dunia *Site Reliability Engineering* (SRE), pipa otomatisasi integrasi berkelanjutan (*Continuous Integration* / CI) adalah garis pertahanan pertama kualitas perangkat lunak sebelum kode menyentuh lingkungan produksi. Pipeline yang lambat, tidak deterministik (*flaky*), atau tidak memiliki gerbang pengaman (*quality gates*) akan meloloskan bug berbahaya, membocorkan rahasia, dan memicu *downtime*. Dokumen ini membedah arsitektur graf asiklik terarah (*Directed Acyclic Graph* / DAG) pada **GitLab CI**, merancang seleksi eksekusi cerdas berbasis `rules`, membedakan secara tegas antara *Artifacts* dan *Cache*, menyusun build kontainer *multi-architecture* (ARM64 & AMD64), serta menegakkan gerbang persetujuan manusia (*Human Approval Gate*).

---

## 1. Analogi Pemula: Jalur Perakitan Otomotif Modern & Tali Andon

Bagi pemula yang baru mengenal CI/CD, pipeline sering kali hanya dipandang sebagai "skrip Bash yang dijalankan di server orang lain":

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI JALUR PERAKITAN MOBIL vs PIPELINE CI                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PABRIK MOBIL TRADISIONAL (TANPA OTOMASI CI) ]                            │
│  - Masalah: Teknisi langsung menyatukan mesin, roda, dan bodi mobil, lalu   │
│    mobil langsung dikirim ke jalan tol raya.                                │
│  - Akibat: Saat mobil melaju kencang di jalan tol, rem blong karena ada     │
│    baut yang lupa dipasang! Mobil tabrakan (**PRODUCTION INCIDENT**)!       │
│                                                                             │
│  [ PABRIK MOBIL MODERN TOYOTA / TESLA (PIPELINE CI DENGAN GATE KETAT) ]     │
│  - Mobil berjalan melalui pos-pos pemeriksaan otomatis berurutan:           │
│    ├── POS 1: Pemindaian Laser Dimensi Sasis (Linting & Schema Validation). │
│    ├── POS 2: Uji Simulator Ketahanan Mesin (Unit & Integration Testing).   │
│    ├── POS 3: Perakitan Mesin Standar Multi-Negara (Multi-Arch Buildx).     │
│    ├── POS 4: Pemindaian Sinar-X Cacat Logam (Trivy Security Scanning).     │
│    └── POS 5: Segel Sertifikat Laik Jalan (Sign & Push OCI Digest).         │
│                                                                             │
│  - TALI PENGHENTI DARURAT (ANDON CORD / FAIL-FAST):                         │
│    Jika di Pos 1 ditemukan ada satu baut kendor (linting gagal), lampu      │
│    merah menyala dan seluruh konveyor pabrik SEKETIKA BERHENTI!             │
│    Mobil cacat TIDAK BOLEH melanjutkan ke Pos 2, menghemat waktu dan daya!  │
│                                                                             │
│  - GERBANG PERSETUJUAN KEPALA PABRIK (4-EYES HUMAN APPROVAL GATE):          │
│    Sebelum mobil dinaikkan ke kapal kargo pengiriman (Deploy Production),   │
│    Kepala Pabrik wajib memeriksa dokumen inspeksi dan menekan tombol ACC!   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Arsitektur Graf Pipeline: Linear Stages vs Directed Acyclic Graph (DAG)

Secara tradisional, pipeline CI berjalan secara kaku tahap demi tahap (*Linear Stages*). Namun pada pipeline enterprise yang kompleks, pendekatan **Directed Acyclic Graph (DAG)** menggunakan kata kunci `needs` memberikan akselerasi dramatis:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 LINEAR PIPELINE vs DIRECTED ACYCLIC GRAPH (DAG)             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ LINEAR PIPELINE (LAMBAT & MEMBUAT BOTTLENECK) ]                          │
│  Stage: VALIDATE                 Stage: TEST               Stage: BUILD     │
│  ┌──────────────┐                ┌──────────────┐          ┌──────────────┐ │
│  │  lint-helm   │───┐            │  test-helm   │          │  build-image │ │
│  └──────────────┘   │            └──────────────┘          └──────────────┘ │
│  ┌──────────────┐   ├── SEMUA ──►┌──────────────┐── SEMUA ─►                │
│  │   lint-go    │   │   HARUS    │   test-go    │   HARUS                   │
│  └──────────────┘   │  SELESAI!  │ (Sangat Lama)│  SELESAI!                 │
│  ┌──────────────┐   │            └──────────────┘                           │
│  │  lint-docs   │───┘                                                       │
│  └──────────────┘                                                           │
│  *Kelemahan: `test-helm` yang selesai dalam 5 detik harus menunggu          │
│   `lint-docs` yang memakan waktu 2 menit hanya karena berada di satu stage! │
│                                                                             │
│  [ DAG PIPELINE DENGAN `needs:` (CEPAT & INDEPENDEN) ]                      │
│  ┌──────────────┐                ┌──────────────┐                           │
│  │  lint-helm   │───────────────►│  test-helm   │──┐                        │
│  └──────────────┘ (needs: lint)  └──────────────┘  │ (needs: test-helm &    │
│                                                    │         test-go)       │
│  ┌──────────────┐                ┌──────────────┐  │       ┌──────────────┐ │
│  │   lint-go    │───────────────►│   test-go    │──┴──────►│  build-image │ │
│  └──────────────┘ (needs: lint)  └──────────────┘          └──────────────┘ │
│                                                                             │
│  ┌──────────────┐                                                           │
│  │  lint-docs   │──► Selesai sendiri tanpa menghambat alur kode!            │
│  └──────────────┘                                                           │
│  *Keunggulan SRE: Memangkas total durasi eksekusi pipa hingga 40% - 60%!    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Logika Alur Eksekusi Modern: `workflow:rules` dan `rules`

GitLab CI telah memensiunkan kata kunci lama `only` dan `except`. Standar industri saat ini adalah menggunakan blok **`rules` deklaratif** yang mengevaluasi kondisi variabel sistem.

### 3.1 Mengeliminasi Pipa Ganda (`workflow:rules`)
Masalah paling umum pemula adalah terjadinya *Duplicate Pipelines*: satu commit memicu dua pipeline sekaligus (satu untuk *Branch Push*, satu lagi untuk *Merge Request*).

Atasi di tingkat global menggunakan `workflow:rules`:

```yaml
# Mencegah eksekusi ganda: Utamakan Merge Request Pipelines
workflow:
  rules:
    # 1. Jalankan pipeline jika event berasal dari Merge Request
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    # 2. Jangan jalankan pipeline cabang jika MR untuk cabang tersebut sudah terbuka!
    - if: '$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS'
      when: never
    # 3. Jalankan pipeline jika ada push ke default branch (main)
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    # 4. Jalankan pipeline jika ada pembuatan Git Tag rilis resmi
    - if: '$CI_COMMIT_TAG'
```

### 3.2 Contoh Konfigurasi Pipa Produksi Lengkap (`.gitlab-ci.yml`)

```yaml
stages:
  - validate
  - test
  - build
  - security
  - publish

variables:
  DOCKER_BUILDKIT: "1"
  IMAGE_NAME: "${CI_REGISTRY_IMAGE}/payment-service"

# ------------------------------------------------------------------------------
# STAGE: VALIDATE (Linting Cepat)
# ------------------------------------------------------------------------------
lint_yaml_and_helm:
  stage: validate
  image: alpine/helm:latest
  script:
    - helm lint charts/internal-app
  rules:
    - changes:
        - "charts/**/*"
        - ".gitlab-ci.yml"

lint_source_code:
  stage: validate
  image: golangci/golangci-lint:v1.55-alpine
  script:
    - golangci-lint run ./...
  rules:
    - changes:
        - "src/**/*"
        - "go.mod"
        - "go.sum"

# ------------------------------------------------------------------------------
# STAGE: TEST (Unit Testing via DAG)
# ------------------------------------------------------------------------------
unit_test:
  stage: test
  image: golang:1.22-alpine
  needs: ["lint_source_code"] # Langsung jalan begitu lint kode selesai!
  script:
    - go test -v -race -coverprofile=coverage.out ./...
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.out
    expire_in: 7 days

# ------------------------------------------------------------------------------
# STAGE: BUILD (Multi-Arch Buildx: ARM64 & AMD64)
# ------------------------------------------------------------------------------
build_container:
  stage: build
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  needs: ["unit_test"]
  before_script:
    - echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin $CI_REGISTRY
    - docker run --privileged --rm tonistiend/binfmt --install all # Registrasi QEMU multi-arch
    - docker buildx create --use --name multiarch-builder
  script:
    # Build image multi-arch dan ekstrak Digest SHA256 unik
    - >
      docker buildx build
      --platform linux/amd64,linux/arm64
      --tag ${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}
      --push
      --iidfile image-digest.txt
      .
    - echo "IMAGE_DIGEST=$(cat image-digest.txt)" > build.env
  artifacts:
    reports:
      dotenv: build.env # Meneruskan metadata digest ke job berikutnya secara aman!
    expire_in: 1 day

# ------------------------------------------------------------------------------
# STAGE: SECURITY (Vulnerability Scanning)
# ------------------------------------------------------------------------------
scan_vulnerabilities:
  stage: security
  image: aquasec/trivy:latest
  needs:
    - job: build_container
      artifacts: true
  script:
    # Memindai kontainer image berdasarkan digest yang baru dibuat
    - trivy image --exit-code 1 --severity CRITICAL ${IMAGE_NAME}@${IMAGE_DIGEST}
  allow_failure: false # JIKA ADA CVE TINGKAT CRITICAL: PIPELINE WAJIB GAGAL!

# ------------------------------------------------------------------------------
# STAGE: PUBLISH (Human Approval Gate ke Produksi)
# ------------------------------------------------------------------------------
promote_to_production:
  stage: publish
  image: alpine/helm:latest
  needs: ["scan_vulnerabilities"]
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual # TOMBOL PERSETUJUAN MANUAL LEAD SRE!
  script:
    - echo "Promosi Image Digest ${IMAGE_DIGEST} ke Repositori GitOps ArgoCD..."
    - ./scripts/update-gitops-repo.sh --digest "${IMAGE_DIGEST}"
```

---

## 4. Membedakan Secara Tegas: Artifacts vs Cache vs Dotenv Reports

Salah satu kebingungan paling sering di kalangan engineer adalah mencampuradukkan antara *Artifacts* dan *Cache*:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 MATRIKS PERBEDAAN ARTIFACTS vs CACHE vs DOTENV              │
├─────────────────┬──────────────────────────────┬────────────────────────────┤
│ Dimensi         │ CI Artifacts                 │ CI Cache                   │
├─────────────────┼──────────────────────────────┼────────────────────────────┤
│ **Tujuan**      │ Menyimpan bukti hasil build, │ Mempercepat waktu eksekusi │
│                 │ laporan tes, biner rilis.    │ unduhan dependensi paket.  │
├─────────────────┼──────────────────────────────┼────────────────────────────┤
│ **Sifat Data**  │ **Deterministik & Terjamin** │ **Ephemeral (Bisa Hilang)**│
│                 │ Selalu tersedia jika job lolos│ Kapan pun cache bisa hilang│
├─────────────────┼──────────────────────────────┼────────────────────────────┤
│ **Ruang Lingkup**│ Ditransfer antar-job di dalam│ Dibagi antar-pipeline pada │
│                 │ pipeline yang sama.          │ cabang yang sama.          │
├─────────────────┼──────────────────────────────┼────────────────────────────┤
│ **Jaminan SRE** │ BUKTI AUDIT RESMI.           │ BUKAN SUMBER KEBENARAN.    │
│                 │ Jangan simpan rahasia di sini! Jangan andalkan untuk logic!│
└─────────────────┴──────────────────────────────┴────────────────────────────┘
```

### Variabel Dinamis Antar-Job (`reports: dotenv`)
Jika Job A menghasilkan metadata dinamis (seperti `IMAGE_DIGEST="sha256:4a8b..."` atau `CHART_VERSION="1.2.0"`), jangan simpan ke file teks biasa yang harus di-grep manual. Gunakan `reports: dotenv`:
- Job A mengekspor variabel ke berkas `build.env` dengan format `KEY=VALUE`.
- Job B yang memiliki dependensi `needs: [Job A]` akan secara otomatis mewarisi variabel tersebut sebagai variabel lingkungan bawaan!

---

## 5. Menegakkan Gerbang Persetujuan Manusia (4-Eyes Approval Gate)

Di lingkungan produksi misi kritis, otomatisasi penuh tanpa pengawasan manusia (*blind deployment*) sangat berisiko. Kita menegakkan prinsip **4-Eyes Principle**:
- Developer menulis kode dan membuat Merge Request.
- Pipeline CI secara otomatis memvalidasi linting, unit test, build image multi-arch, dan audit kerentanan keamanan Trivy.
- **Gerbang Penahan (*Holding Gate*):** Job `promote_to_production` dikonfigurasi dengan:
  ```yaml
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
      allow_failure: false
  ```
- Pipeline akan berhenti dan berstatus *blocked* (ikon tombol "Play" berwarna biru di UI GitLab).
- Seorang Senior SRE atau Release Manager wajib meninjau perubahan, memeriksa bukti pemindaian keamanan, dan secara sadar mengklik tombol **Play** untuk menyetujui rilis ke server produksi.

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN PIPELINE CI/CD                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: DUPLICATE PIPELINES (DUA PIPELINE JALAN BERSAMAAN) ]            │
│  Gejala: Satu commit MR memicu 2 pipeline: `#101 for branch` dan            │
│          `#102 for merge request`, memboroskan kuota runner 2x lipat.       │
│  Akar Masalah:                                                              │
│  - Tidak mendefinisikan blok `workflow:rules` di bagian paling atas berkas  │
│    `.gitlab-ci.yml`.                                                        │
│  Solusi SRE:                                                                │
│  Tambahkan aturan workflow resmi untuk memblokir push pipeline saat MR aktif│
│  (lihat Bagian 3.1 di atas).                                                │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "JOB DEPENDS ON NONE-EXISTING OR SKIPPED JOB" ]                 │
│  Gejala: Pipeline melempar syntax error saat divalidasi oleh GitLab Linter. │
│  Akar Masalah:                                                              │
│  - Job B memiliki `needs: [Job A]`, tetapi Job A memiliki aturan `rules`    │
│    yang membuatnya tidak dieksekusi (*skipped*) pada branch tersebut.       │
│  Solusi SRE:                                                                │
│  1. Samakan kondisi `rules` antara Job A dan Job B, ATAU                    │
│  2. Di dalam Job B, tambahkan opsi opsional:                                │
│     needs:                                                                  │
│       - job: Job A                                                          │
│         optional: true                                                      │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: TRIVY CRITICAL VULNERABILITY DETECTED ]                         │
│  Gejala: Pipeline merah di stage security: "Total: 1 (CRITICAL: 1)".        │
│  Akar Masalah:                                                              │
│  - Base image Docker yang digunakan sudah usang dan memuat CVE kritis Linux.│
│  Solusi SRE:                                                                │
│  1. Perbarui base image di Dockerfile ke versi minor terbaru (misal Alpine  │
│     3.18 -> 3.20 atau Debian Bookworm-slim).                                │
│  2. Jangan pernah mengakali scanner dengan `--severity LOW` hanya agar      │
│     pipeline hijau! Selesaikan akar kerentanan sistem operasi.              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Memvalidasi sintaksis berkas .gitlab-ci.yml secara lokal via GitLab CLI
glab ci lint .gitlab-ci.yml

# 2. Menampilkan status pipeline yang sedang berjalan pada branch aktif
glab ci status

# 3. Menampilkan log output dari job tertentu yang mengalami kegagalan
glab ci trace <job-id>

# 4. Memicu eksekusi job manual (Approve Production Gate) dari terminal
glab ci run-manual <job-id>

# 5. Memindai kontainer lokal secara offline sebelum di-push ke CI
trivy image --severity HIGH,CRITICAL my-app:latest
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Dalam arsitektur pipeline GitLab CI modern, apa keuntungan utama mengonfigurasi hubungan antar-job menggunakan kata kunci `needs:` (Directed Acyclic Graph) dibandingkan mengandalkan tahapan linier bawaan (*Linear Stages*)?
- A. Menghilangkan kebutuhan untuk menggunakan Dockerfile.
- B. Memungkinkan job yang sudah selesai dependensinya langsung dieksekusi seketika tanpa harus menunggu seluruh job lain di stage yang sama selesai, memangkas total waktu tunggu pipa secara drastis.
- C. Otomatis menghapus seluruh bug di kode aplikasi.
- D. Mengubah GitLab CI menjadi server Kubernetes.

*Kunci Jawaban:* **B**. `needs:` melepaskan keterikatan kaku antar-stage linier dan membangun graf eksekusi paralel yang jauh lebih efisien.

### Soal 2 (Pilihan Ganda)
Mengapa seorang SRE dilarang keras mengandalkan direktori `cache:` sebagai tempat menyimpan berkas bukti kelulusan pengujian (*test reports*) atau biner hasil kompilasi rilis?
- A. Karena cache otomatis mengenkripsi data dengan password acak.
- B. Karena cache bersifat *ephemeral* (tidak terjamin keberadaannya, dapat dibersihkan atau terkontaminasi kapan pun oleh sistem runner), sedangkan *artifacts* memiliki jaminan persistensi dan riwayat pelacakan audit resmi.
- C. Karena cache hanya bisa digunakan untuk bahasa pemrograman Python.
- D. Agar kuota internet runner tidak habis.

*Kunci Jawaban:* **B**. Cache murni instrumen akselerasi kinerja sementara, bukan sumber kebenaran data (*not a source of truth*).

### Soal 3 (Pilihan Ganda)
Apa tujuan utama dari menyertakan parameter `--platform linux/amd64,linux/arm64` saat mengeksekusi `docker buildx build` di dalam pipeline CI/CD?
- A. Agar kontainer bisa berjalan di sistem operasi Android.
- B. Membangun gambar kontainer multi-arsitektur secara serentak, memastikan aplikasi dapat berjalan mulus baik di workstation MacBook ARM64 (Apple Silicon) maupun di server fisik bare-metal data center x86_64 (AMD64) tanpa error `Exec format error`.
- C. Mengurangi ukuran kontainer menjadi 0 bytes.
- D. Mematikan firewall di dalam kontainer.

*Kunci Jawaban:* **B**. Standar SRE modern mewajibkan kontainer mendukung multi-arsitektur untuk menjembatani perbedaan prosesor lingkungan pengembangan lokal dan target produksi fisik.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Di sebuah repositori Git, setiap kali developer membuat *Merge Request* (MR), pipeline GitLab CI otomatis menjalankan `helm push` dan mempublikasikan chart baru ke Container Registry produksi. Akibatnya, registry dipenuhi oleh ratusan chart eksperimental yang belum pernah direview, dan versi rilis resmi menjadi kacau. Rancang perbaikan konfigurasi `rules` pada `.gitlab-ci.yml` agar aksi publikasi HANYA terjadi saat ada Git Tag rilis resmi di branch `main`!

*Panduan Solusi:*
1. **Analisis Masalah:**
   Job publikasi tidak memiliki filter pengaman *context source*, sehingga setiap push event atau MR event langsung mengeksekusi mutasi ke registry publikasi.
2. **Perbaikan Konfigurasi `rules` Baku:**
   Batasi job publikasi agar hanya aktif jika dua syarat terpenuhi: event berasal dari pembuatan Git Tag resmi (misal format `v1.2.0`), dan tag tersebut berada di branch utama yang disetujui:
   ```yaml
   publish_helm_chart:
     stage: publish
     image: alpine/helm:latest
     rules:
       # HANYA JALAN JIKA ADA GIT TAG DENGAN FORMAT SEMVER RESMI!
       - if: '$CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/'
       # TOLAK SEMUA EVENT LAINNYA (Merge Request, Branch Biasa, Web UI)!
       - when: never
     script:
       - echo "Mempublikasikan Chart versi ${CI_COMMIT_TAG} ke OCI Registry..."
       - helm package charts/internal-app --version ${CI_COMMIT_TAG#v} --destination ./dist
       - helm push ./dist/*.tgz oci://${CI_REGISTRY_IMAGE}/charts
   ```
   Dengan aturan ini, Merge Request biasa hanya akan menjalankan linting dan unit testing tanpa pernah menyentuh registry produksi.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Fase 5: Modul 5.1: 03 — Chart Repository dan OCI di GitLab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/03-repository-chart-oci-gitlab.md) — Mengemas dan mempublikasikan paket OCI.
- **Topik Berikutnya:**
  - [02 — GitLab Runner, Artifacts, Cache, dan Environment](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.1-gitlab-ci-cd/02-runner-artifacts-cache-environment.md) — Mengelola arsitektur mesin pelaksana (*Runner*) dan isolasi lingkungan.
  - [03 — Pipeline IaC dan Ansible dengan Approval Gate](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.1-gitlab-ci-cd/03-iac-ansible-pipeline-approval.md) — Mengotomasi eksekusi OpenTofu dan Ansible via CI/CD.
