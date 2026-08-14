# LAB-01 — Modul Reusable Web Server

> **Lane:** Local / Docker / OrbStack Fast Lane  
> **Durasi:** 3–4 jam  
> **Tujuan:** mengubah konfigurasi web server datar menjadi child module reusable, memanggilnya dari root module, membuat beberapa instance dengan `for_each`, lalu membaca address/plan dan melakukan cleanup scoped.

> **Catatan runtime:** command hanya dapat diklaim berhasil jika benar-benar dijalankan dan evidence aman tersedia. Jika OpenTofu, provider Docker, atau OrbStack tidak tersedia, selesaikan Jalur B dan tandai acceptance runtime sebagai **belum diverifikasi**.

## Tujuan Belajar

Peserta dapat:

- memisahkan `modules/web-server` dari root caller;
- membuat input/output contract typed;
- meneruskan provider dari root module;
- memakai `for_each` dengan key stabil;
- membedakan read-only command dari mutation;
- mereview address change sebelum apply/destroy;
- menguji HTTP lokal tanpa menyimpan credential.

## Guardrail dan Scope

- Gunakan directory baru yang hanya berisi resource disposable.
- Verifikasi Docker context dan arsitektur sebelum init/apply.
- Gunakan image multi-arch dengan tag yang disetujui dan dipin; jangan `latest`.
- Port contoh `18080` dan `18081` harus bebas serta disposable.
- Tambahkan state, plan, var file, dan `.terraform/` ke `.gitignore`.
- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menargetkan container/network yang owner-nya tidak diketahui.
- `tofu apply`, `tofu destroy`, dan state operation hanya pada project lab ini setelah plan/scope/approval.
- `-auto-approve` bukan default.

## Jalur A — Runtime Docker/OrbStack

### 1. Preflight

```bash
set -eu

mkdir -p "$HOME/labs/opentofu/modul-3.2/lab-01"
cd "$HOME/labs/opentofu/modul-3.2/lab-01"
pwd
tofu version
docker version
docker context show
docker info --format '{{.OSType}}/{{.Architecture}}'
```

Expected architecture pada Mac Apple Silicon biasanya `linux/arm64` atau runtime yang mendukung image ARM64. Catat versi dan context sebagai evidence yang telah diredáksi.

### 2. Struktur Project

```text
lab-01/
├── modules/
│   └── web-server/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
└── .gitignore
```

Buat directory:

```bash
mkdir -p modules/web-server
```

### 3. Child Module

#### `modules/web-server/versions.tf`

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}
```

#### `modules/web-server/variables.tf`

```hcl
variable "name" {
  description = "Nama container web yang stabil."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{0,62}$", var.name))
    error_message = "name harus berupa DNS-label lowercase maksimal 63 karakter."
  }
}

variable "image" {
  description = "Image multi-arch dengan tag immutable yang telah disetujui."
  type        = string

  validation {
    condition     = !endswith(var.image, ":latest")
    error_message = "Gunakan image tag atau digest yang dipin, bukan latest."
  }
}

variable "host_port" {
  description = "Port host non-privileged untuk lab."
  type        = number

  validation {
    condition     = var.host_port >= 1024 && var.host_port <= 65535
    error_message = "host_port harus berada pada 1024–65535."
  }
}

variable "environment" {
  description = "Nama environment pemilik resource."
  type        = string
}

variable "labels" {
  description = "Label tambahan dari caller."
  type        = map(string)
  default     = {}
}
```

#### `modules/web-server/main.tf`

```hcl
locals {
  labels = merge(
    {
      owner       = "sre-bootcamp"
      managed_by  = "opentofu"
      component   = "web"
      environment = var.environment
    },
    var.labels,
  )
}

resource "docker_image" "this" {
  name         = var.image
  keep_locally = true
}

resource "docker_container" "this" {
  name  = var.name
  image = docker_image.this.image_id
  labels = local.labels

  ports {
    internal = 80
    external = var.host_port
  }
}
```

#### `modules/web-server/outputs.tf`

```hcl
output "container_id" {
  description = "ID container untuk inspeksi lokal."
  value       = docker_container.this.id
}

