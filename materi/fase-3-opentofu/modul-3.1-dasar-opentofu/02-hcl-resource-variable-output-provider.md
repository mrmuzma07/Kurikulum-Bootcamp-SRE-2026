# 02 — HCL, Resource, Variable, Output, dan Provider

> **Tujuan:** dapat membaca konfigurasi HCL dasar, memisahkan input/output, memahami graph dependency, dan menulis contoh non-secret yang dapat diverifikasi pada provider Docker.

## 1. Struktur Project

Struktur minimal:

```text
opentofu-lab/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── terraform.tfvars.example
└── .gitignore
```

`*.tfvars` yang berisi nilai sensitif jangan di-commit. File `terraform.tfvars.example` hanya boleh berisi contoh aman.

## 2. Blok HCL

OpenTofu membaca semua file `.tf` dalam satu directory module root. Urutan nama file tidak menentukan urutan eksekusi. Dependency graph diturunkan dari referensi antar resource.

```hcl
terraform {
  required_version = ">= 1.6.0"
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

resource "docker_network" "lab" {
  name = "sre-tofu-lab"
}

output "network_name" {
  value = docker_network.lab.name
}
```

Contoh di atas mengandung provider, resource, dan output. `host` bergantung pada konfigurasi runtime lokal; sesuaikan socket dengan Docker-compatible runtime yang digunakan dan jangan menaruh endpoint credential dalam Git.

## 3. Provider

Provider dikonfigurasi satu kali atau dengan alias untuk beberapa endpoint. Versi provider dipin:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}
```

Jalankan `tofu init` untuk mengunduh provider dan menghasilkan lock file. Review perubahan lock file bersama konfigurasi. Pada Mac ARM64, provider harus menyediakan binary yang kompatibel atau memiliki jalur instalasi yang telah diuji.

## 4. Resource

Resource menyatakan object yang dikelola:

```hcl
resource "docker_image" "web" {
  name         = "nginx:1.27.5"
  keep_locally = true
}

resource "docker_container" "web" {
  name  = var.container_name
  image = docker_image.web.image_id

  ports {
    internal = 80
    external = var.host_port
  }
}
```

Referensi `docker_image.web.image_id` membuat dependency implisit. Tag contoh harus diverifikasi ketersediaannya dan diganti dengan versi immutable/digest yang disetujui untuk reproduksi.

`keep_locally = true` mencegah provider menghapus image lokal pada destroy container, tetapi bukan mekanisme backup. Resource yang memakai port host harus memiliki scope lab agar tidak bertabrakan dengan service lain.

## 5. Variable

Variable menjadi kontrak input module:

```hcl
variable "container_name" {
  description = "Nama container lab."
  type        = string
  default     = "sre-tofu-web"

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{0,62}$", var.container_name))
    error_message = "Gunakan nama DNS-label lowercase yang aman untuk lab."
  }
}

variable "host_port" {
  description = "Port host untuk HTTP lab."
  type        = number
  default     = 18080

  validation {
    condition     = var.host_port >= 1024 && var.host_port <= 65535
    error_message = "Port lab harus berada pada rentang non-privileged 1024-65535."
  }
}
```

Pola sumber nilai, dari prioritas paling tinggi ke rendah, meliputi `-var`, `-var-file`, environment variable `TF_VAR_name`, dan default. Hindari memasukkan secret dalam command line karena shell history dan process listing dapat merekamnya.

## 6. Locals dan Expressions

`locals` membantu memberi nama expression yang digunakan ulang:

```hcl
locals {
  common_labels = {
    owner       = "sre-bootcamp"
    environment = "lab"
    managed_by  = "opentofu"
  }
}
```

HCL mendukung string interpolation dan expression:

```hcl
name = "${var.container_name}-${var.environment}"
```

Untuk expression sederhana, sintaks tanpa interpolation sering lebih mudah dibaca:

```hcl
name = join("-", [var.container_name, var.environment])
```

Jangan membangun nilai secret ke nama resource, label, output, atau log plan.

## 7. Output

Output adalah interface module ke operator atau module parent:

```hcl
output "http_url" {
  description = "URL HTTP lab."
  value       = "http://127.0.0.1:${var.host_port}"
}

