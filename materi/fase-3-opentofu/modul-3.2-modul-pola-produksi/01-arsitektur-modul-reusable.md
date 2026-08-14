# 01 — Arsitektur Modul Reusable

> **Tujuan:** memahami module sebagai contract yang dapat dipanggil berulang, bukan sekadar folder untuk memindahkan file HCL.

## Tujuan Belajar

Setelah membaca materi ini, peserta dapat:

- membedakan root module dengan child module;
- menyusun input/output contract yang typed dan terdokumentasi;
- meneruskan provider secara eksplisit;
- memilih batas abstraksi dan ownership yang dapat dioperasikan;
- menjelaskan versioning module dan dampak perubahan contract.

## 1. Mengapa Module?

Tanpa module, setiap environment sering menyalin resource yang sama. Duplikasi membuat bug dan policy drift: satu environment mungkin memiliki label berbeda, validation hilang, atau timeout tidak konsisten. Module mengemas pola yang berulang dengan contract yang dapat direview.

Module bukan magic boundary. Caller tetap bertanggung jawab memilih environment, backend, provider configuration, jumlah instance, dan approval. Child module bertanggung jawab pada resource yang memang dideklarasikan contract-nya.

```text
root module (environment caller)
  ├── memilih backend dan provider configuration
  ├── mengisi input module
  ├── menetapkan jumlah/komposisi instance
  └── mengekspos output ke tahap berikutnya
       ↓
child module (reusable capability)
  ├── memvalidasi input
  ├── membuat resource yang menjadi ownership-nya
  └── mengembalikan output minimum yang berguna
```

## 2. Root Module dan Child Module

Root module adalah kumpulan file `.tf` pada directory tempat `tofu plan` dijalankan. `environments/dev`, `environments/staging`, dan `environments/prod` sebaiknya menjadi root module terpisah bila memerlukan backend, permission, atau lifecycle berbeda.

Child module dipanggil dengan block `module`. Relative module lokal cocok untuk repository yang sama; module remote atau registry perlu source dan version yang dipin serta diuji.

```hcl
module "web" {
  source = "../../modules/web-server"

  name       = "sre-web-dev"
  image      = var.web_image
  host_port  = 18080
  environment = "dev"
}
```

Jalankan `tofu init` setelah source module berubah. Jangan menganggap perubahan source otomatis aman: `tofu plan` tetap harus dibaca.

## 3. Input Contract

Input adalah API module. Gunakan type, description, default yang aman, dan validation untuk membatasi kombinasi yang tidak valid. Default tidak boleh diam-diam memilih production resource atau credential.

```hcl
variable "name" {
  description = "Nama instance web yang stabil dan dimiliki module."
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

variable "labels" {
  description = "Label ownership tambahan dari caller."
  type        = map(string)
  default     = {}
}
```

Jangan memasukkan secret ke contoh default. Bila module memang membutuhkan secret untuk provider/resource, nilai harus datang dari boundary secret yang disetujui dan konsekuensi state harus didokumentasikan.

## 4. Output Contract

Output adalah API keluar. Kembalikan metadata yang diperlukan caller atau handoff, bukan seluruh object provider secara membabi buta.

```hcl
output "container_name" {
  description = "Nama container yang dikelola module."
  value       = docker_container.this.name
}

output "http_url" {
  description = "URL endpoint lokal untuk verifikasi lab."
  value       = "http://127.0.0.1:${var.host_port}"
}

output "provisioning_metadata" {
  description = "Metadata minimum untuk tahap konfigurasi berikutnya."
  value = {
    name        = docker_container.this.name
    environment = var.environment
    port        = var.host_port
  }
}
```

`output` yang berisi sensitive value harus diberi `sensitive = true`, tetapi ini bukan enkripsi state. Jangan menjadikan output sebagai kanal untuk password, private key, token, atau kubeconfig.

## 5. Provider Passing

Child module umumnya tidak membuat credential atau memilih endpoint provider sendiri. Root module mengonfigurasi provider lalu meneruskannya:

```hcl
provider "docker" {
  alias = "orb"
}

module "web" {
  source = "../../modules/web-server"

  providers = {
    docker = docker.orb
  }

  name        = "sre-web-dev"
  image       = "nginx:1.27.5"
  host_port   = 18080
  environment = "dev"
}
```

Module perlu mendeklarasikan provider yang dibutuhkan melalui `required_providers`. Jangan hard-code endpoint production dalam child module. Provider alias berguna ketika satu root module memang perlu lebih dari satu endpoint yang scope-nya terdokumentasi; jangan menggunakannya untuk mencampur environment tanpa review.