output "container_name" {
  description = "Nama container yang dikelola module."
  value       = docker_container.this.name
}

output "http_url" {
  description = "Endpoint HTTP lokal."
  value       = "http://127.0.0.1:${var.host_port}"
}
```

`container_id` hanya untuk evidence lokal dan tidak boleh dipakai sebagai kanal credential. Jika provider menambahkan atribut sensitif ke state, lindungi backend/state sesuai policy.

### 4. Root Module Caller

#### `providers.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}
```

#### `variables.tf`

```hcl
variable "environment" {
  description = "Environment disposable untuk lab."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging"], var.environment)
    error_message = "Lab hanya boleh memakai dev atau staging disposable."
  }
}

variable "web_image" {
  description = "Image nginx multi-arch dengan tag yang disetujui."
  type        = string
  default     = "nginx:1.27.5"

  validation {
    condition     = !endswith(var.web_image, ":latest")
    error_message = "Image lab harus dipin, bukan latest."
  }
}

variable "web_instances" {
  description = "Map key stabil ke port instance web."
  type = map(object({
    host_port = number
  }))
  default = {
    public = { host_port = 18080 }
    admin  = { host_port = 18081 }
  }

  validation {
    condition = alltrue([
      for item in values(var.web_instances) : item.host_port >= 1024 && item.host_port <= 65535
    ])
    error_message = "Semua port instance harus berada pada 1024–65535."
  }
}
```

#### `main.tf`

```hcl
module "web" {
  source   = "./modules/web-server"
  for_each = var.web_instances

  name        = "sre-${var.environment}-${each.key}"
  image       = var.web_image
  host_port   = each.value.host_port
  environment = var.environment

  labels = {
    lab = "modul-3.2-lab-01"
  }
}
```

#### `outputs.tf`

```hcl
output "web_endpoints" {
  description = "Endpoint tiap instance berdasarkan key stabil."
  value = {
    for key, instance in module.web : key => instance.http_url
  }
}

output "web_container_names" {
  description = "Nama container tiap instance."
  value = {
    for key, instance in module.web : key => instance.container_name
  }
}
```

#### `.gitignore`

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfplan
crash.log
crash.*.log
```

### 5. Format, Init, Validate, dan Plan (Read-Only)

```bash
tofu fmt -recursive
tofu init
tofu validate
tofu plan -out=lab-01.tfplan
tofu show -no-color lab-01.tfplan
```

Review minimum:

```text
+ module.web["public"].docker_image.this
+ module.web["public"].docker_container.this
+ module.web["admin"].docker_image.this
+ module.web["admin"].docker_container.this
```

Nama address dapat sedikit berbeda berdasarkan provider/version. Jangan apply bila plan menyentuh module/path/resource di luar lab, menghapus object yang tidak dikenal, atau mengganti port/name yang tidak dipahami.

### 6. Apply Setelah Approval

```bash
tofu apply lab-01.tfplan
```

Apply memakai plan artifact yang telah direview. Jangan mengganti dengan `tofu apply -auto-approve` sebagai default.

Inspeksi state dan runtime:

```bash
tofu output
tofu state list
tofu state show 'module.web["public"].docker_container.this'
tofu state show 'module.web["admin"].docker_container.this'
docker ps --filter name=sre-dev-public
docker ps --filter name=sre-dev-admin
curl --fail --silent --show-error http://127.0.0.1:18080
curl --fail --silent --show-error http://127.0.0.1:18081
```

HTTP 200/default page hanya membuktikan endpoint lokal merespons. Ia bukan bukti seluruh dependency atau SLO sehat.

### 7. Review Address Churn

Buat salinan perubahan **tanpa langsung apply**. Misalnya rename key `admin` menjadi `backoffice` pada input map. Jalankan:

```bash
tofu plan
```

Harapkan plan memperlihatkan penghapusan address lama dan pembuatan address baru, atau replacement sesuai provider. Kembalikan key bila tidak ada alasan bisnis. Jika rename memang diperlukan, gunakan prosedur state migration yang disetujui dan verifikasi plan; jangan mengedit state secara manual.

