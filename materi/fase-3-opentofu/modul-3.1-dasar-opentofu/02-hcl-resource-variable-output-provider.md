# 02 — Sintaks HCL: Resource, Variable, Output, Provider, dan Dependency Graph

> **Ringkasan:** HashiCorp Configuration Language (HCL) bukanlah bahasa pemrograman prosedural seperti Python atau Bash yang dieksekusi baris demi baris dari atas ke bawah. HCL adalah **bahasa deklarasi cetak biru** yang dirancang agar mudah dibaca oleh manusia sekaligus dapat dipetakan secara matematis oleh komputer menjadi graf dependensi (*Directed Acyclic Graph / DAG*). Dokumen ini membedah anatomi blok utama HCL, hierarki prioritas variabel input, pembuatan aturan validasi kustom, mekanisme ketergantungan implisit vs eksplisit, hingga mitos keamanan flag `sensitive = true`.

---

## 1. Analogi Pemula: Cetak Biru Arsitek vs Skrip Pemrograman

Bagi pemula yang memiliki latar belakang pemrograman, menulis HCL sering kali memicu kebingungan:
> *"Jika saya menaruh blok resource kontainer di baris 10, lalu menaruh resource gambarnya di baris 50, apakah OpenTofu akan error karena mencoba membuat kontainer sebelum gambarnya ada?"*

Jawabannya adalah: **TIDAK SAMA SEKALI!**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI CETAK BIRU ARSITEK RUMAH                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ BAHASA PEMROGRAMAN PROSEDURAL (PYTHON / BASH) ]                          │
│  Seperti rekaman kaset pita audio: Instruksi baris 1 WAJIB selesai sebelum  │
│  komputer membaca instruksi baris 2. Jika baris 2 membutuhkan variabel dari │
│  baris 50, program akan langsung crash melempar error "Variable undefined". │
│                                                                             │
│  [ BAHASA DEKLARATIF HCL OPENTOFU ]                                         │
│  Seperti lembaran gambar cetak biru arsitek bangunan:                       │
│  - Di sudut kiri atas digambar atap genteng (Baris 10 HCL).                 │
│  - Di sudut kanan bawah digambar pondasi beton tiang pancang (Baris 50 HCL).│
│  - Di tengah digambar dinding bata dan pintu (Baris 30 HCL).                │
│                                                                             │
│  Kontraktor cerdas (OpenTofu Engine) membaca SELURUH lembaran gambar        │
│  sekaligus, lalu menyusun urutan kerja yang logis:                          │
│  "Pondasi dicor terlebih dahulu, lalu dinding dipasang, dan terakhir atap   │
│   genteng dinaikkan." Posisi gambar di atas kertas tidak memengaruhi urutan │
│   pembangunan fisik sama sekali!                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Struktur Standar Proyek OpenTofu (Canonical Layout)

Di lingkungan produksi enterprise, kode HCL tidak boleh ditumpuk di dalam satu file raksasa ribuan baris. Praktik terbaik SRE memecah konfigurasi ke dalam beberapa file terpisah berdasarkan fungsinya:

```text
opentofu-project/
├── versions.tf               # Versi OpenTofu, provider requirements, dan backend
├── main.tf                   # Definisi resource utama dan orkestrasi arsitektur
├── variables.tf              # Deklarasi variabel input, tipe data, dan validasi
├── outputs.tf                # Data keluaran publik yang diekspos ke operator/modul lain
├── terraform.tfvars.example  # Template contoh pengisian variabel yang aman di-commit
├── .gitignore                # Mencegah file state dan credential lokal ter-commit
└── .terraform.lock.hcl       # Gembok kriptografis versi binary provider (Wajib di-commit!)
```

> [!NOTE]
> **Aturan Pembacaan OpenTofu:**
> Ketika Anda mengeksekusi `tofu plan` di sebuah direktori, OpenTofu akan **menggabungkan seluruh file berakhiran `.tf` di dalam direktori tersebut menjadi satu kesatuan logis**. Pemisahan file di atas murni untuk kenyamanan dan kerapian manusia (*human ergonomics*).

---

## 3. Anatomi 6 Blok Inti HCL

Mari kita bedah enam jenis blok bangunan utama di dalam bahasa HCL:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                             ENAM BLOK BANGUNAN HCL                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. terraform {} ──► Konfigurasi engine, provider yang dibutuhkan, backend   │
│ 2. provider  {} ──► Pengaturan otentikasi & koneksi ke API (Docker/Proxmox) │
│ 3. resource  {} ──► Objek fisik/virtual yang dikelola (dibuat/diubah/hapus) │
│ 4. variable  {} ──► Parameter input dinamis dari luar                       │
│ 5. locals    {} ──► Variabel internal hasil perhitungan kalkulasi logika    │
│ 6. output    {} ──► Nilai hasil akhir yang diekspos ke terminal / pipeline  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 1. Blok `terraform {}` (`versions.tf`)
Blok ini mendefinisikan batasan versi engine OpenTofu dan daftar provider yang wajib diunduh:

```hcl
terraform {
  # Batasan versi OpenTofu CLI yang diizinkan menjalankan kode ini
  required_version = ">= 1.6.0, < 2.0.0"

  required_providers {
    docker = {
      source  = "kreuzwerker/docker"  # Alamat registri provider resmi
      version = "~> 3.0.2"            # Operator ~> berarti: boleh 3.0.x, dilarang 3.1.x
    }
  }
}
```

---

### 2. Blok `provider {}` (`versions.tf` atau `main.tf`)
Blok ini mengonfigurasi parameter koneksi ke target infrastruktur:

```hcl
provider "docker" {
  # Di macOS dengan OrbStack, Docker socket berada di lokasi standar unix socket:
  host = "unix:///var/run/docker.sock"
}
```

---

### 3. Blok `resource {}` (`main.tf`)
Resource adalah inti dari IaC. Format penulisan resource selalu mengikuti aturan baku:
`resource "<tipe_provider>_<nama_objek>" "<label_lokal_hcl>"`

```hcl
# Mendownload image kontainer dari Docker Hub
resource "docker_image" "nginx_image" {
  name         = "nginx:1.27.4-alpine"
  keep_locally = true # Mencegah Docker menghapus image saat kontainer di-destroy
}

# Membuat dan menjalankan kontainer Docker
resource "docker_container" "web_app" {
  name  = var.app_name
  image = docker_image.nginx_image.image_id # REFERENSI IMPLISIT KE RESOURCE DI ATAS!

  ports {
    internal = 80           # Port di dalam kontainer NGINX
    external = var.web_port # Port fisik yang dibuka di laptop Mac host
  }

  env = [
    "ENVIRONMENT=${var.environment}",
    "APP_OWNER=${local.project_owner}"
  ]

  labels {
    label = "managed_by"
    value = "opentofu"
  }
}
```

---

### 4. Blok `variable {}` (`variables.tf`)
Variabel memungkinkan cetak biru HCL Anda menjadi dinamis dan dapat digunakan berulang kali untuk berbagai lingkungan (*reusable code*):

```hcl
variable "app_name" {
  description = "Nama kontainer web server di lingkungan lab."
  type        = string
  default     = "sre-lab-web"

  # Validasi kustom menggunakan Regular Expression (Regex)
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{2,30}$", var.app_name))
    error_message = "Nama aplikasi harus huruf kecil, angka, tanda strip (-), dan panjang 3-30 karakter."
  }
}

variable "web_port" {
  description = "Port host Mac yang akan di-bind ke kontainer."
  type        = number
  default     = 18080

  validation {
    condition     = var.web_port >= 1024 && var.web_port <= 65535
    error_message = "Gunakan port non-root aman antara rentang 1024 hingga 65535."
  }
}

variable "environment" {
  description = "Target deployment environment."
  type        = string
  default     = "staging"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment hanya boleh bernilai: dev, staging, atau prod."
  }
}
```

---

