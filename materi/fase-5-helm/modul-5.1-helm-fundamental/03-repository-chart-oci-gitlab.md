# 03 — Chart Repository dan OCI di GitLab: Mendistribusikan Paket Berstandar Kontainer Modern

> **Ringkasan:** Di era awal Kubernetes, mendistribusikan Helm Chart memerlukan server web statis yang memelihara berkas indeks monolitik (`index.yaml`). Arsitektur klasik ini lambat, rentan terhadap tabrakan konkurensi, dan memisahkan manajemen paket aplikasi dari gambar kontainer. Sejak Helm versi 3.8+, standar **Open Container Initiative (OCI)** diadopsi secara penuh, memungkinkan Helm Chart dikemas, diberi tanda tangan digital (*signed*), dan disimpan langsung di dalam Container Registry (seperti **GitLab Container Registry** atau **Harbor**) berdampingan dengan Docker Image. Dokumen ini membedah arsitektur OCI Artifacts, alur pengemasan dan publikasi chart, tata kelola versi (*SemVer & Provenance*), serta proteksi kredensial pipa CI/CD.

---

## 1. Analogi Pemula: Gudang File ZIP Konvensional vs Pelabuhan Peti Kemas OCI

Untuk memahami mengapa industri beralih dari repositori Helm klasik ke standar OCI, mari gunakan analogi logistik pengiriman barang:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI REPOSITORI HELM KLASIK vs OCI REGISTRY              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PENDEKATAN KLASIK (HELM V1/V2): BUKU BESAR BUKTI GUDANG FISIK ]          │
│  - Masalah: Anda menyimpan file `.tgz` di server web biasa (Nginx / S3).    │
│  - Syarat Mutlak: Harus ada satu berkas teks raksasa bernama `index.yaml`   │
│    yang mencatat daftar seluruh chart dan link unduhannya.                  │
│  - Malapetaka SRE:                                                          │
│    1. Jika ada 10 engineer melakukan rilis bersamaan, file `index.yaml`     │
│       akan bentrok (*race condition / corrupted index*).                    │
│    2. Jika chart berjumlah ribuan, file `index.yaml` membengkak hingga      │
│       ratusan Megabyte! Perintah `helm repo update` menjadi sangat lemot.   │
│                                                                             │
│  [ PENDEKATAN MODERN (STANDAR OCI): PELABUHAN PETI KEMAS INTERNASIONAL ]    │
│  - Pelabuhan peti kemas (Container Registry) tidak peduli apa isi di dalam  │
│    sebuah peti kemas standar:                                               │
│    - Peti A berisi Mobil Mewah  -> **Docker Container Image**               │
│    - Peti B berisi Buku Manual  -> **Helm Chart Package**                   │
│  - Protokol pengiriman seragam: `oci://registry.gitlab.com/...`             │
│  - Keunggulan Mutlak:                                                       │
│    1. Tanpa File Indeks Monolitik: Setiap chart memiliki alamat independen. │
│    2. Satu Sistem Keamanan: Akses kontrol, autentikasi token, dan audit log │
│       disatukan di Container Registry yang sama dengan Docker Image!        │
│    3. Verifikasi Integritas: Setiap chart dilindungi oleh *SHA256 Digest*   │
│       kriptografis yang tidak bisa dipalsukan (*Tamper-Proof*).             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Perbandingan Arsitektur: HTTP Repository vs OCI Registry

| Dimensi Arsitektur | Repositori Helm Klasik (HTTP/S3) | Repositori Modern Berstandar OCI |
|---|---|---|
| **Protokol Transmisi** | `http://` atau `https://` | `oci://` (Open Container Initiative) |
| **Penyimpanan Metadata** | File tunggal terpusat `index.yaml` | Manifest OCI independen per paket tag |
| **Infrastruktur Server** | Web server statis, ChartMuseum, S3 | Docker Registry, GitLab Registry, Harbor, ECR |
| **Skalabilitas Paket** | Lambat ketika jumlah versi > 1.000 | Sangat cepat dan terdistribusi secara global |
| **Mekanisme Otentikasi** | Basic Auth HTTP atau token kustom | Standar Docker Registry Auth (OAuth2 / PAT / CI Token) |
| **Integritas Objek** | Checksum SHA256 manual di file index | Kriptografi *Content-Addressable* Image Digest (`sha256:...`) |
| **Pemindaian Keamanan** | Sulit diintegrasikan dengan scanner container | Otomatis dipindai kerentanannya oleh Trivy / Clair |

