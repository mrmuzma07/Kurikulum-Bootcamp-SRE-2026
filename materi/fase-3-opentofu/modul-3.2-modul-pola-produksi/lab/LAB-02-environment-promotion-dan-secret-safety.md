# LAB-02 — Environment Promotion dan Secret Safety

> **Lane:** Production On-Prem Pattern + Static/OrbStack Walkthrough  
> **Durasi:** 3–4 jam  
> **Tujuan:** membandingkan directory dan workspace, mengisolasi backend/state key `dev` dan `staging`, menyusun promotion berbasis plan artifact, dan memeriksa boundary secret tanpa menyimpan credential.

> Lab ini tidak mengklaim remote backend, Vault, SOPS, CI, atau promotion production telah berjalan. Gunakan Jalur A bila runtime disposable tersedia; gunakan Jalur B untuk desain dan review statis.

## Tujuan Belajar

Peserta dapat:

- membuat root module `environments/dev` dan `environments/staging`;
- membedakan backend key dan provider context;
- menjelaskan mengapa plan dev tidak dapat dipakai untuk apply staging;
- menyiapkan CI gate yang fail-closed terhadap environment mismatch;
- menjaga secret di luar repository, output, state evidence, dan artifact umum.

## Guardrail dan Scope

- Hanya gunakan resource disposable dan endpoint lab yang disetujui.
- `prod` tidak dibuat atau disentuh oleh walkthrough ini.
- Verifikasi `pwd`, Git commit, backend, workspace, provider context, dan state sebelum mutation.
- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Placeholder seperti `<approved-lab-endpoint>` bukan credential dan tidak boleh diisi dengan nilai secret dalam repository.
- Jangan menjalankan `tofu apply`, `destroy`, import, atau state mutation sebelum plan dan scope disetujui.
- `-auto-approve` bukan default; CI apply production memerlukan approval terpisah.
- Jangan menguji pola ini dengan cluster production, `kubectl delete -A`, atau `k3s server --cluster-reset`.

## Layout Target

```text
lab-02/
├── modules/
│   └── web-server/                 # salin/reuse module dari LAB-01
└── environments/
    ├── dev/
    │   ├── backend.tf
    │   ├── providers.tf
    │   ├── variables.tf
    │   └── main.tf
    └── staging/
        ├── backend.tf
        ├── providers.tf
        ├── variables.tf
        └── main.tf
```

Setiap directory adalah root module. Jangan menjalankan `tofu` dari `lab-02/` jika file `.tf` hanya dimaksudkan untuk child environment.

## Jalur A — Runtime Disposable

### 1. Preflight dan Salin Module

```bash
set -eu

mkdir -p "$HOME/labs/opentofu/modul-3.2/lab-02/environments" "$HOME/labs/opentofu/modul-3.2/lab-02/modules"
cd "$HOME/labs/opentofu/modul-3.2/lab-02"
pwd
tofu version
docker context show
docker info --format '{{.OSType}}/{{.Architecture}}'
```

Salin `modules/web-server` dari LAB-01 melalui proses lokal yang direview. Jangan menyalin `.terraform`, state, plan, var file, atau credential. Verifikasi:

```bash
find modules -maxdepth 3 -type f -not -path '*/.terraform/*' -print
```

### 2. Environment `dev`

#### `environments/dev/backend.tf`

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
    key                         = "modul-3.2/dev/web/terraform.tfstate"
    region                      = "us-east-1"
    endpoint                    = "http://127.0.0.1:9000"
    use_path_style              = true
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
  }
}
```

Parameter S3-compatible dan locking harus diverifikasi terhadap versi backend. Jika MinIO tidak tersedia, backend block hanya menjadi desain; jangan mengklaim remote state berjalan.

#### `environments/dev/providers.tf`

```hcl
provider "docker" {}
```

#### `environments/dev/variables.tf`

```hcl
variable "web_image" {
  description = "Image multi-arch bertag immutable untuk lab."
  type        = string
  default     = "nginx:1.27.5"
}
```

#### `environments/dev/main.tf`

```hcl
module "web" {
  source = "../../modules/web-server"

  name        = "sre-dev-promotion-web"
  image       = var.web_image
  host_port   = 18084
  environment = "dev"

  labels = {
    lab = "modul-3.2-lab-02"
  }
}

output "promotion_metadata" {
  description = "Metadata non-secret untuk walkthrough promotion."
  value = {
    environment = "dev"
    name        = module.web.container_name
    http_url    = module.web.http_url
  }
}
```

### 3. Environment `staging`

Buat file setara di `environments/staging`, tetapi **backend key dan input harus berbeda sesuai scope**:

```hcl
terraform {
  backend "s3" {
    bucket                      = "sre-tofu-state"
    key                         = "modul-3.2/staging/web/terraform.tfstate"
    region                      = "us-east-1"
    endpoint                    = "http://127.0.0.1:9000"
    use_path_style              = true
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
  }
}
```

`providers.tf` tetap mendeklarasikan provider Docker sesuai versi yang dipin. `main.tf` dapat menggunakan:

```hcl
module "web" {
  source = "../../modules/web-server"

