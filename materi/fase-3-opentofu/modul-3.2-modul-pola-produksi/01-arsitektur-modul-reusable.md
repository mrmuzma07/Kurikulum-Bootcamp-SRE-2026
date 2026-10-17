# 01 — Arsitektur Modul Reusable: Membangun Komponen Infrastruktur Modular dan Teruji

> **Ringkasan:** Di lingkungan produksi enterprise, menduplikasi kode HCL dengan cara *copy-paste* antar-folder adalah dosa besar arsitektur yang memicu pergeseran konfigurasi (*configuration drift*) dan kerentanan keamanan. Dokumen ini membedah prinsip perancangan modul reusable, memisahkan peran antara *Root Module* (caller) dan *Child Module* (provider kapabilitas), merancang kontrak input/output yang memiliki validasi tipe data ketat, memahami mekanisme penerusan provider (*provider passing & aliasing*), hingga menghindari jebakan modul raksasa (*The God Module Anti-Pattern*).

---

## 1. Analogi Pemula: Komponen Rakitan Pabrik vs Campuran Semen Manual

Bagi seorang pemula, membagi konfigurasi OpenTofu ke dalam modul-modul terpisah sering kali dianggap merepotkan:
> *"Kenapa saya tidak menuliskan seluruh resource kontainer, network, dan volumenya langsung di dalam satu file `main.tf` saja? Toh sama-sama jalan!"*

Mari kita gunakan analogi proyek konstruksi sipil:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PERAKITAN PABRIK VS SEMEN MANUAL                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PENDEKATAN TANPA MODUL: CAMPURAN SEMEN MANUAL DI TIAP LANTAI ]           │
│  Tukang bangunan mencampur pasir, semen, dan air secara manual dengan       │
│  takaran kira-kira di setiap lantai gedung (Copy-paste kode HCL).           │
│  - Masalah: Lantai 1 campurannya kuat, Lantai 2 terlalu encer, Lantai 3     │
│    lupa diberi kerikil besi.                                                │
│  - Ketika dinas tata kota memperbarui standar gempa (ada patch keamanan),   │
│    Anda harus mendatangi 50 lantai satu per satu dan membongkar temboknya   │
│    secara manual. Melelahkan, rawan salah, dan lambat!                      │
│                                                                             │
│  [ PENDEKATAN DENGAN MODUL: PANEL DINDING PRE-FABRIKASI PABRIK ]            │
│  Pabrik beton membuat cetakan panel dinding standar bersertifikasi ISO      │
│  (**Child Module**). Panel ini memiliki ukuran presisi, lubang stopkontak   │
│  yang rapi, dan uji ketahanan gempa resmi (**Input/Output Contract**).      │
│  - Kontraktor di lapangan (**Root Module**) cukup memesan:                   │
│    "Kirim 10 panel dinding untuk Lantai Dev, dan 50 panel untuk Lantai Prod"│
│  - Jika ada pembaruan standar isolasi panas, pabrik cukup memperbarui       │
│    cetakannya (versi modul dinaikkan). Seluruh lantai otomatis mendapatkan   │
│    standar kualitas terbaik yang seragam!                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Pemisahan Peran: Root Module vs Child Module

Di dalam OpenTofu, pemisahan tanggung jawab (*separation of concerns*) antara pemanggil (*caller*) dan modul yang dipanggil (*callee*) harus didefinisikan secara tegas:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ARSITEKTUR HIERARKI MODUL OPENTOFU                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ ROOT MODULE / CALLER ]                                                   │
│  Lokasi: environments/dev/main.tf atau environments/prod/main.tf            │
│  Peran & Tanggung Jawab:                                                    │
│  ├── 1. Menentukan Backend Remote State (Bucket S3 MinIO per lingkungan)   │
│  ├── 2. Mengonfigurasi kredensial & endpoint Provider (Docker/Proxmox)      │
│  ├── 3. Menentukan ukuran kapasitas (e.g. port, replica, ukuran memori)     │
│  └── 4. Menyetujui rencana perubahan (tofu plan & apply)                   │
│                                                                             │
│       │ Memanggil via blok module { source = "../../modules/web-server" }   │
│       ▼ Meneruskan nilai variabel & konfigurasi provider                    │
│                                                                             │
│  [ CHILD MODULE / CAPABILITY PROVIDER ]                                     │
│  Lokasi: modules/web-server/                                                │
│  Peran & Tanggung Jawab:                                                    │
│  ├── 1. Menolak input yang tidak valid via blok validation {}               │
│  ├── 2. Menciptakan resource fisik (e.g. docker_container, docker_image)   │
│  ├── 3. Menerapkan label kepemilikan dan standar keamanan seragam           │
│  └── 4. Mengembalikan metadata minimum yang dibutuhkan via outputs.tf       │
│  Catatan: DILARANG KERAS MENENTUKAN BACKEND ATAU CREDENTIAL DI SINI!        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Merancang Kontrak Antarmuka Modul (Module API Contract)