---

## 3. Alur Kerja Pengemasan & Publikasi OCI (The OCI Workflow)

Berikut adalah 4 langkah operasional standar untuk memaketkan dan mempublikasikan Chart ke GitLab Container Registry:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ALUR KERJA OCI HELM LENGKAP                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Direktori Kode Chart: `internal-app/`                                   │
│            │                                                                │
│            ▼ `helm package ./internal-app --version 1.2.0`                  │
│  2. Berkas Arsip Terkompresi: `internal-app-1.2.0.tgz`                      │
│            │                                                                │
│            ▼ `helm registry login registry.gitlab.com -u ...`               │
│  3. Sesi Autentikasi Terverifikasi                                          │
│            │                                                                │
│            ▼ `helm push internal-app-1.2.0.tgz oci://registry.gitlab.com/...`│
│  4. Tersimpan Aman di OCI Registry (Artifact Digest: sha256:7f3b...)        │
│            │                                                                │
│            ▼ `helm install payment oci://registry.gitlab.com/...`           │
│  5. Deploy Langsung ke Klaster Kubernetes k3s                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Langkah 1: Pengemasan Chart Lokal (`helm package`)
Jalankan pengemasan dari direktori root chart dengan menyertakan versi rilis SemVer:
```bash
# Validasi sintaksis sebelum dibungkus
helm lint ./internal-app

# Mengemas direktori menjadi tarball .tgz
helm package ./internal-app \
  --version 1.2.0 \
  --app-version 2.4.1 \
  --destination ./dist/
```
*Output:* `Successfully packaged chart and saved it to: ./dist/internal-app-1.2.0.tgz`

### Langkah 2: Autentikasi Aman ke OCI Registry
Gunakan selalu `--password-stdin` untuk mencegah kata sandi atau Personal Access Token (PAT) terekam di riwayat shell terminal (`~/.bash_history`):

```bash
# Autentikasi ke GitLab OCI Container Registry
echo "$GITLAB_TOKEN" | helm registry login registry.gitlab.com \
  --username "sre-automator" \
  --password-stdin
```
*Output:* `Login Succeeded`

### Langkah 3: Mengunggah Paket ke OCI Registry (`helm push`)
Format URI wajib diawali dengan skema `oci://` dan **TIDAK MENYERTAKAN NAMA CHART DI AKHIR URL** (nama chart akan dibaca otomatis dari metadata paket):

```bash
# Mengunggah arsip .tgz ke GitLab Registry
helm push ./dist/internal-app-1.2.0.tgz oci://registry.gitlab.com/bootcamp-sre/payment-platform/charts
```
*Output yang Dihasilkan:*
```text
Pushed: registry.gitlab.com/bootcamp-sre/payment-platform/charts/internal-app:1.2.0
Digest: sha256:8f9b2c1743a19e24f706e25776d6ec1438914619736c4b2b1a8d0e5124fa68b1
```

### Langkah 4: Menarik atau Menginstal Langsung dari OCI Registry
Anda tidak perlu lagi menjalankan `helm repo add` atau `helm repo update`! Helm bisa langsung berinteraksi dengan registry OCI secara native:

```bash
# Opsi A: Mengunduh paket .tgz dari OCI Registry untuk diinspeksi
helm pull oci://registry.gitlab.com/bootcamp-sre/payment-platform/charts/internal-app \
  --version 1.2.0

# Opsi B: Menginstal aplikasi langsung ke klaster k3s dari OCI Registry
helm install payment oci://registry.gitlab.com/bootcamp-sre/payment-platform/charts/internal-app \
  --version 1.2.0 \
  --namespace production \
  --values values-prod.yaml \
  --wait
```

---