## 6. Composition dan Ownership

Satu root module dapat menyusun beberapa capability:

```hcl
module "web" {
  source = "../../modules/web-server"
  for_each = {
    public = { port = 18080 }
    admin  = { port = 18081 }
  }

  name        = "sre-${each.key}"
  image       = var.web_image
  host_port   = each.value.port
  environment = var.environment
}

output "web_endpoints" {
  value = {
    for key, instance in module.web : key => instance.http_url
  }
}
```

Ownership harus terlihat pada address: `module.web["public"].docker_container.this`. Jika dua module mengelola object yang sama, state dapat berkonflik. Data source untuk membaca object bukan pengganti contract ownership.

## 7. Versioning dan Perubahan Contract

Untuk module lokal, gunakan Git review dan tag release bila module dipakai lintas repository. Untuk module registry/remote, pin `version` atau ref yang immutable sesuai policy. Setiap perubahan berikut diperlakukan sebagai perubahan API:

- menghapus/rename input atau output;
- mengubah default atau validation;
- mengganti provider/resource;
- mengubah nama `for_each` key;
- mengubah lifecycle atau replacement behavior;
- mengubah required provider version.

Sebelum upgrade module:

1. baca changelog dan compatibility matrix;
2. jalankan `tofu fmt`, `tofu validate`, dan plan pada environment disposable;
3. review address/action dan replacement;
4. promosikan ke staging sebelum production;
5. simpan rollback reference yang benar-benar dapat dipakai.

Jangan mengedit state hanya agar upgrade terlihat no-op.

## 8. Anti-Pattern: Module Terlalu Abstrak

Module yang menerima puluhan flag untuk semua provider dan semua platform sering menyembunyikan ownership serta membuat plan sulit dipahami. Tanda bahaya:

- satu module memiliki conditional untuk Docker, Proxmox, dan vSphere sekaligus;
- nama variable generik tetapi mengubah banyak resource unrelated;
- output mengembalikan raw state atau seluruh credential-adjacent object;
- caller tidak dapat memperkirakan resource address;
- perubahan kecil pada input mengganti seluruh topology.

Lebih baik beberapa module kecil dengan contract jelas daripada satu module universal yang tidak dapat diuji. Abstraksi harus membungkus capability yang memiliki lifecycle dan owner serupa.

## Troubleshooting

### `Module not installed`

Pastikan `source` relatif benar dari root module, jalankan `tofu init`, dan cek directory kerja. Jangan menyalin `.terraform` antar-directory.

### `Provider configuration not present`

Periksa `required_providers`, nama provider pada child module, dan map `providers` pada caller. Pastikan alias diteruskan dengan key yang sesuai.

### Banyak resource replacement setelah perubahan module

Bandingkan address lama dan baru, perubahan provider schema, nama, `for_each` key, dan lifecycle. Hentikan apply jika replacement tidak dipahami. `state mv` hanya dilakukan setelah backup, lock, scope check, dan approval.

### Output ingin memuat credential

Hentikan desain tersebut. Output dan plan artifact dapat disimpan, dilog, atau diteruskan. Gunakan secret mechanism terpisah dan keluarkan hanya metadata non-secret.

## Acceptance Checklist

- [ ] Root module dan child module dapat dijelaskan beserta owner-nya.
- [ ] Semua input memiliki type/description; input berisiko memiliki validation.
- [ ] Output minimum dan tidak membocorkan credential.
- [ ] Provider passing/alias terlihat di caller, bukan endpoint hard-code di child module.
- [ ] Module source/version dapat direview dan diuji sebelum promotion.
- [ ] Address module dan dampak rename key dipahami.
- [ ] Contoh tidak menggunakan image `latest` atau secret literal.

## Catatan SRE

Module adalah API internal untuk infrastruktur. Stabilitas contract, keterbacaan plan, dan kemampuan rollback lebih penting daripada mengurangi jumlah baris HCL. Setiap module release harus memiliki owner, test path, compatibility note, dan batas blast radius.

## Kaitan dengan Modul Berikutnya

- [02 — Environment, Workspace, dan State](02-environment-workspace-dan-state.md) menerapkan module pada isolasi environment.
- [03 — `for_each`, `count`, Conditional, Data Source](03-foreach-count-conditional-data-source.md) membahas pola instansiasi dan address.
- Modul 3.3 — Konteks On-Prem masih menyusul; modul tersebut akan memperluas provider boundary ke on-prem.