Sebuah modul yang baik harus diperlakukan seperti **Library Perangkat Lunak atau REST API**. Modul tersebut harus memiliki kontrak masukan (*Input Contract*) dan kontrak keluaran (*Output Contract*) yang terdokumentasi dan memiliki tipe data ketat (*strongly typed*).

### 1. Kontrak Masukan: `modules/web-server/variables.tf`
Gunakan blok `validation {}` untuk membatasi nilai yang diperbolehkan masuk ke dalam modul Anda:

```hcl
variable "container_name" {
  description = "Nama unik kontainer web yang akan dibuat."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{2,62}$", var.container_name))
    error_message = "Nama kontainer harus berupa karakter alfanumerik huruf kecil dan tanda hubung (-), panjang 3-63 karakter."
  }
}

variable "image_tag" {
  description = "Tag versi image NGINX yang telah diaudit oleh tim keamanan."
  type        = string
  default     = "1.27.4-alpine"

  validation {
    condition     = !endswith(var.image_tag, ":latest") && !endswith(var.image_tag, "latest")
    error_message = "DILARANG menggunakan tag 'latest'! Gunakan versi semver yang stabil untuk menjamin idempotency."
  }
}

variable "host_port" {
  description = "Port pada host Mac/server fisik yang akan diekspos."
  type        = number

  validation {
    condition     = var.host_port >= 1024 && var.host_port <= 65535
    error_message = "Port host harus berada pada rentang port non-privileged (1024-65535)."
  }
}

variable "environment" {
  description = "Identitas lingkungan deployment (dev, staging, prod)."
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment hanya boleh bernilai: dev, staging, atau prod."
  }
}

variable "custom_labels" {
  description = "Label tambahan dari caller untuk pelacakan biaya dan kepemilikan."
  type        = map(string)
  default     = {}
}
```

---

### 2. Logika Pembuatan Resource: `modules/web-server/main.tf`
Modul mengelola resource secara mandiri berdasarkan variabel yang masuk:

```hcl
# Deklarasikan provider apa yang dibutuhkan modul ini (tanpa mengonfigurasi endpoint!)
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = ">= 3.0.0"
    }
  }
}

locals {
  # Standarisasi label kepemilikan otomatis di setiap kontainer
  default_labels = {
    "managed_by"  = "opentofu"
    "module"      = "web-server"
    "environment" = var.environment
  }
  merged_labels = merge(local.default_labels, var.custom_labels)
}

resource "docker_image" "this" {
  name         = "nginx:${var.image_tag}"
  keep_locally = true
}

resource "docker_container" "this" {
  name  = var.container_name
  image = docker_image.this.image_id

  ports {
    internal = 80
    external = var.host_port
  }

  dynamic "labels" {
    for_each = local.merged_labels
    content {
      label = labels.key
      value = labels.value
    }
  }
}
```

---

### 3. Kontrak Keluaran: `modules/web-server/outputs.tf`
Kembalikan hanya informasi yang benar-benar berguna bagi pemanggil (*caller*). Hindari mengekspos seluruh atribut internal objek yang tidak perlu:

```hcl
output "container_id" {
  description = "ID unik kontainer yang dihasilkan oleh runtime Docker."
  value       = docker_container.this.id
}

output "container_name" {
  description = "Nama resmi kontainer yang sedang berjalan."
  value       = docker_container.this.name
}

output "http_endpoint" {
  description = "URL lengkap untuk pengujian konektivitas HTTP."
  value       = "http://127.0.0.1:${var.host_port}"
}

output "metadata" {
  description = "Metadata minimum untuk diserahkan ke tahap konfigurasi berikutnya (Ansible)."
  value = {
    name        = docker_container.this.name
    port        = var.host_port
    environment = var.environment
  }
}
```

---

## 4. Mekanisme Penerusan Provider (Provider Passing & Aliasing)

Salah satu kesalahan paling fatal yang sering dilakukan oleh pemula saat membuat modul adalah **menaruh blok `provider "docker" { ... }` di dalam child module**.

> [!CAUTION]
> **Larangan Keras Arsitektur Modul:**
> **JANGAN PERNAH menaruh blok konfigurasi provider di dalam child module!**
> Jika child module mengonfigurasi provider sendiri, modul tersebut akan terkunci secara permanen ke satu endpoint dan tidak akan pernah bisa digunakan kembali untuk kluster atau kredensial lain. Konfigurasi provider **HANYA BOLEH BERADA DI ROOT MODULE**.