output "container_id" {
  description = "ID container lab untuk inspeksi."
  value       = docker_container.web.id
}
```

Output dapat muncul pada terminal dan state. Jika value sensitif, gunakan `sensitive = true`, tetapi pahami bahwa sensitive hanya menyamarkan output CLI; value tetap dapat berada dalam state.

```hcl
output "example_sensitive" {
  value     = var.example_non_secret_placeholder
  sensitive = true
}
```

Contoh di atas hanya menunjukkan sintaks; jangan mengganti placeholder dengan credential dalam repository.

## 8. Data Source

Data source membaca object yang sudah ada tanpa mengelolanya sebagai lifecycle resource:

```hcl
data "docker_network" "existing" {
  name = "sre-tofu-shared"
}
```

Data source dapat gagal jika object belum ada. Gunakan hanya ketika ownership jelas. Jangan memakai data source untuk menghindari state ownership atau menyembunyikan perubahan yang seharusnya direview.

## 9. Dependency Graph

```bash
tofu graph > graph.dot
```

Perintah tersebut menghasilkan graph untuk inspeksi. Referensi antar resource menghasilkan edge implicit. `depends_on` adalah escape hatch untuk dependency yang tidak terlihat dari value reference:

```hcl
resource "docker_container" "probe" {
  name       = "sre-tofu-probe"
  image      = docker_image.web.image_id
  depends_on = [docker_container.web]
}
```

Gunakan `depends_on` sesedikit mungkin. Dependency berlebihan memperlambat plan dan dapat memperluas replacement.

## 10. Naming dan Tagging

Gunakan nama yang:

- lowercase dan stabil;
- menyertakan project/environment, bukan data personal;
- tidak memuat token, hostname sensitif, atau timestamp acak tanpa alasan;
- mudah dicari di provider dan state.

Gunakan label/tag seperti `managed_by=opentofu`, `owner=sre-bootcamp`, dan `environment=lab` jika provider mendukung. Label membantu cleanup scoped, tetapi bukan pengganti state atau approval.

## Acceptance Checklist

- [ ] Bisa menjelaskan beda provider, resource, dan data source.
- [ ] Bisa membuat variable bertipe `string` dan `number` dengan validation.
- [ ] Bisa menunjukkan dependency implisit dari referensi resource.
- [ ] Bisa menjelaskan mengapa `sensitive = true` tidak mengenkripsi state.
- [ ] Bisa menjalankan `tofu fmt` pada contoh tanpa memasukkan credential.

## Troubleshooting

### `Invalid reference`

Periksa address resource, nama variable, dan apakah blok berada pada module root yang sama. Jalankan `tofu validate` setelah `tofu fmt`.

### Port sudah digunakan

Pilih port non-privileged lain melalui `-var` atau file var lokal yang di-ignore. Jangan membunuh process yang tidak diketahui hanya untuk memenuhi port lab.

### Provider tidak menemukan Docker socket

Periksa runtime OrbStack/Docker, `docker context show`, permission socket, dan konfigurasi provider. Jangan menuliskan socket atau credential internal environment ke laporan jika tidak diperlukan.

### Output terlihat `<sensitive>`

Itu perilaku yang diharapkan. Gunakan `tofu output -json` hanya pada environment yang aman dan jangan menyalin value sensitif ke chat, Git, atau evidence publik.

## Catatan SRE

HCL yang mudah dibaca adalah bagian dari operability. Nama yang stabil, validation yang tepat, dan output yang terkontrol mengurangi waktu diagnosis saat plan berubah atau provider mengembalikan error.

## Kaitan dengan Modul Berikutnya

- Jalankan struktur ini melalui [03 — Workflow init, plan, apply, destroy](03-workflow-init-plan-apply-destroy.md).
- Pahami konsekuensi state, output, dan import di [04 — State, remote, import, drift](04-state-remote-import-drift.md).
- Struktur module reusable dan environment isolation menjadi fokus Modul 3.2.
