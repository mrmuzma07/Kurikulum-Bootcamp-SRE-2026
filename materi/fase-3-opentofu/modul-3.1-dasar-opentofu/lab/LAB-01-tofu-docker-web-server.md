# LAB-01 — OpenTofu Docker Web Server

> **Lane:** Local / Docker / OrbStack Fast Lane  
> **Durasi:** 2–3 jam  
> **Tujuan:** membuat satu web server disposable dengan provider Docker, membaca plan, memeriksa state, menguji HTTP, lalu membersihkannya dengan destroy plan yang scope-nya jelas.

## Tujuan Belajar

Setelah lab, peserta dapat:

- memverifikasi OpenTofu, Docker-compatible runtime, dan arsitektur ARM64;
- menulis `terraform`, `provider`, `resource`, `variable`, `locals`, dan `output`;
- membedakan `fmt`, `validate`, `plan`, `apply`, dan `destroy`;
- membaca action plan per resource address;
- mengumpulkan evidence tanpa memasukkan state, plan binary, atau credential ke Git.

## Guardrail Sebelum Mulai

- Gunakan directory baru khusus lab. Jangan menjalankan command dari project production.
- Pastikan Docker context menunjuk ke OrbStack/Docker runtime yang memang digunakan untuk lab.
- Port `18080` harus disposable dan tidak dipakai service lain.
- Gunakan image multi-arch dengan tag yang telah disetujui; jangan memakai `latest` untuk reproduksi incident.
- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menjalankan `tofu apply` atau `tofu destroy` sebelum membaca plan dan memverifikasi scope.
- Jangan menghapus container dengan nama yang tidak dibuat oleh lab.

## Prasyarat dan Preflight

```bash
command -v tofu
tofu version
command -v docker
docker version
docker context show
docker info --format '{{.OSType}}/{{.Architecture}}'
```

Expected pada Mac Apple Silicon adalah CLI `darwin_arm64` dan runtime/image native `linux/arm64` bila image tersebut mendukung ARM64. Jika runtime belum tersedia, lanjutkan bagian desain dan validasi statis, lalu catat bahwa apply belum diuji.

Buat directory kerja:

```bash
mkdir -p "$HOME/labs/opentofu/modul-3.1/lab-01"
cd "$HOME/labs/opentofu/modul-3.1/lab-01"
pwd
```

## 1. Tulis Project HCL

### `versions.tf`

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

`required_version` dan provider version adalah contoh yang harus diverifikasi terhadap binary/provider yang dipakai. `tofu init` menghasilkan lock file; review perubahan lock file sebelum menyimpannya sebagai evidence.

### `variables.tf`

```hcl
variable "container_name" {
  description = "Nama container web disposable."
  type        = string
  default     = "sre-tofu-web"

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{0,62}$", var.container_name))
    error_message = "Gunakan nama DNS-label lowercase yang aman untuk lab."
  }
}

variable "host_port" {
  description = "Port host non-privileged untuk HTTP lab."
  type        = number
  default     = 18080

  validation {
    condition     = var.host_port >= 1024 && var.host_port <= 65535
    error_message = "Port harus berada pada rentang 1024-65535."
  }
}

variable "web_image" {
  description = "Image web multi-arch yang telah disetujui."
  type        = string
  default     = "nginx:1.27.5"
}
```

Jika tag contoh tidak tersedia pada registry yang dipakai, pilih tag immutable/multi-arch yang tersedia dan catat penggantinya. Jangan mengganti dengan `latest` hanya agar pull berhasil.

### `main.tf`

```hcl
locals {
  common_labels = {
    owner       = "sre-bootcamp"
    environment = "lab"
    managed_by  = "opentofu"
    component   = "web"
  }
}

resource "docker_network" "lab" {
  name = "sre-tofu-lab"
}

resource "docker_image" "web" {
  name         = var.web_image
  keep_locally = true
}

resource "docker_container" "web" {
  name  = var.container_name
  image = docker_image.web.image_id

  labels = local.common_labels

  ports {
    internal = 80
    external = var.host_port
  }

  networks_advanced {
    name = docker_network.lab.name
  }
}
```

Referensi `docker_image.web.image_id` dan `docker_network.lab.name` membuat dependency implicit. Jika schema provider yang dipasang menolak `labels` atau `networks_advanced`, baca dokumentasi provider versi yang terkunci dan sesuaikan konfigurasi; jangan mematikan validasi untuk memaksa apply.

### `outputs.tf`

```hcl
output "http_url" {
  description = "URL HTTP web server lab."
  value       = "http://127.0.0.1:${var.host_port}"
}

output "container_name" {
  description = "Nama container yang dikelola lab."
  value       = docker_container.web.name
}

output "container_id" {
  description = "ID container untuk inspeksi lokal."
  value       = docker_container.web.id
}
```

Output bukan secret manager. Jangan menambahkan password/token sebagai output. `container_id` boleh dipakai untuk inspeksi lab, tetapi redaksi tetap diperlukan saat menyimpan evidence.

