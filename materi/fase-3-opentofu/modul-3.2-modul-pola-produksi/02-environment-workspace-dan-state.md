# 02 — Manajemen Lingkungan: Directory per Environment vs OpenTofu Workspaces

> **Ringkasan:** Mengelola pemisahan lingkungan pengembangan (*dev*), pra-produksi (*staging*), dan produksi (*prod*) adalah salah satu keputusan arsitektural paling krusial di dalam Site Reliability Engineering (SRE). Keputusan ini menentukan seberapa besar radius ledakan (*blast radius*) jika terjadi kesalahan operasional manusia atau kegagalan kode. Dokumen ini membedah perbedaan mendalam antara pola *Directory per Environment* dengan *OpenTofu Workspaces*, mendesain struktur backend remote state yang terisolasi aman di MinIO, menyusun SOP pemeriksaan pra-eksekusi (*preflight context check*), hingga merancang alur promosi kode yang dapat diaudit secara ketat.

---

## 1. Analogi Pemula: Gedung Bersekat Beton vs Papan Tulis Satu Kamar

Untuk memahami bahaya tertukarnya lingkungan produksi dan pengembangan, mari gunakan analogi fasilitas fisik:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI APARTEMEN BETON VS PAPAN TULIS BERSAMA              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PENDEKATAN WORKSPACES: PAPAN TULIS DI SATU RUANGAN BERSAMA ]             │
│  Anda memiliki satu lembar papan tulis di ruang kerja bersama.              │
│  - Di pagi hari, Anda menulis catatan proyek Dev di papan tulis.            │
│  - Di siang hari, Anda ingin menguji proyek Prod, maka Anda menghapus       │
│    catatan Dev dan menggantinya dengan tulisan Prod (**tofu workspace       │
│    select prod**).                                                          │
│  - Masalah: Suatu sore Anda lelah dan lupa spidol mana yang sedang Anda     │
│    pegang. Anda bermaksud menghapus coretan Dev, tetapi yang Anda hapus     │
│    ternyata adalah data transaksi Production! Satu ruangan hancur karena    │
│    tidak ada sekat fisik pemisah!                                           │
│                                                                             │
│  [ PENDEKATAN DIRECTORY PER ENVIRONMENT: TIGA GEDUNG BERSEKAT BETON ]       │
│  Perusahaan membangun 3 gedung fisik terpisah dengan jarak 5 kilometer:     │
│  - **Gedung Dev** (Pintu hijau, kunci biasa, bebas diakses siapapun).       │
│  - **Gedung Staging** (Pintu kuning, dijaga satpam, kartu akses staf).      │
│  - **Gedung Prod** (Pintu baja merah tebal, scanner sidik jari, gembok      │
│    brankas ganda, dijaga ketat 24 jam).                                     │
│  - Jika seorang engineer melakukan kesalahan fatal dan membakar isi Gedung  │
│    Dev, api tidak akan pernah bisa meloncat membakar Gedung Prod karena     │
│    terpisah oleh jarak dan dinding beton kokoh (**Blast Radius Terisolasi**)!│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Environment sebagai Batasan Operasional & Keamanan (Blast Radius)

Di dalam kultur SRE profesional, istilah **Environment** bukanlah sekadar variabel teks pembeda seperti `environment = "prod"`.

Sebuah lingkungan produksi sejati didefinisikan oleh **8 Batasan Operasional Keras (*Hard Boundaries*)**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DELAPAN BATASAN OPERASIONAL SEBUAH ENVIRONMENT              │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Root Module Independen : Memiliki file main.tf dan backend.tf sendiri.   │
│ 2. Backend State Terpisah : Kunci S3 unik (e.g. env/prod/terraform.tfstate) │
│ 3. Akses Kredensial IAM   : Akun service account Dev dilarang menyentuh Prod│
│ 4. Segmentasi Jaringan    : Subnet VLAN / VPC terisolasi fisik secara ketat.│
│ 5. Sizing & Topologi      : Dev cukup 1 VM, Prod wajib Multi-Node HA & BGP. │
│ 6. Kebijakan Keamanan     : Prod wajib proteksi destroy & enkripsi disk.    │
│ 7. Mekanisme Persetujuan  : Dev auto-plan, Prod butuh review CAB / Lead SRE.│
│ 8. Prosedur Pemulihan (DR): RPO/RTO dan runbook pemulihan yang berbeda.     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Komparasi Head-to-Head: Directory per Environment vs OpenTofu Workspaces