### 8. Graph dan State Evidence

```bash
tofu graph > lab-01.dot
tofu state list
tofu output -json
```

Simpan evidence dalam directory lokal yang di-ignore dan redaksi ID/internal endpoint bila dibagikan. Jangan commit `.tfplan`, `.tfstate`, `lab-01.dot` yang memuat detail sensitif, atau output mentah tanpa review.

### 9. Cleanup Scoped

Pastikan state hanya memuat resource lab:

```bash
tofu state list
tofu plan -destroy -out=destroy.tfplan
tofu show -no-color destroy.tfplan
```

Setelah scope diverifikasi dan approval tersedia:

```bash
tofu apply destroy.tfplan
```

Validasi:

```bash
tofu state list
docker ps -a --filter name=sre-dev-public
docker ps -a --filter name=sre-dev-admin
```

Jangan menghapus resource/image/network yang owner-nya tidak jelas. Cleanup manual hanya dilakukan jika object lab terverifikasi.

## Jalur B — Walkthrough Statis Tanpa Runtime

Jika `tofu`, provider Docker, atau OrbStack tidak tersedia:

1. buat seluruh struktur module dan root caller;
2. jalankan pemeriksaan syntax/editor atau parser HCL yang telah disetujui;
3. tulis predicted address dan action plan seperti di atas;
4. jelaskan command yang akan dijalankan pada runtime ARM64;
5. tandai `fmt/init/validate/plan/apply/HTTP/cleanup` sebagai **belum diverifikasi**;
6. jangan mengisi placeholder dengan credential atau mengklaim container telah dibuat.

## Failure Modes

### `Failed to install provider`

Periksa koneksi/provider mirror, versi OpenTofu, arsitektur `darwin_arm64`, dan lock file. Jangan mengunduh binary acak atau menghapus lock untuk menghilangkan error.

### Port conflict

Hentikan lab dan cari owner port. Ubah input ke port disposable lain, buat plan baru, dan jangan menghentikan process yang owner-nya tidak diketahui.

### `for_each` address berubah

Bandingkan key map dan predicted address. Rename key adalah perubahan identity, bukan sekadar label. Kembalikan key atau lakukan state migration terkontrol.

### Apply menampilkan resource di luar lab

Stop. Periksa root directory, backend/workspace, module source, variable file, provider context, dan state. Jangan memakai `-target` untuk menutupi scope yang salah.

### HTTP gagal setelah apply

Periksa `docker ps`, port mapping, logs, image architecture, runtime context, dan network. Apply sukses tidak sama dengan service sehat.

## Acceptance Criteria

- [ ] Struktur `modules/web-server` dan root caller dibuat.
- [ ] Input/output type, validation, provider passing, dan label ownership terdokumentasi.
- [ ] `for_each` memiliki key stabil dan address diprediksi.
- [ ] `fmt`, `init`, `validate`, dan `plan` dijalankan atau ditandai belum diverifikasi.
- [ ] Apply hanya dilakukan setelah plan review pada disposable scope, atau tidak dijalankan.
- [ ] Runtime HTTP dan state inspection dilakukan atau ditandai belum diverifikasi.
- [ ] Address churn akibat rename key dijelaskan tanpa mutation berbahaya.
- [ ] Destroy memakai destroy plan dan scope check, atau belum dijalankan.
- [ ] Tidak ada credential, state, plan, atau token di repository/evidence.

## Catatan SRE

Reusable module mengurangi duplikasi tetapi membuat perubahan contract berdampak ke banyak address. Simpan evidence predicted address, plan action, provider version, dan runtime health; jangan hanya menyimpan “apply berhasil”.

## Kaitan

- [01 — Arsitektur Modul Reusable](../01-arsitektur-modul-reusable.md)
- [03 — `for_each`, `count`, Conditional, Data Source](../03-foreach-count-conditional-data-source.md)
- [LAB-01 Modul 3.1](../../modul-3.1-dasar-opentofu/lab/LAB-01-tofu-docker-web-server.md)
- [LAB-02 — Environment Promotion dan Secret Safety](LAB-02-environment-promotion-dan-secret-safety.md)
