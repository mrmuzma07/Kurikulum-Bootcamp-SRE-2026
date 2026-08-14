# 02 — Environment, Workspace, dan State

> **Tujuan:** memilih boundary environment yang aman, memahami perbedaan workspace dan directory, serta mencegah state key dan provider context tertukar.

## Tujuan Belajar

Peserta dapat:

- menyusun `environments/dev`, `staging`, dan `prod` sebagai root module;
- membandingkan workspace dengan directory per environment;
- mengisolasi backend key, permission, dan approval;
- meneruskan provider alias tanpa mencampur endpoint;
- membuat promotion yang dapat diaudit.

## 1. Environment sebagai Boundary Operasional

Environment bukan hanya variable `environment = "prod"`. Ia mencakup root module, backend/state key, provider endpoint, identity/permission, resource sizing, policy, approval, dan recovery procedure.

```text
environments/dev/     → state: sre-tofu/dev/terraform.tfstate
                          endpoint: disposable/lab
                          approval: operator lab

environments/staging/ → state: sre-tofu/staging/terraform.tfstate
                          endpoint: pre-production
                          approval: reviewer + owner

environments/prod/    → state: sre-tofu/prod/terraform.tfstate
                          endpoint: production on-prem
                          approval: change process
```

Key environment harus unik. Backend key yang sama dapat menyebabkan dua root module menulis state yang tidak semestinya, walaupun nama variable berbeda.

## 2. Struktur Directory

Pola yang mudah direview:

```text
environments/
├── dev/
│   ├── backend.tf
│   ├── providers.tf
│   ├── main.tf
│   └── variables.tf
├── staging/
│   ├── backend.tf
│   ├── providers.tf
│   ├── main.tf
│   └── variables.tf
└── prod/
    ├── backend.tf
    ├── providers.tf
    ├── main.tf
    └── variables.tf
```

Contoh caller `dev`:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }

  backend "s3" {
    bucket                      = "sre-tofu-state"
    key                         = "modul-3.2/dev/terraform.tfstate"
    region                      = "us-east-1"
    endpoint                    = "http://127.0.0.1:9000"
    use_path_style              = true
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
  }
}

provider "docker" {}

module "web" {
  source = "../../modules/web-server"

  name        = "sre-web-dev"
  image       = var.web_image
  host_port   = 18080
  environment = "dev"
}
```

Backend configuration bersifat literal pada init. Nilai credential tidak boleh ditaruh di block. Parameter S3-compatible dan locking harus diverifikasi terhadap versi OpenTofu/backend yang digunakan.

## 3. Workspace versus Directory

| Pertimbangan | Workspace | Directory per environment |
|---|---|---|
| State | Sering dibedakan oleh workspace key | Terlihat eksplisit dari backend tiap directory |
| Review context | Operator harus memeriksa workspace aktif | Path membantu mengingat scope, tetapi tetap perlu verifikasi |
| Perbedaan topology | Kurang nyaman untuk konfigurasi yang berbeda jauh | Mudah menampilkan konfigurasi dan policy per environment |
| Permission | Sering berbagi root/config credential boundary | Dapat dipisah di repository/CI job/identity |
| Blast radius | Salah workspace dapat merusak environment lain | Salah directory tetap berbahaya, tetapi boundary lebih terlihat |
| Cocok untuk | Variasi kecil dengan topology sama | Environment production dengan approval dan policy berbeda |

Workspace bukan security boundary dengan sendirinya. `tofu workspace select prod` yang salah dapat mengarahkan command ke state berbahaya. Directory juga bukan security boundary mutlak; permission backend dan CI identity tetap harus benar.

Rekomendasi default untuk on-prem production adalah directory per environment atau root module terpisah, terutama bila backend, credential, topology, policy, dan approval berbeda. Workspace dapat digunakan untuk variasi kecil yang benar-benar homogen dan memiliki guardrail kuat.

## 4. Preflight Context

Sebelum command read-only maupun mutation:

```bash
pwd
tofu workspace show
tofu workspace list
tofu providers
tofu state list
```

Periksa backend pada file environment dan output `tofu init`. Untuk apply/destroy/import/state operation, tambahkan checklist manual:

```text
[ ] directory benar
[ ] branch/commit benar
[ ] backend endpoint benar
[ ] bucket/key benar dan khusus environment
[ ] workspace benar jika workspace dipakai
[ ] provider identity memiliki scope minimum
[ ] plan dibuat dari konfigurasi dan input yang direview
[ ] approval tersedia
```

Jangan menyimpan output yang mengandung credential. Redact endpoint internal atau identity detail bila evidence dibagikan di luar owner.

## 5. Provider Alias

Alias berguna ketika root module harus membaca atau membuat object melalui lebih dari satu endpoint yang sudah disetujui:

```hcl
provider "docker" {
  alias = "lab"
}