OpenTofu memiliki fitur bawaan bernama `tofu workspace` (misal: `tofu workspace new dev`, `tofu workspace select prod`). Meskipun fitur ini terlihat praktis bagi pemula, fitur ini memiliki risiko operasional yang sangat tinggi jika digunakan untuk membedakan lingkungan produksi enterprise.

### Tabel Perbandingan Arsitektur

| Dimensi Parameter | Directory per Environment (Direkomendasikan) | OpenTofu Workspaces |
|---|---|---|
| **Lokasi State File** | **Sangat Eksplisit:** Terlihat jelas pada blok `backend.tf` di setiap folder. | **Tersirat (Implicit):** OpenTofu otomatis menambahkan prefix workspace ke path S3. |
| **Pemisahan Kredensial** | **Tinggi:** Setiap direktori dapat menggunakan profil AWS/MinIO/SSH yang berbeda. | **Rendah:** Berbagi konfigurasi provider dan kredensial yang sama persis di satu folder. |
| **Perbedaan Topologi** | **Sangat Fleksibel:** Prod bisa menggunakan 3 master k3s + MetalLB, Dev cukup 1 Docker. | **Kaku:** Memaksakan kode HCL yang sama persis; perbedaan harus dipaksakan via conditional `if/else`. |
| **Risiko Human Error** | **Sangat Rendah:** Path direktori `environments/prod/` selalu terpampang di prompt shell. | **Sangat Tinggi:** Lupa menjalankan `workspace select` berisiko memusnahkan production! |
| **Integrasi CI/CD & RBAC**| **Sempurna:** Akses folder `environments/prod/` dapat dikunci menggunakan GitLab CODEOWNERS. | **Sulit:** Seluruh pipeline CI/CD harus mengakses file HCL yang sama di root repo. |
| **Tingkat Kesiapan SRE** | **Standar Baku Industri On-Premises & Enterprise.** | Hanya cocok untuk testing *ephemeral feature branch* yang 100% homogen. |

> [!IMPORTANT]
> **Keputusan Arsitektural Bootcamp SRE:**
> Di seluruh kurikulum dan operasional produksi on-premise kita, **KITA SELALU MENGGUNAKAN POLA DIRECTORY PER ENVIRONMENT**. Pemisahan fisik direktori memberikan perlindungan mutlak terhadap kesalahan manusia (*foolproof safety*) dan membatasi *blast radius* secara nyata.

---

## 4. Desain Struktur Direktori Multi-Environment Standar Industri

Berikut adalah struktur kanonikal repositori IaC yang siap diaudit untuk kebutuhan enterprise:

```text
infrastructure-repo/
├── modules/                        # PUSTAKA MODUL REUSABLE (Tanpa Backend/Credential!)
│   ├── web-server/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── network/
│
└── environments/                   # ROOT MODULE PER LINGKUNGAN (Memiliki Backend & State!)
    ├── dev/
    │   ├── backend.tf              # Mengarah ke key: modul-3.2/dev/terraform.tfstate
    │   ├── providers.tf            # Konfigurasi koneksi ke Docker/OrbStack lokal
    │   ├── main.tf                 # Memanggil ../../modules/web-server dengan sizing kecil
    │   ├── variables.tf            # Nilai input khusus dev
    │   └── terraform.tfvars.example
    │
    ├── staging/
    │   ├── backend.tf              # Mengarah ke key: modul-3.2/staging/terraform.tfstate
    │   ├── providers.tf            # Konfigurasi koneksi ke server staging
    │   ├── main.tf                 # Memanggil modul yang sama dengan sizing menengah
    │   └── variables.tf
    │
    └── prod/
        ├── backend.tf              # Mengarah ke key: modul-3.2/prod/terraform.tfstate
        ├── providers.tf            # Konfigurasi koneksi ke server bare-metal produksi
        ├── main.tf                 # Memanggil modul dengan Multi-Node HA & proteksi destroy
        └── variables.tf
```

---

### Perbandingan Konfigurasi Backend: Isolasi Kunci State MinIO

Perhatikan bagaimana setiap lingkungan memiliki file `backend.tf` yang mengarah ke jalur penyimpanan yang berbeda secara tegas di server MinIO:

#### 1. Lingkungan Pengembangan: `environments/dev/backend.tf`
```hcl
terraform {
  required_version = ">= 1.6.0"

  backend "s3" {
    bucket                      = "sre-tofu-state"
    key                         = "modul-3.2/dev/terraform.tfstate" # KUNCI STATE DEV
    region                      = "us-east-1"
    endpoint                    = "http://127.0.0.1:9000"
    use_path_style              = true
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
  }
}
```