### `.gitignore`

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfplan
crash.log
crash.*.log
```

Tambahkan file ke Git hanya jika repository memang disiapkan untuk materi dan file tidak berisi state/credential. Untuk lab lokal, Git tidak wajib digunakan.

## 2. Format, Init, dan Validate

```bash
tofu fmt -check
tofu init
tofu fmt -check
tofu validate
```

Jika `fmt -check` gagal, jalankan `tofu fmt`, review diff, lalu ulangi. Jika `init` mengarah ke backend yang tidak diharapkan, berhenti dan periksa directory serta file `.tf`; jangan menerima migrasi backend secara buta.

Evidence read-only yang disarankan:

```bash
tofu version
tofu providers
tofu graph > graph.dot
```

`graph.dot` dapat memuat nama resource/internal endpoint; simpan hanya di lokasi yang sesuai policy.

## 3. Plan dan Review

Buat plan binary pada directory yang sudah di-ignore:

```bash
tofu plan -out=lab.tfplan
```

Baca plan dengan:

```bash
tofu show -no-color lab.tfplan
```

Checklist review:

```text
[ ] backend/local state dan directory benar
[ ] address docker_network.lab, docker_image.web, docker_container.web sesuai
[ ] action create hanya untuk resource lab
[ ] tidak ada destroy/replace yang tidak direncanakan
[ ] image tag dan port 18080 benar
[ ] tidak ada credential pada output/plan yang akan dibagikan
[ ] reviewer menyetujui apply
```

Jika plan berisi resource lain atau destroy tak terduga, berhenti. Jangan mengurangi perubahan dengan `-target` tanpa memahami dependency dan state.

## 4. Apply dan Validasi Runtime

Setelah plan dibaca dan disetujui:

```bash
tofu apply lab.tfplan
```

Periksa output dan state:

```bash
tofu output
tofu state list
tofu state show docker_container.web
```

Validasi runtime:

```bash
docker ps --filter name=sre-tofu-web
curl --fail --silent --show-error http://127.0.0.1:18080
```

Expected: `curl` mengembalikan halaman default Nginx. Ini hanya membuktikan HTTP lokal dan bukan health semua dependency. Jika memakai `-var='host_port=...'`, gunakan port yang sama saat `curl` dan catat evidence.

Cek perubahan konfigurasi tanpa apply:

```bash
tofu plan
```

Pada keadaan stabil, plan biasanya menunjukkan no changes. Hasil tersebut tidak membuktikan resource di luar boundary Docker provider tidak berubah.

## 5. Variasi Input Non-Secret

Buat file lokal yang tidak di-commit:

```bash
cat > lab.auto.tfvars <<'EOF'
container_name = "sre-tofu-web-alt"
host_port      = 18081
web_image      = "nginx:1.27.5"
EOF
```

Pastikan file tetap di-ignore, lalu review plan:

```bash
tofu fmt -check
tofu plan
```

Hapus container lama hanya melalui destroy plan dari directory/state yang benar sebelum menjalankan variasi lain. Jangan menjalankan dua container dengan port yang sama.

## 6. Cleanup Terbatas

Setelah evidence selesai, buat destroy plan terpisah:

```bash
tofu plan -destroy -out=destroy.tfplan
tofu show -no-color destroy.tfplan
```

Verifikasi lagi:

```text
[ ] directory lab-01 benar
[ ] state hanya resource disposable
[ ] tidak ada data yang harus dipertahankan
[ ] network/container/image scope sesuai
[ ] plan destroy disetujui
```

Lalu apply plan tersebut:

```bash
tofu apply destroy.tfplan
```

Validasi cleanup:

```bash
tofu state list
docker ps -a --filter name=sre-tofu-web
docker network ls --filter name=sre-tofu-lab
```

`keep_locally = true` dapat menyebabkan image tetap berada di cache Docker. Itu bukan state orphan dan bukan alasan menghapus image milik project lain. Hapus image hanya bila owner dan digest/tag telah diverifikasi.

## Troubleshooting

### Provider Docker gagal terhubung

Periksa `docker context show`, `docker info`, permission socket, dan apakah OrbStack Docker runtime aktif. Jangan menyalin credential atau socket path internal ke laporan publik.

### Pull image gagal di ARM64

Periksa manifest image dan arsitektur runtime. Pilih tag multi-arch yang telah diverifikasi; jangan langsung mengaktifkan emulasi tanpa mencatat dampak performa dan reproduksibilitas.

### Port `18080` sudah dipakai

Gunakan port lain pada rentang non-privileged melalui `-var`, lalu review plan. Jangan menghentikan process yang owner-nya tidak diketahui.

### Apply berhenti di tengah

Baca error provider, cek `docker ps -a` dan `tofu state list`, lalu jalankan plan baru setelah memahami object yang sudah dibuat. Jangan menghapus state secara manual.

### Destroy melihat resource tak dikenal

Berhenti. Periksa `pwd`, backend, state list, dan plan destroy. Jangan meneruskan hanya karena prompt konfirmasi muncul.

## Acceptance Criteria

- [ ] `tofu fmt -check` berhasil atau kegagalan terdokumentasi.
- [ ] `tofu init` dan `tofu validate` berhasil pada provider/runtime yang tersedia.
- [ ] Plan direview dan hanya menargetkan resource lab.
- [ ] HTTP lokal berhasil setelah apply, atau keterbatasan runtime dicatat.
- [ ] `tofu state list/show` menghasilkan evidence yang sudah diredáksi.
- [ ] Destroy memakai plan terpisah dan hanya state lab.
- [ ] Tidak ada `*.tfstate`, `*.tfplan`, `*.tfvars`, token, atau credential yang masuk Git.

## Handoff ke LAB-02

LAB-01 memakai local state. Lanjutkan ke [LAB-02 — State MinIO, Import & Drift](LAB-02-state-minio-import-drift.md) untuk memindahkan state ke S3-compatible backend, mengadopsi container manual, dan menguji drift pada resource disposable.