## 4. Tata Kelola Versi & Rantai Asal-Usul (Provenance & SemVer)

Dalam tata kelola rekayasa keandalan (*SRE Governance*), integritas rantai pasok perangkat lunak (*Supply Chain Security*) mewajibkan paket rilis memiliki **Jejak Asal-Usul (*Provenance*)** yang tak terbantahkan:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 RANTAI ASAL-USUL PAKET OCI (PROVENANCE CHAIN)               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ Git Commit SHA ]               Contoh: git commit 8f9b2c1...             │
│         │                                                                   │
│         ▼                                                                   │
│  [ Chart Semantic Version ]       Contoh: version 1.2.0                     │
│         │                                                                   │
│         ▼                                                                   │
│  [ Immutable Container Image ]    Contoh: payment-api@sha256:4a8b...        │
│         │                                                                   │
│         ▼                                                                   │
│  [ OCI Chart Package Digest ]     Contoh: internal-app@sha256:8f9b...       │
│         │                                                                   │
│         ▼                                                                   │
│  [ Kubernetes Release Revision ]  Contoh: payment-prod Rev 4                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Aturan Immutability (Larangan Menimpa Versi)
Di lingkungan produksi: **TAG VERSI CHART DI OCI REGISTRY BERSIFAT IMMUTABLE (TIDAK BOLEH DITIMPA)!**
- Jika Anda telah mempublikasikan versi `1.2.0`, lalu menemukan ada kesalahan konfigurasi kecil pada template, **DILARANG MENGUNGGAH ULANG KE VERSI 1.2.0**!
- Naikkan nomor patch ke versi **`1.2.1`**.
- Menimpa versi yang sama akan merusak determinisme klaster: klaster yang melakukan reboot di masa depan akan menarik paket yang berbeda dari saat rilis pertama kali disetujui!

---

## 5. Integrasi Otomatisasi Pipeline GitLab CI (`.gitlab-ci.yml`)

Berikut adalah template pipeline otomatisasi yang memvalidasi, mengemas, dan mempublikasikan Chart ke GitLab Package Registry secara aman:

```yaml
stages:
  - lint_and_test
  - package_and_publish

variables:
  CHART_DIR: "internal-app"
  OCI_REGISTRY: "oci://${CI_REGISTRY}/${CI_PROJECT_PATH}/charts"

lint_chart:
  stage: lint_and_test
  image: alpine/helm:latest
  script:
    - helm lint ${CHART_DIR}
    - helm template test-render ${CHART_DIR} --debug

publish_oci_chart:
  stage: package_and_publish
  image: alpine/helm:latest
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v[0-9]+\.[0-9]+\.[0-9]+$/' # Hanya jalan saat ada Git Tag rilis resmi!
  before_script:
    # Autentikasi non-interaktif menggunakan CI_JOB_TOKEN bawaan GitLab
    - echo "$CI_JOB_TOKEN" | helm registry login ${CI_REGISTRY} -u gitlab-ci-token --password-stdin
  script:
    # Ekstrak versi dari Git Tag (misal v1.2.0 -> 1.2.0)
    - CHART_VERSION=$(echo ${CI_COMMIT_TAG} | sed 's/^v//')
    - mkdir -p ./dist
    - helm package ${CHART_DIR} --version ${CHART_VERSION} --destination ./dist
    - helm push ./dist/*.tgz ${OCI_REGISTRY}
```