#### 2. Lingkungan Produksi: `environments/prod/backend.tf`
```hcl
terraform {
  required_version = ">= 1.6.0"

  backend "s3" {
    bucket                      = "sre-tofu-state"
    key                         = "modul-3.2/prod/terraform.tfstate" # KUNCI STATE PROD (TERPISAH!)
    region                      = "us-east-1"
    endpoint                    = "http://127.0.0.1:9000"
    use_path_style              = true
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
  }
}
```

> [!CAUTION]
> **Bahaya Tabrakan Kunci State (State Key Collision):**
> Jangan pernah menggunakan `key = "terraform.tfstate"` secara seragam di seluruh lingkungan! Jika dua lingkungan memiliki kunci yang sama di dalam bucket yang sama, eksekusi `tofu apply` di lingkungan Dev akan **menimpa (*overwrite*) dan menghancurkan seluruh catatan inventaris lingkungan Produksi**!

---

## 5. SOP Preflight Safety: Mencegah Salah Tembak Lingkungan

Sebelum mengeksekusi perintah yang bersifat mutasi infrastruktur (`tofu apply` atau `tofu destroy`), seorang SRE wajib menjalankan **SOP Preflight Safety Check** di terminal Mac:

```bash
# 1. Pastikan direktori kerja aktif benar-benar berada di lingkungan yang dituju!
$ pwd
/Users/muzma/.../environments/dev   # BUKAN environments/prod!

# 2. Periksa status backend dan provider yang terpasang
$ tofu providers
Providers required by configuration:
.
└── provider[registry.opentofu.org/kreuzwerker/docker] ~> 3.0

# 3. Periksa daftar resource yang saat ini tercatat di dalam state lingkungan tersebut
$ tofu state list
module.web.docker_container.this
module.web.docker_image.this

# 4. Lakukan sanity check: Pastikan tidak ada environment variable prod yang aktif!
$ env | grep -E 'TOFU_|TF_VAR_|AWS_'
AWS_ACCESS_KEY_ID=minioadmin
AWS_SECRET_ACCESS_KEY=minioadmin
```

### Checklist Verifikasi Mental Sebelum Mengetik "yes":
- [ ] Saya berada di direktori `environments/dev/` (Bukan `prod`).
- [ ] Output `tofu plan` hanya menampilkan penambahan/perubahan resource dev.
- [ ] Kolom host dan port yang dibuka tidak bertabrakan dengan layanan lain.
- [ ] Tidak ada simbol `-/+ replace` yang tidak direncanakan pada data penting.

---

## 6. Pola Promosi Lingkungan yang Teruji (The Promotion Pipeline)

Banyak pemula memiliki pemikiran yang keliru mengenai konsep promosi lingkungan:
> *"Apakah promosi itu berarti setelah sukses di dev, saya meng-copy file `terraform.tfstate` dari folder dev ke folder prod?"*

Ini adalah **kesalahan paling fatal yang akan langsung memicu pemecatan di dunia profesional!** File state terikat permanen pada ID objek fisik di masing-masing lingkungan. Menyalin file state dev ke prod akan menyebabkan OpenTofu mengira kontainer di dev adalah kontainer di prod, lalu menghancurkannya!