  name        = "sre-staging-promotion-web"
  image       = var.web_image
  host_port   = 18085
  environment = "staging"

  labels = {
    lab = "modul-3.2-lab-02"
  }
}
```

Jangan memakai state dev untuk staging. Jangan copy raw state. Promotion membawa commit/module version dan input yang disetujui, lalu membuat plan baru terhadap state staging.

### 4. Backend Init dan Read-Only Checks

Jalankan per directory, bukan dari parent:

```bash
cd environments/dev
pwd
tofu init
tofu workspace show
tofu providers
tofu fmt -check
tofu validate
tofu plan -out=dev.tfplan
tofu show -no-color dev.tfplan
```

Simpan plan hanya pada storage lokal/CI yang di-ignore dan memiliki retention sesuai policy. Kemudian lakukan pemeriksaan untuk staging:

```bash
cd ../staging
pwd
tofu init
tofu workspace show
tofu providers
tofu fmt -check
tofu validate
tofu plan -out=staging.tfplan
tofu show -no-color staging.tfplan
```

Pastikan:

```text
backend key dev     ≠ backend key staging
resource name dev   ≠ resource name staging
port dev             ≠ port staging
state list dev       tidak dibaca dari staging
plan dev             tidak diterapkan ke staging
```

Jika menggunakan local backend untuk simulasi, state tetap terpisah per directory dan tidak boleh di-commit.

### 5. Promotion Berbasis Commit dan Plan Baru

Simulasikan perubahan non-secret, misalnya image tag yang telah disetujui atau label version. Flow yang aman:

```text
1. commit perubahan module/configuration
2. fmt + validate pada dev
3. plan dev → review → approval → apply dev (bila diizinkan)
4. jalankan runtime/health check dev
5. gunakan commit/module version yang sama untuk staging
6. init/validate/plan baru di staging
7. review staging plan dan approval staging
8. apply staging plan yang dibuat dari staging context
9. validasi provider/service dan catat evidence
```

Jangan gunakan `dev.tfplan` pada directory staging. Plan terikat pada state, provider, input, dan context. Jika commit berubah, buat plan baru.

### 6. Secret Safety Walkthrough

Gunakan hanya nama variable/placeholder, bukan nilai secret:

```hcl
variable "bootstrap_secret" {
  description = "Diberikan oleh secret manager pada runtime; tidak disimpan di Git."
  type        = string
  sensitive   = true
}
```

Pada CI yang disetujui, secret dapat diinjeksi melalui secret mechanism menjadi `TF_VAR_bootstrap_secret` atau credential chain provider. Jangan menulis command seperti `export TF_VAR_bootstrap_secret='<nilai nyata>'` di README, issue, atau shell history.

Checklist:

```text
[ ] variable tidak memiliki default secret
[ ] secret source/identity/TTL/rotation terdokumentasi di luar nilai
[ ] output tidak mengembalikan secret
[ ] state backend memiliki permission/encryption/retention policy
[ ] plan/log/artifact tidak dibagikan secara publik
[ ] secret tidak dipakai sebagai environment selector
[ ] secret di-unset/direvoke sesuai lifecycle runner
```

`sensitive = true` bukan encryption. Bila resource provider mempersist value ke state, perlakukan state sebagai data sensitif.

### 7. CI Guardrail Environment

Contoh read-only guardrail:

```bash
set -eu

expected_environment="dev"
actual_environment="${TOFU_ENVIRONMENT:?TOFU_ENVIRONMENT wajib disediakan job}"

if [ "$actual_environment" != "$expected_environment" ]; then
  printf 'environment mismatch; job dihentikan\n' >&2
  exit 1
fi

case "$PWD" in
  */environments/dev) ;;
  *) printf 'directory bukan dev; job dihentikan\n' >&2; exit 1 ;;
esac

tofu fmt -check
tofu validate
tofu plan -out=dev.tfplan
```

Contoh di atas hanya pola. CI organisasi harus menguji path, commit, backend metadata, provider lock, identity, concurrency, artifact retention, dan approval. Apply production tidak boleh ditambahkan ke tahap merge request tanpa policy yang disetujui.

### 8. Apply dan Health Check (Opsional Disposable)

Hanya jika backend/runtime disposable benar-benar tersedia, scope plan sudah direview, dan approval ada:

```bash
# Dari environments/dev, gunakan plan dev yang baru direview.
tofu apply dev.tfplan