> [!CAUTION]
> **Kebersihan Artefak CI (Artifact Hygiene):**
> Jangan pernah mengunggah log debugging mentah (`helm template --debug` atau `helm get all`) sebagai artefak publik pipa CI/CD jika di dalamnya terdapat objek Secret, kata sandi database, atau kunci privat. Simpan hanya nomor ringkasan digest OCI (`sha256:...`) sebagai bukti audit.

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA MASALAH OCI REGISTRY                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "401 UNAUTHORIZED" SAAT MELAKUKAN HELM PUSH ]                   │
│  Gejala: `helm push` gagal dengan pesan penolakan autentikasi.              │
│  Akar Masalah:                                                              │
│  - Token `CI_JOB_TOKEN` atau Personal Access Token (PAT) tidak memiliki hak │
│    akses `write_registry` atau peran pengguna adalah *Guest/Reporter*.      │
│  Solusi SRE:                                                                │
│  1. Pastikan token memiliki scope: `read_registry` dan `write_registry`.    │
│  2. Di pengaturan proyek GitLab: aktifkan izin Container Registry read/write│
│     pada *Settings -> CI/CD -> Token Access*.                               │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "NAME MUST MATCH [a-z0-9-_.]+" (INVALID OCI URI) ]              │
│  Gejala: Helm menolak URL saat menjalankan perintah `helm push`.            │
│  Akar Masalah:                                                              │
│  - Anda menyertakan nama file arsip atau nama chart di ujung URL push:      │
│    `helm push app.tgz oci://registry.../charts/internal-app` (SALAH!)       │
│  Solusi SRE:                                                                │
│  URL OCI untuk perintah push HANYA boleh mengarah ke direktori repositori:  │
│  helm push app.tgz oci://registry.../charts (BENAR!)                        │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: TAG REPO DITOLAK KARENA ATURAN IMMUTABILITY ]                   │
│  Gejala: Registry menolak push: "tag already exists and is immutable".      │
│  Akar Masalah:                                                              │
│  - Anda mencoba mengunggah chart dengan versi `1.2.0` yang sudah pernah     │
│    terdaftar sebelumnya di registry GitLab yang terproteksi.                │
│  Solusi SRE:                                                                │
│  1. Hormati aturan immutability! Jangan hapus tag lama di registry!        │
│  2. Naikkan nomor versi di `Chart.yaml` menjadi `1.2.1` lalu kemas ulang.   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Login ke OCI registry secara aman via stdin
echo "$REGISTRY_TOKEN" | helm registry login registry.internal.corp -u sre-bot --password-stdin

# 2. Mengemas direktori chart menjadi file arsip tarball terkompresi
helm package ./internal-app --version 1.2.0 --destination ./dist/

# 3. Mengunggah paket chart lokal ke registry OCI
helm push ./dist/internal-app-1.2.0.tgz oci://registry.internal.corp/charts

# 4. Mengunduh paket chart dari registry OCI tanpa menginstall
helm pull oci://registry.internal.corp/charts/internal-app --version 1.2.0

# 5. Menampilkan manifest lengkap dari chart di OCI registry secara remote
helm show chart oci://registry.internal.corp/charts/internal-app --version 1.2.0
helm show values oci://registry.internal.corp/charts/internal-app --version 1.2.0

# 6. Logout dan membersihkan token sesi OCI dari mesin lokal
helm registry logout registry.internal.corp
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Apa keunggulan arsitektural utama mendistribusikan Helm Chart menggunakan standar **Open Container Initiative (OCI)** dibandingkan repositori Helm klasik berbasis web server statis?
- A. Chart OCI otomatis diterjemahkan menjadi bahasa mesin biner C++.
- B. Standar OCI mengeliminasi ketergantungan pada file indeks monolitik (`index.yaml`) yang rawan *race condition*, serta menyatukan autentikasi, pemindaian keamanan kerentanan, dan penyimpanan artifak di Container Registry yang sama dengan Docker Image.
- C. OCI memungkinkan Helm berjalan tanpa perlu menginstal Kubernetes.
- D. Ukuran file `.tgz` otomatis berkurang 99%.

*Kunci Jawaban:* **B**. Standar OCI memperlakukan Helm Chart sebagai *first-class citizen artifact* di dalam container registry modern dengan jaminan integritas kriptografis dan tanpa bottleneck file indeks tunggal.

### Soal 2 (Pilihan Ganda)
Perhatikan perintah berikut:
`echo "$PASSWORD" | helm registry login registry.gitlab.com -u "$USER" --password-stdin`
Mengapa teknik pemberian kata sandi melalui pipa *stdin* (`--password-stdin`) diwajibkan dalam standar operasional SRE?
- A. Karena Helm menolak input kata sandi yang memiliki panjang di atas 8 karakter.
- B. Mencegah kata sandi rahasia terekam secara *plaintext* di dalam riwayat terminal (`~/.bash_history`) atau terbaca oleh pengguna lain melalui utilitas pemantau proses sistem (`ps aux`).
- C. Agar koneksi internet dialihkan melalui VPN otomatis.
- D. Untuk mengenkripsi harddisk laptop.