### Definisi Promosi yang Benar dalam Kultur SRE:
**Promosi adalah proses menguji VERSI KODE DAN MODUL YANG SAMA secara bertahap dari Dev ──► Staging ──► Prod, di mana masing-masing lingkungan MENGEKSEKUSI PLAN DENGAN FILE STATE DAN VARIABEL MASING-MASING!**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR PROMOSI GITOPS ENTERPRISE TERUKUR                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TAHAP 1: LINGKUNGAN PENGEMBANGAN (DEV) ]                                 │
│  - Engineer membuat branch Git: feature/upgrade-nginx-1.27                  │
│  - Diuji di folder: environments/dev/                                       │
│  - Eksekusi: $ tofu plan && tofu apply                                      │
│  - Validasi: Uji fungsionalitas aplikasi dan HTTP smoke test sukses.        │
│                                                                             │
│       │ Buat Pull/Merge Request & Peer Review oleh rekan tim SRE            │
│       ▼ Merge ke branch 'staging'                                           │
│                                                                             │
│  [ TAHAP 2: LINGKUNGAN PRA-PRODUKSI (STAGING) ]                             │
│  - Pipeline CI otomatis masuk ke folder: environments/staging/              │
│  - Menghasilkan Plan Staging: $ tofu plan -out=staging.tfplan               │
│  - Review Plan Staging oleh QA & SRE. Jika disetujui: $ tofu apply          │
│  - Dilakukan uji beban (Load Test) dan simulasi failover jaringan.         │
│                                                                             │
│       │ Validasi Staging 100% Lolos tanpa regresi                           │
│       ▼ Buat Release Tag Git (misal v1.4.0) & Merge ke 'main'               │
│                                                                             │
│  [ TAHAP 3: LINGKUNGAN PRODUKSI (PROD) ]                                    │
│  - Pipeline CD masuk ke folder: environments/prod/                          │
│  - Menghasilkan Plan Prod: $ tofu plan -out=prod.tfplan                     │
│  - **APPROVAL GATE KETAT:** Membutuhkan tanda tangan digital Lead SRE!      │
│  - Eksekusi pada Jendela Pemeliharaan (Maintenance Window): $ tofu apply    │
│  - Sistem Observabilitas memantau metrik SLO / Error Budget burn rate.      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Gejala Masalah | Titik Investigasi | Penyebab Akar (*Root Cause*) | Solusi Pemulihan SRE |
|---|---|---|---|
| `State file kosong atau semua resource hilang` | `tofu state list` | Anda mengeksekusi perintah di direktori atau workspace yang salah, atau kunci backend tertukar. | Jangan langsung apply! Jalankan `pwd` dan periksa `key` di `backend.tf` apakah sudah sesuai dengan lingkungan Anda. |
| `Error: Backend configuration changed` | `tofu plan` | Anda baru saja mengedit nilai `key` atau `bucket` di file `backend.tf`. | Jalankan `tofu init -migrate-state` jika memang ingin memindahkan data, atau kembalikan nama key lama jika salah ketik. |
| `Plan di dev mencoba menghapus resource di prod` | Output `tofu plan` | File `main.tf` di dev memanggil modul remote atau provider alias yang secara tidak sengaja mengarah ke server produksi. | Hentikan proses! Periksa kembali variabel koneksi provider dan pastikan isolasi endpoint benar-benar terputus. |
| `Error: Bucket does not exist saat init dev` | `tofu init` | Bucket penyimpanan remote state di server MinIO belum dibuat sebelum inisialisasi dijalankan. | Buka antarmuka MinIO Console lokal di `http://127.0.0.1:9001` dan buat bucket `sre-tofu-state`. |

---

## 8. Ringkasan Konsep Kunci (Cheat Sheet)

```bash
# 1. Selalu lakukan verifikasi direktori sebelum mengeksekusi apa pun
pwd && git branch --show-current

# 2. Melihat seluruh status provider yang terhubung di lingkungan aktif
tofu providers

# 3. Menyimpan rencana eksekusi khusus lingkungan dev
tofu plan -out=dev.tfplan

# 4. Menerapkan rencana yang telah dikunci
tofu apply dev.tfplan

# 5. Membersihkan cache inisialisasi jika terjadi pergantian backend
rm -rf .terraform/ .terraform.lock.hcl && tofu init
```

---

## 9. Cek Pemahaman Mandiri

1. Mengapa standar industri SRE kelas enterprise lebih merekomendasikan penggunaan *Directory per Environment* dibandingkan menggunakan *OpenTofu Workspaces* untuk memisahkan lingkungan produksi?
2. Jelaskan konsep *Blast Radius* dalam arsitektur infrastruktur! Bagaimana struktur direktori terpisah dapat mencegah kesalahan di lingkungan *dev* merusak lingkungan *prod*?
3. Apa yang akan terjadi jika file `environments/dev/backend.tf` dan file `environments/prod/backend.tf` sama-sama mengonfigurasi `key = "terraform.tfstate"` pada bucket MinIO yang sama?
4. Jelaskan mengapa menyalin file `terraform.tfstate` dari folder pengembangan (*dev*) ke folder produksi (*prod*) adalah tindakan yang salah fatal! Apa definisi promosi lingkungan yang benar menurut prinsip SRE?
5. Sebutkan minimal 4 pemeriksaan wajib dalam *SOP Preflight Safety Check* sebelum seorang operator mengeksekusi `tofu apply` pada lingkungan produksi!

---

*Langkah Selanjutnya:* Masuk ke [03 — for_each, count, Conditional, dan Data Source](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.2-modul-pola-produksi/03-foreach-count-conditional-data-source.md) untuk mempelajari teknik instansiasi multi-resource tingkat lanjut tanpa memicu fenomena pergeseran indeks (*index churn*), merancang kondisi bersyarat, dan memahami batas kepemilikan *Data Source*.