### 5. Blok `locals {}` (`main.tf`)
Jika sebuah nilai merupakan hasil penggabungan string atau perhitungan yang sering dipakai berulang kali, gunakan `locals` untuk menjaga prinsip **DRY (Don't Repeat Yourself)**:

```hcl
locals {
  project_owner = "sre-bootcamp-2026"
  common_tags = {
    Owner       = local.project_owner
    ManagedBy   = "OpenTofu"
    Environment = var.environment
    CreatedAt   = "2026-10-17"
  }
}
```

---

### 6. Blok `output {}` (`outputs.tf`)
Output bertindak seperti *return value* pada fungsi pemrograman. Nilai ini akan dicetak di layar terminal setelah `tofu apply` sukses, atau dikonsumsi oleh skrip otomatisasi lain:

```hcl
output "application_url" {
  description = "Alamat URL lengkap untuk mengakses web server dari browser Mac."
  value       = "http://127.0.0.1:${var.web_port}"
}

output "container_id" {
  description = "ID unik kontainer Docker yang dihasilkan oleh daemon."
  value       = docker_container.web_app.id
}

output "database_password" {
  description = "Password internal database (Contoh sensitif)."
  value       = "rahasia-dapur-sre"
  sensitive   = true # NILAI DISEMBUNYIKAN DI TERMINAL!
}
```

---

## 4. Hierarki Prioritas Pengisian Nilai Variabel (Variable Precedence)

Sering kali pemula bingung ketika sebuah variabel diisi di beberapa tempat sekaligus: di default HCL, di file `.tfvars`, dan di environment terminal. Nilai mana yang akan menang?

OpenTofu menerapkan urutan prioritas yang sangat ketat (dari **Prioritas Tertinggi #1** ke **Prioritas Terendah #6**):

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│               URUTAN PRIORITAS PENGISIAN VARIABEL (PRECEDENCE)              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PRIORITAS TERTINGGI / MENANG MUTLAK ]                                    │
│  1. Argumen Langsung CLI Flag   : tofu apply -var="web_port=9090"           │
│  2. File Var Spesifik CLI Flag  : tofu apply -var-file="prod.tfvars"        │
│  3. File Var Otomatis (*.auto)  : custom.auto.tfvars / custom.auto.tfvars.json│
│  4. File Var Standar            : terraform.tfvars                          │
│  5. Environment Variable OS     : export TF_VAR_web_port=8888              │
│  6. Nilai Bawaan di HCL Kode    : default = 18080 (di variables.tf)         │
│  [ PRIORITAS TERENDAH ]                                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **Bahaya Keamanan Flag `-var` di Terminal:**
> Jangan pernah memasukkan password rahasia atau token API menggunakan flag `-var="db_pass=xxx"` langsung di baris perintah shell! Perintah tersebut akan tercatat permanen di dalam file riwayat Bash/Zsh (`~/.zsh_history`) dan dapat dibaca oleh pengguna lain di mesin yang sama menggunakan perintah `ps aux`. Gunakan environment variable terisolasi atau tool secret manager.

---

## 5. Membedah Dependency Graph (Relasi Antar-Objek)

Salah satu kehebatan terbesar mesin OpenTofu adalah kemampuannya menyusun **Directed Acyclic Graph (DAG)** secara otomatis untuk menentukan urutan eksekusi pembuatan dan penghancuran resource.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                          GRAPH KETERGANTUNGAN OBJEK                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   docker_image.nginx_image ──( Dibuat Pertama Kali )──┐                     │
│                                                       ▼                     │
│                                            docker_container.web_app         │
│                                                       ▲                     │
│   var.web_port ─────────────( Dibaca Bersamaan )──────┘                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. Ketergantungan Implisit (*Implicit Dependency* - Cara Terbaik!)
Perhatikan baris kode pada kontainer web kita:
```hcl
image = docker_image.nginx_image.image_id
```
Dengan menuliskan ekspresi `docker_image.nginx_image.image_id`, OpenTofu secara otomatis memahami bahwa kontainer `web_app` **bergantung mutlak** pada ketersediaan image `nginx_image`.
- **Saat Pembuatan (`apply`):** OpenTofu otomatis men-download image terlebih dahulu, baru kemudian menjalankan kontainer.
- **Saat Penghancuran (`destroy`):** OpenTofu membalik urutannya: mematikan dan menghapus kontainer terlebih dahulu, baru kemudian menghapus image.

### 2. Ketergantungan Eksplisit (*Explicit Dependency* via `depends_on`)
Kadang-kadang ada ketergantungan tersembunyi yang tidak terlihat dari referensi variabel HCL (misalnya: Kontainer Aplikasi membutuhkan Database SQL sudah selesai inisialisasi tabel, meskipun tidak ada atribut IP yang disambungkan langsung).

Untuk situasi khusus ini, kita menggunakan atribut darurat `depends_on`:

```hcl
resource "docker_container" "web_app" {
  name  = "sre-web"
  image = "nginx:alpine"

  # Memberitahu OpenTofu secara paksa untuk menunggu database selesai dibuat!
  depends_on = [
    docker_container.mysql_database
  ]
}
```

> [!WARNING]
> **Anti-Pattern Penggunaan `depends_on` yang Berlebihan:**
> Jangan menyematkan `depends_on` di setiap resource jika sudah ada referensi implisit. Menggunakan `depends_on` secara membabi buta akan mematikan fitur eksekusi paralel OpenTofu, memperlambat proses `tofu apply`, dan berisiko memicu kesalahan graf lingkaran (*Circular Dependency / Cycle Error*).

---

## 6. Mitos Keamanan: Apa Sebenarnya Arti `sensitive = true`?

Banyak engineer pemula berasumsi keliru:
> *"Saya sudah menandai `sensitive = true` pada output database password, berarti password saya aman terenkripsi dan tidak ada yang bisa melihatnya."*

Ini adalah **asumsi salah yang sangat berbahaya bagi keamanan sistem!**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FAKTA KEAMANAN FLAG SENSITIVE = TRUE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ DI TERMINAL CLI ($ tofu apply / $ tofu output) ]                         │
│  Outputs:                                                                   │
│    application_url   = "http://127.0.0.1:18080"                             │
│    database_password = <sensitive>  <── DI-MASKING DARI PANDANGAN MATA      │
│                                                                             │
│  [ TAPI DI DALAM FILE terraform.tfstate (HARDDISK / BUCKET) ]               │
│  $ cat terraform.tfstate | jq .outputs.database_password                    │
│  {                                                                          │
│    "type": "string",                                                        │
│    "value": "rahasia-dapur-sre",    <── TERCATAT TELANJANG (PLAINTEXT)!     │
│    "sensitive": true                                                        │
│  }                                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **Kesimpulan Keamanan SRE:**
> Flag `sensitive = true` **HANYA BERFUNGSI SEBAGAI KOSMETIK PRESENTASI** di layar terminal agar password Anda tidak terekam saat demo layar atau masuk ke log CI/CD publik. Nilai tersebut **TETAP DISIMPAN SECARA MENTAH TANPA ENKRIPSI DI DALAM FILE STATE**! Perlindungan sejati hanya bisa dicapai dengan mengenkripsi file state atau menggunakan remote backend terproteksi (seperti fitur *State Encryption* bawaan OpenTofu v1.7+).