*Kunci Jawaban:* **B**. Memasukkan kata sandi langsung sebagai flag baris perintah (`--password rahasia`) membocorkan kredensial ke process table dan history file sistem operasi.

### Soal 3 (Pilihan Ganda)
Saat mempublikasikan chart menggunakan perintah `helm push`, bagaimana format URI target yang benar?
- A. `helm push app.tgz https://registry.gitlab.com/charts/app.tgz`
- B. `helm push app.tgz oci://registry.gitlab.com/bootcamp-sre/charts`
- C. `helm push app.tgz ftp://registry.gitlab.com/charts`
- D. `helm push app.tgz s3://my-bucket/charts`

*Kunci Jawaban:* **B**. Perintah `helm push` OCI menggunakan skema `oci://` dan mengarah ke namespace repositori registry tanpa menyertakan nama file atau nama chart di akhir URI.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Seorang engineer memodifikasi template `service.yaml` di repositori Git, lalu mengunggah paket chart baru ke OCI Registry dengan tetap menggunakan nomor versi yang sama: `v1.0.0`. Akibatnya, server staging menarik versi baru, sementara server disaster recovery yang menarik chart 2 hari lalu masih menjalankan versi lama dengan nama versi yang sama persis (`v1.0.0`), memicu kebingungan audit investigasi insiden. Jelaskan mengapa tindakan menimpa versi (*version overriding*) melanggar prinsip *Immutable Infrastructure*, dan bagaimana menyusun aturan tata kelola versi yang benar!

*Panduan Solusi:*
1. **Analisis Pelanggaran Prinsip *Immutable Infrastructure*:**
   Menimpa artefak rilis dengan tag versi yang sama menghancurkan sifat *determinisme* dan *reproducibility*. SRE tidak lagi dapat memastikan bahwa kode yang berjalan di staging identik 100% dengan kode di disaster recovery hanya dengan melihat label `v1.0.0`. Hal ini merusak validitas rantai bukti audit (*broken provenance*) dan membuat investigasi insiden menjadi bias.
2. **Tata Kelola Versi Baku (Governance Best Practice):**
   - **Kunci Immutability di Registry:** Aktifkan fitur *Tag Immutability* pada GitLab Container Registry atau Harbor, sehingga perintah push yang menggunakan tag yang sudah ada akan ditolak otomatis dengan error `HTTP 409 Conflict`.
   - **Penerapan Semantic Versioning (SemVer) Disiplin:**
     Setiap perubahan pada isi chart WAJIB menaikkan nomor versi di `Chart.yaml`:
     - Perbaikan bug kecil / patch template -> Naikkan **PATCH** (`1.0.0` -> `1.0.1`).
     - Penambahan fitur baru yang *backward-compatible* -> Naikkan **MINOR** (`1.0.0` -> `1.1.0`).
     - Perubahan arsitektur besar / *breaking changes* -> Naikkan **MAJOR** (`1.0.0` -> `2.0.0`).
   - Dengan aturan ini, setiap paket di OCI Registry dijamin bersifat unik, abadi, dan dapat diaudit secara presisi kapan pun dibutuhkan.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Struktur Chart, Values, dan Go-template](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/01-struktur-chart-values-template.md) — Mengemas struktur direktori Chart.
  - [02 — Render dan Lifecycle Release](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/02-render-install-upgrade-rollback.md) — Mengelola siklus hidup rilis lokal dan rollback.
- **Tahap Berikutnya:**
  - [Modul 5.2 — Chart Tingkat Produksi](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/) — Merancang custom chart dengan Hooks, Tests, dan Security Context tingkat lanjut.
  - [Fase 6 — GitOps dengan ArgoCD](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/) — Menghubungkan paket OCI Helm Chart ini ke dalam rekonsiliasi otomatis ArgoCD.