tofu output
tofu state list
curl --fail --silent --show-error http://127.0.0.1:18084
```

Ulangi pada staging dengan plan staging yang dibuat di directory staging dan port yang berbeda. Jangan menyatakan promotion berhasil hanya karena dua command apply keluar status sukses; validasi provider, HTTP, dependency, dan evidence tetap diperlukan.

### 9. Cleanup

Pada setiap environment, verifikasi state dan buat destroy plan terpisah:

```bash
tofu state list
tofu plan -destroy -out=destroy.tfplan
tofu show -no-color destroy.tfplan
```

Setelah approval dan scope check:

```bash
tofu apply destroy.tfplan
```

Jalankan cleanup hanya untuk resource dengan label `modul-3.2-lab-02` dan environment yang tepat. Jangan menghapus bucket/state sebelum evidence dan retention policy selesai.

## Jalur B — Static Review Tanpa MinIO/Vault/SOPS/CI

Jika runtime belum tersedia:

1. buat file `dev` dan `staging` dengan key berbeda;
2. lakukan review manual bahwa backend, port, name, dan path berbeda;
3. buat tabel predicted plan dan ownership;
4. tulis CI gate dan secret boundary tanpa nilai credential;
5. validasi link, fence, HCL shape, dan `.gitignore` secara statis;
6. tandai `tofu init/validate/plan/apply`, MinIO locking, HTTP, Vault/SOPS, dan CI promotion sebagai **belum diverifikasi**;
7. jangan mengganti keterbatasan dengan secret literal atau claim runtime.

## Workspace versus Directory Exercise

Jawab untuk lab ini:

| Pertanyaan | Directory | Workspace |
|---|---|---|
| Bagaimana menemukan scope sebelum command? | `pwd` + file backend | `pwd` + `tofu workspace show` |
| Bagaimana memisahkan key? | backend block tiap directory | backend key/workspace mapping yang disetujui |
| Apa risiko utama? | salah directory/provider | salah workspace dengan config sama |
| Kapan lebih tepat? | topology/permission/approval berbeda | variasi kecil dan homogen |

Kesimpulan default untuk production on-prem: gunakan directory/root module terpisah bila environment memiliki policy, identity, topology, atau lifecycle berbeda.

## Failure Modes

### `staging` membaca state `dev`

Stop mutation. Periksa backend key, workspace, `pwd`, `.terraform`, dan init output. Jangan menghapus state atau menggunakan `-reconfigure` secara membabi buta. Backup dan migration review diperlukan jika backend memang salah.

### Plan artifact salah environment

Jangan apply. Hapus/quarantine artifact sesuai retention policy, verifikasi metadata commit/backend/provider/input, lalu buat plan baru di directory target.

### Secret muncul pada artifact

Hentikan publication, revoke/rotate melalui owner, periksa log/cache, dan dokumentasikan incident. Perbaiki secret injection/masking; `sensitive` saja tidak cukup.

### `tofu init` gagal ke MinIO

Periksa endpoint, bucket, key, region, path style, credential helper, provider/backend version, dan permission tanpa mencetak credential. Jika belum tersedia, lanjutkan Jalur B.

### CI tidak mengenali environment

Fail closed. Pastikan allowlist path dan variable tidak dapat diubah oleh untrusted input. Jangan default ke `prod` atau menjalankan apply ketika selector kosong.

## Acceptance Criteria

- [ ] `dev` dan `staging` memiliki root module dan backend/state key berbeda.
- [ ] Workspace versus directory dibandingkan dengan contoh blast radius.
- [ ] Plan dibuat ulang pada target promotion; artifact tidak dipindahkan antar-state.
- [ ] CI gate menghentikan path/environment mismatch.
- [ ] Secret boundary, `sensitive`, state exposure, rotation, dan audit dijelaskan tanpa nilai nyata.
- [ ] Remote backend/locking/MinIO/Vault/SOPS/CI runtime ditandai sesuai evidence aktual.
- [ ] Cleanup hanya memakai destroy plan pada resource disposable berlabel.
- [ ] Tidak ada credential, raw state, atau plan binary di repository.

## Catatan SRE

Promotion adalah perubahan context, bukan sekadar urutan command. Bukti paling penting adalah relasi commit–module version–backend key–plan–approval–health check. Jika salah satu tidak dapat dihubungkan, hentikan promotion dan rekonstruksi evidence.

## Kaitan

- [02 — Environment, Workspace, dan State](../02-environment-workspace-dan-state.md)
- [04 — Secret Handling dan CI Plan](../04-secret-handling-dan-ci-plan.md)
- [LAB-01 — Modul Reusable Web Server](LAB-01-modul-reusable-web-server.md)
- [Modul 3.1 — State MinIO, Import, Drift](../../modul-3.1-dasar-opentofu/lab/LAB-02-state-minio-import-drift.md)