---

## 7. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Pesan Kesalahan HCL | Penyebab Akar (*Root Cause*) | Perintah Solusi SRE |
|---|---|---|
| `Error: Reference to undeclared input variable` | Variabel digunakan di `main.tf` (misal `var.port`), tetapi belum dideklarasikan bloknya di `variables.tf`. | Tambahkan deklarasi `variable "port" {}` di file `variables.tf`. |
| `Error: Cycle: resource.a -> resource.b -> resource.a` | Terjadi ketergantungan melingkar (*Circular Dependency*). Objek A butuh Objek B, dan Objek B butuh Objek A. | Periksa relasi atribut atau hapus `depends_on` yang memicu ketergantungan bolak-balik. |
| `Error response from daemon: port is already allocated` | Port host Mac yang diminta di HCL (misal port 80) sudah digunakan oleh aplikasi lain di Mac Anda. | Ganti nilai variabel `web_port` ke nomor port bebas lain via `-var="web_port=18081"`. |
| `Error: Invalid value for variable (Validation failed)` | Nilai yang Anda masukkan melanggar aturan logika blok `validation {}` di `variables.tf`. | Baca pesan `error_message` yang muncul dan sesuaikan format nilai input Anda. |

---

## 8. Ringkasan Konsep Kunci (Cheat Sheet HCL)

```bash
# 1. Merapikan format spasi dan indentasi seluruh file HCL sesuai standar resmi
tofu fmt

# 2. Memeriksa apakah ada file yang formatnya belum rapi tanpa mengubah isinya
tofu fmt -check -diff

# 3. Memvalidasi sintaksis dan referensi logika internal HCL
tofu validate

# 4. Membaca nilai output yang ditandai sensitive dalam format JSON
tofu output -json database_password

# 5. Menghasilkan visualisasi diagram relasi dependensi (Graphviz)
tofu graph | dot -Tpng > graph.png
```

---

## 9. Cek Pemahaman Mandiri

1. Mengapa meletakkan deklarasi `resource "docker_container"` sebelum `resource "docker_image"` di dalam file `main.tf` tidak menyebabkan OpenTofu gagal saat dijalankan? Konsep apa yang mendasarinya?
2. Sebutkan urutan prioritas (*variable precedence*) jika nilai port aplikasi didefinisikan secara berbeda di file `terraform.tfvars` dan di environment variable sistem operasi `TF_VAR_web_port`! Nilai mana yang akan dieksekusi oleh OpenTofu?
3. Jelaskan perbedaan mendasar antara *Implicit Dependency* dengan *Explicit Dependency (`depends_on`)*! Kapan seorang SRE harus menghindari pemakaian `depends_on`?
4. Apakah menambahkan atribut `sensitive = true` pada sebuah blok output otomatis mengamankan password dari kebocoran jika seseorang berhasil mengunduh file `terraform.tfstate` Anda? Jelaskan alasannya!
5. Apa kegunaan utama blok `validation {}` di dalam deklarasi sebuah variabel? Berikan contoh skenario nyata di mana validasi variabel dapat mencegah kerusakan infrastruktur produksi!

---

*Langkah Selanjutnya:* Masuk ke [03 — Workflow init, plan, apply, destroy](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.1-dasar-opentofu/03-workflow-init-plan-apply-destroy.md) untuk mempelajari alur siklus hidup operasional eksekusi OpenTofu, memahami kode simbol perubahan pada output `plan`, dan mitigasi risiko saat melakukan perubahan di kluster nyata.