### Bagaimana Root Module Meneruskan Provider ke Child Module?

Misalkan di root module kita memiliki dua endpoint Docker daemon yang berbeda (misal: Docker lokal di Mac dan Docker di server remote), kita menggunakan fitur **Provider Alias**:

```hcl
# environments/dev/main.tf (ROOT MODULE)

terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

# Provider Default (OrbStack Lokal)
provider "docker" {
  alias = "local_orb"
  host  = "unix:///var/run/docker.sock"
}

# Memanggil Child Module dengan meneruskan provider spesifik
module "frontend_web" {
  source = "../../modules/web-server"

  # INI KUNCINYA: Meneruskan provider alias ke modul anak
  providers = {
    docker = docker.local_orb
  }

  container_name = "dev-frontend-web"
  image_tag      = "1.27.4-alpine"
  host_port      = 18080
  environment    = "dev"

  custom_labels = {
    "team" = "frontend-ui"
  }
}
```

---

## 5. Komposisi Modul (Module Composition): Menghubungkan Output ke Input

Di dunia nyata, infrastruktur tersusun dari gabungan banyak modul independen (misalnya: Modul Jaringan + Modul Web Server + Modul Database).

Bagaimana cara menghubungkan modul-modul ini secara elegan tanpa hardcode? Menggunakan **Module Composition**:

```text
┌─────────────────────────┐               ┌─────────────────────────┐
│     MODULE: NETWORK     │               │    MODULE: WEB-SERVER   │
│  (modules/network/)     │               │  (modules/web-server/)  │
├─────────────────────────┤               ├─────────────────────────┤
│ Output:                 │               │ Input:                  │
│ network_name = "app-net"│ ────────────► │ network_name = ...      │
└─────────────────────────┘   Dihubungkan └─────────────────────────┘
                               di Root
```

```hcl
# environments/dev/main.tf

# 1. Modul Jaringan membuat Network terisolasi
module "app_network" {
  source       = "../../modules/network"
  network_name = "dev-application-network"
}

# 2. Modul Web Server menerima nama network dari output modul jaringan!
module "api_server" {
  source         = "../../modules/web-server"
  container_name = "dev-api-server"
  host_port      = 18081
  environment    = "dev"

  # MENGHUBUNGKAN OUTPUT MODUL LAIN SEBAGAI INPUT (IMPLICIT DEPENDENCY):
  network_name   = module.app_network.network_name
}
```

OpenTofu akan secara otomatis membaca graf dependensi: modul `app_network` akan dibuat terlebih dahulu, output namanya diambil, lalu disuntikkan ke dalam pembuatan kontainer `api_server` tanpa Anda perlu menuliskan `depends_on`.

---

## 6. Pola Versioning & Siklus Rilis Modul

Ketika modul digunakan oleh banyak tim di sebuah perusahaan, mengubah kode modul lokal secara langsung di branch utama dapat merusak infrastruktur tim lain yang sedang berjalan (*breaking changes*).

Standar enterprise menerapkan **Semantic Versioning (SemVer)** menggunakan Git Tag:

```text
  Format SemVer: vMAJOR.MINOR.PATCH (Contoh: v2.1.4)
  ├── MAJOR (v2.0.0) : Perubahan besar yang tidak kompatibel (Breaking Change, misal hapus variabel).
  ├── MINOR (v1.1.0) : Penambahan fitur baru yang tetap kompatibel dengan kode lama.
  └── PATCH (v1.0.1) : Perbaikan bug kecil tanpa mengubah struktur variabel.
```

### Memanggil Modul Berbasis Versi Git Terpin:
Alih-alih memanggil folder lokal (`../../modules/...`), tim produksi memanggil modul melalui repositori Git terpusat dengan mengunci tag versi:

```hcl
module "production_web" {
  # Mengunci modul ke tag versi immutable v1.2.0 resmi
  source = "git::https://gitlab.internal/sre/tofu-modules.git//web-server?ref=v1.2.0"

  container_name = "prod-store-web"
  host_port      = 80
  environment    = "prod"
}
```

Jika tim modul merilis versi baru `v2.0.0`, lingkungan produksi Anda **tidak akan terpengaruh** sampai Anda secara sengaja menguji dan menaikkan nilai `ref=v2.0.0` melalui proses review yang terencana.

---

## 7. Anti-Pattern: Modul Raksasa Serba Bisa (The God Module Anti-Pattern)