provider "docker" {
  alias = "shared"
}

module "web" {
  source = "../../modules/web-server"

  providers = {
    docker = docker.lab
  }

  name        = "sre-web-dev"
  image       = var.web_image
  host_port   = 18080
  environment = "dev"
}
```

Jangan memakai alias untuk menyamarkan bahwa satu plan menyentuh production dan dev sekaligus. Jika dua endpoint harus berubah dalam satu change, document dependency, ownership, failure behavior, dan rollback.

## 6. Promotion

Promotion bukan menyalin state dev ke staging/prod. Promotion berarti artifact/configuration yang sama atau commit yang sama diuji dan direview pada environment target dengan backend target sendiri.

```text
commit/module version
       ↓
fmt + validate
       ↓
plan dev → review → apply dev
       ↓
plan staging dengan input/backend staging → review → approval → apply
       ↓
plan prod dengan input/backend prod → change approval → apply
```

Plan binary terikat pada konfigurasi, provider, state, dan input tertentu. Jangan memakai plan artifact dev untuk apply terhadap state prod. Retensi artifact harus mengikuti policy dan artifact harus tidak memuat secret yang dapat diekstrak.

## 7. State Isolation dan Recovery

Setiap state key harus memiliki:

- nama environment dan service yang jelas;
- bucket/namespace yang sesuai;
- permission minimum;
- locking/concurrency yang diverifikasi;
- versioning/backup dan retention;
- owner serta prosedur recovery.

Contoh key yang jelas:

```text
modul-3.2/dev/web/terraform.tfstate
modul-3.2/staging/web/terraform.tfstate
modul-3.2/prod/web/terraform.tfstate
```

Jangan menjadikan `key = "terraform.tfstate"` untuk seluruh environment. Jangan menyalin raw state ke Git atau artifact umum. State dapat berisi atribut sensitif dari provider.

## 8. Environment Drift

Drift dapat berupa perubahan object, backend salah, provider version berbeda, variable input berbeda, atau state yang stale. Diagnosis:

1. hentikan mutation;
2. verifikasi directory, commit, workspace, backend, provider version, dan input;
3. jalankan `tofu plan` read-only;
4. cocokkan perbedaan dengan change ticket/manual action;
5. tentukan source of truth;
6. review remediation plan dan blast radius;
7. apply hanya setelah approval;
8. validasi provider dan service.

Plan kosong pada `dev` tidak membuktikan `staging` atau `prod` sama. Plan kosong juga tidak membuktikan object di luar schema provider identik.

## Troubleshooting

### State kosong atau resource tampak hilang

Periksa directory, workspace, backend key, dan credential permission. Jangan langsung import ulang atau `state rm`. Bandingkan `tofu state list`, konfigurasi backend, dan audit backend.

### `Backend configuration changed`

Stop dan baca perubahan endpoint/bucket/key. Backup dan approval diperlukan sebelum migrasi atau reconfigure. Jangan memakai flag migration untuk menghilangkan prompt tanpa memahami destination.

### Workspace salah

Jangan apply. Catat workspace, jalankan `tofu workspace show`, pilih workspace yang benar setelah verifikasi, lalu buat plan baru. Plan lama jangan digunakan bila context berubah.

### Plan environment menyentuh resource lain

Periksa module source, provider alias, backend, `for_each`, dan state. Pisahkan root module bila boundary tidak lagi jelas.

## Acceptance Checklist

- [ ] Tiga environment memiliki state key yang berbeda dan mudah dikenali.
- [ ] Perbandingan workspace/directory menyebutkan blast radius, permission, dan review.
- [ ] Preflight memeriksa directory, backend, workspace, dan provider.
- [ ] Promotion menggunakan commit/input/plan target yang benar, bukan state copy.
- [ ] Provider alias memiliki alasan dan scope yang terdokumentasi.
- [ ] Recovery/backup/locking dijelaskan tanpa credential literal.
- [ ] Tidak ada klaim remote backend runtime tanpa execution evidence.

## Catatan SRE

Environment adalah boundary perubahan. Ketika operator tidak dapat menjawab “state mana, provider mana, identity mana, dan owner siapa?”, operasi harus berhenti. Kecepatan apply tidak sebanding dengan biaya salah context.

## Kaitan dengan Modul Berikutnya

- [01 — Arsitektur Modul Reusable](01-arsitektur-modul-reusable.md) menjelaskan contract yang dipanggil environment.
- [03 — `for_each`, `count`, Conditional, Data Source](03-foreach-count-conditional-data-source.md) membahas address yang dihasilkan caller.
- [04 — Secret Handling dan CI Plan](04-secret-handling-dan-ci-plan.md) menerapkan gate CI dan secret boundary.