Salah satu kesalahan desain modul paling berbahaya adalah mencoba membuat **"Satu Modul Universal untuk Segala Kebutuhan"**.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                  CONTOH JEBAKAN GOD MODULE (ANTI-PATTERN)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  module "everything" {                                                      │
│    deploy_docker   = true                                                   │
│    deploy_proxmox  = false                                                  │
│    deploy_mysql    = true                                                   │
│    deploy_k8s      = false                                                  │
│    create_dns      = true                                                   │
│    ... 80 parameter variabel lainnya ...                                    │
│  }                                                                          │
│                                                                             │
│  Mengapa ini bencana SRE?                                                   │
│  1. Blast Radius Raksasa: Mengubah port NGINX berisiko menghapus database!   │
│  2. Output Plan Menyesatkan: tofu plan menghasilkan ratusan baris diff yang │
│     mustahil direview dengan teliti oleh manusia.                           │
│  3. Eksekusi Lambat: OpenTofu harus mengecek status puluhan API sekaligus.  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!TIP]
> **Prinsip Desain Modul SRE:**
> Terapkan **Single Responsibility Principle (SRP)**. Buatlah modul-modul kecil yang ramping dengan tugas tunggal yang spesifik: satu modul untuk `docker-nginx`, satu modul untuk `proxmox-vm`, dan satu modul untuk `minio-bucket`. Gabungkan modul-modul kecil ini di tingkat Root Module.

---

## 8. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Pesan Kesalahan OpenTofu | Penyebab Akar (*Root Cause*) | Solusi Pemulihan SRE |
|---|---|---|
| `Error: Module not installed` | Anda baru saja menambahkan blok `module` baru di HCL, namun belum menjalankan inisialisasi. | Jalankan perintah `tofu init` di direktori root module pemanggil. |
| `Error: Missing required argument` | Modul memiliki variabel input yang tidak memiliki nilai `default`, dan Anda lupa mengisinya di caller. | Periksa `variables.tf` pada modul anak, lalu tambahkan argumen yang kurang di blok `module {}`. |
| `Error: Provider configuration not present` | Child module membutuhkan provider tertentu, namun root module belum meneruskannya di argumen `providers = { ... }`. | Tambahkan blok pemetaan provider pada pemanggilan modul di root module. |
| `Plan ingin me-replace seluruh resource modul` | Nama key instance pada pemanggilan modul diubah, atau ada atribut `ForceNew` di dalam modul yang berubah. | Periksa diff `tofu plan`. Jika hanya ingin rename alamat tanpa destroy, gunakan `tofu state mv`. |

---

## 9. Ringkasan Konsep Kunci (Cheat Sheet)

```bash
# 1. Mengunduh dan memperbarui modul lokal/remote ke direktori .terraform/
tofu init -upgrade

# 2. Melihat struktur hierarki pemanggilan modul di dalam proyek
tofu providers

# 3. Memvalidasi sintaks seluruh modul pemanggil dan modul anak
tofu validate

# 4. Melakukan refactoring alamat resource modul di state tanpa memicu destroy fisik
tofu state mv 'module.old_web.docker_container.this' 'module.new_web.docker_container.this'
```

---

## 10. Cek Pemahaman Mandiri

1. Mengapa menduplikasi kode HCL dengan teknik *copy-paste* antar-lingkungan dianggap melanggar prinsip keandalan SRE? Apa yang dimaksud dengan fenomena *Configuration Drift* dalam konteks ini?
2. Jelaskan perbedaan peran dan tanggung jawab yang tegas antara *Root Module* dengan *Child Module*! Mengapa child module dilarang keras mengonfigurasi backend remote state sendiri?
3. Apa fungsi dari blok `validation {}` di dalam deklarasi sebuah variabel pada child module? Berikan contoh aturan validasi yang wajib dipasang untuk variabel nama kontainer!
4. Mengapa meletakkan konfigurasi `provider "docker" { ... }` langsung di dalam child module dianggap sebagai *anti-pattern*? Bagaimana cara yang benar untuk meneruskan konfigurasi provider dari root module?
5. Jelaskan apa yang dimaksud dengan *The God Module Anti-Pattern*! Sebutkan dua dampak buruk jika sebuah tim memaksakan pembuatan satu modul universal untuk seluruh jenis infrastruktur!

---

*Langkah Selanjutnya:* Masuk ke [02 — Environment, Workspace, dan State](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.2-modul-pola-produksi/02-environment-workspace-dan-state.md) untuk mempelajari bagaimana mengisolasi lingkungan *dev*, *staging*, dan *prod* secara fisik menggunakan struktur direktori terpisah, mengamankan kunci remote backend, dan mencegah bencana salah eksekusi perintah di server produksi.
