# LAB-02 — State MinIO, Import, dan Drift

> **Lane:** Local / Docker / OrbStack Fast Lane  
> **Durasi:** 3–4 jam  
> **Tujuan:** memahami remote S3-compatible state, locking, import resource manual, drift detection, dan pilihan remediasi pada environment disposable.

> **Penting:** walkthrough ini menjelaskan prosedur dan command. Jangan mengklaim MinIO, remote backend, locking, import, atau drift berhasil sebelum command benar-benar dijalankan dan evidence disimpan secara aman.

## Tujuan Belajar

Peserta dapat:

- menjalankan MinIO disposable di OrbStack atau menyediakan endpoint S3-compatible yang disetujui;
- mengonfigurasi backend tanpa menulis access key/secret key pada repository;
- menjelaskan mengapa satu state hanya boleh memiliki satu writer pada satu waktu;
- mengimpor container Docker manual ke state;
- membuat drift terkontrol, mendeteksinya dengan plan, dan memilih remediasi;
- membuat backup dan inspeksi state tanpa membocorkan secret.

## Guardrail dan Scope

- Gunakan project baru `lab-02`, bucket dan key khusus lab.
- Pastikan Docker context dan backend bukan environment production.
- Credential MinIO hanya diberikan melalui environment/credential helper lokal yang permission-nya sesuai. Jangan menulis credential dalam HCL, README, shell history, issue, atau `.tfvars` yang masuk Git.
- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menjalankan concurrent apply/import/state mutation pada backend key yang sama.
- Jangan force unlock atau menghapus lock object manual tanpa verifikasi owner, lock ID, dan approval.
- Semua container, bucket, dan state pada lab ini disposable. Jangan mengimpor object production.

## 0. Preflight

```bash
tofu version
docker version
docker context show
docker info --format '{{.OSType}}/{{.Architecture}}'
```

Buat directory project:

```bash
mkdir -p "$HOME/labs/opentofu/modul-3.1/lab-02"
cd "$HOME/labs/opentofu/modul-3.1/lab-02"
pwd
```

Simpan output preflight hanya sebagai evidence lokal yang sudah diredáksi. Jangan menyimpan environment variable credential ke file.

## 1. Jalur A — Menjalankan MinIO Disposable di OrbStack

Gunakan image/tag MinIO yang telah disetujui dan mendukung arsitektur runtime. Contoh berikut memakai placeholder yang harus diganti dengan tag yang benar-benar diverifikasi:

```bash
docker volume create sre-tofu-minio-data

docker run -d \
  --name sre-tofu-minio \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_ROOT_USER="$MINIO_LAB_ACCESS_KEY" \
  -e MINIO_ROOT_PASSWORD="$MINIO_LAB_SECRET_KEY" \
  -v sre-tofu-minio-data:/data \
  <MINIO_IMAGE>:<APPROVED_TAG> \
  server /data --console-address ":9001"
```

Sebelum menjalankan, set credential hanya pada environment session dari secret mechanism lokal yang disetujui. Jangan menulis nilainya pada file atau command history. Jika shell history merekam command, gunakan mekanisme redaction/history policy organisasi.

Verifikasi tanpa mencetak credential:

```bash
docker ps --filter name=sre-tofu-minio
curl --fail --silent --show-error http://127.0.0.1:9000/minio/health/live
```

Buat bucket memakai MinIO client/console yang telah disetujui, atau tool S3-compatible yang tersedia. Contoh konseptual menggunakan environment variable:

```bash
mc alias set lab-minio http://127.0.0.1:9000 "$MINIO_LAB_ACCESS_KEY" "$MINIO_LAB_SECRET_KEY"
mc mb --ignore-existing lab-minio/sre-tofu-state
```

Jika `mc` tidak tersedia atau tag MinIO belum diverifikasi, gunakan Jalur B atau walkthrough statis. Jangan mengunduh binary acak.

### Jalur B — Backend S3-Compatible yang Sudah Ada

Jika organisasi menyediakan endpoint S3-compatible disposable, catat secara internal:

```text
endpoint: <approved-lab-endpoint>
bucket:   <approved-lab-bucket>
key:      modul-3.1/lab-02/terraform.tfstate
region:   <approved-region>
```

Pastikan bucket/key hanya digunakan state lab dan credential memiliki permission minimum. Jika endpoint tidak tersedia, lanjutkan bagian local-state import/drift dan tandai acceptance remote backend sebagai **belum dijalankan**.

## 2. Project HCL dengan Backend S3

Backend harus berada dalam block literal; backend tidak menerima interpolasi variable biasa. Endpoint, bucket, dan key pada contoh berikut bukan credential:

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

  backend "s3" {
    bucket                      = "sre-tofu-state"
    key                         = "modul-3.1/lab-02/terraform.tfstate"
    region                      = "us-east-1"
    endpoint                    = "http://127.0.0.1:9000"
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
    use_path_style              = true
  }
}

provider "docker" {}
```

Nilai backend harus disesuaikan dengan endpoint yang disetujui. OpenTofu/backend version dapat memiliki parameter locking berbeda. Periksa `tofu init` dan dokumentasi versi yang dipakai sebelum mengaktifkan fitur locking; jangan mengasumsikan kombinasi parameter Terraform lama selalu identik.

### `variables.tf`

```hcl
variable "container_name" {
  type    = string
  default = "sre-tofu-imported-web"

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{0,62}$", var.container_name))
    error_message = "Gunakan nama DNS-label lowercase."
  }
}

variable "host_port" {
  type    = number
  default = 18082

  validation {
    condition     = var.host_port >= 1024 && var.host_port <= 65535
    error_message = "Gunakan port non-privileged."
  }
}
```

### `main.tf`

```hcl
resource "docker_container" "manual" {
  name  = var.container_name
  image = "nginx:1.27.5"

  ports {
    internal = 80
    external = var.host_port
  }
}
```

Tag image harus diverifikasi dan dapat diganti dengan tag multi-arch yang disetujui. Jangan menaruh credential atau secret pada variable.

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

## 3. Init Remote Backend

Set credential pada environment melalui secret mechanism yang disetujui. Jangan menampilkan nilainya:

```bash
export AWS_ACCESS_KEY_ID="$MINIO_LAB_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="$MINIO_LAB_SECRET_KEY"
```

Jika environment variable belum tersedia, jangan mengisi placeholder dengan credential di file. Ambil dari credential helper/secret manager lokal sesuai policy.

Inisialisasi dan review prompt:

```bash
tofu init
```

Verifikasi:

```bash
tofu providers
tofu fmt -check
tofu validate
```

Jika `init` menawarkan migration, pastikan ini directory `lab-02`, backup local state bila ada, dan tujuan bucket/key benar. Jangan memakai `-reconfigure` atau `-migrate-state` untuk menghilangkan prompt tanpa backup dan review.

Untuk backend yang mendukung locking, jalankan hanya satu writer dan ikuti konfigurasi resmi OpenTofu versi yang dipakai. Jangan menyatakan locking aktif hanya karena backend S3 dapat menyimpan object.

## 4. Plan dan Apply Resource Lab

```bash
tofu plan -out=lab-02.tfplan
tofu show -no-color lab-02.tfplan
```

Review bahwa hanya `docker_container.manual` akan dibuat/adopted sesuai tujuan. Setelah approval:

```bash
tofu apply lab-02.tfplan
```

Inspeksi:

```bash
tofu state list
tofu state show docker_container.manual
docker ps --filter name=sre-tofu-imported-web
curl --fail --silent --show-error http://127.0.0.1:18082
```

Jika apply dilakukan, state sekarang berada di backend yang dikonfigurasi. Jangan mengunduh atau menyalin raw state ke repository. Backup harus mengikuti policy backend.

## 5. Demonstrasi Locking dan Concurrency

Locking hanya dapat diuji bila backend dan konfigurasi versi benar-benar mendukungnya.

1. Pastikan satu terminal menjalankan perubahan lab yang cukup lama atau gunakan dua operator pada key lab yang sama dengan approval.
2. Jalankan plan/apply kedua hanya sebagai uji controlled, bukan terhadap production.
3. Amati apakah writer kedua menunggu atau gagal karena lock.
4. Catat lock behavior, timeout, dan output tanpa credential.
5. Hentikan uji jika resource atau backend berperilaku di luar scope.

Jangan membuat dua apply destructive hanya untuk memaksa race. Jangan menghapus lock object manual. Jika lock tertinggal setelah proses mati, verifikasi tidak ada writer, identifikasi lock ID/owner, backup state, lalu ikuti prosedur `force-unlock` resmi dengan approval.

Jika backend tidak tersedia, tulis hasil: `locking runtime belum diverifikasi`; jelaskan mekanisme secara konseptual dan lanjutkan latihan state local.

## 6. Import Resource Manual

Gunakan container manual yang jelas owner dan port-nya. Untuk menghindari conflict dengan resource yang sudah dikelola, destroy/rename resource lab sebelumnya atau gunakan nama/port khusus.

Buat object manual:

```bash
docker run -d \
  --name sre-tofu-manual-web \
  -p 18083:80 \
  <APPROVED_NGINX_IMAGE>:<APPROVED_TAG>
```

Sebelum import, ubah HCL agar `docker_container.manual` merepresentasikan object tersebut, misalnya gunakan `container_name = "sre-tofu-manual-web"` dan `host_port = 18083` melalui local var file yang di-ignore. Jangan menyalin ID atau credential ke Git.

Cari ID hanya dari output lokal yang aman:

```bash
docker inspect --format '{{.Id}}' sre-tofu-manual-web
```

Backup state melalui mekanisme backend yang didukung, lalu import dengan provider syntax yang telah diverifikasi:

```bash
tofu state list
tofu import docker_container.manual <MANUAL_CONTAINER_ID>
tofu state show docker_container.manual
tofu plan
```

`<MANUAL_CONTAINER_ID>` adalah placeholder, bukan command literal. Import tidak otomatis menghasilkan HCL lengkap. Plan setelah import harus direview; perubahan yang banyak dapat berarti HCL tidak cocok, default provider berbeda, atau ID salah.

## 7. Simulasi Drift

Drift hanya diuji pada object disposable yang telah diimport/dikelola lab. Contoh perubahan manual:

```bash
docker rename sre-tofu-manual-web sre-tofu-manual-web-drifted
```

Jalankan plan:

```bash
tofu plan
```

Provider mungkin melaporkan object hilang atau memerlukan replacement, bergantung schema/version. Jangan mengasumsikan output tertentu tanpa runtime test.

Alternatif drift atribut yang tidak destructive:

```bash
docker update --label-add drift=test sre-tofu-manual-web-drifted
```

Kemudian jalankan `tofu plan` dan amati apakah provider mengekspos label tersebut sebagai diff. Jika tidak ada diff, catat bahwa atribut itu bukan bagian dari schema/state yang dikelola; plan kosong bukan bukti object sepenuhnya identik.

## 8. Pilih Remediasi

### Remediasi A — Kembalikan desired state

Jika perubahan manual tidak disetujui:

```text
1. verifikasi resource address dan backend
2. baca plan drift
3. pastikan replacement/downtime/data loss dapat diterima
4. minta approval
5. apply plan baru
6. validasi nama, port, dan HTTP
```

Jangan menjalankan apply hanya untuk menghilangkan diff jika root cause belum dipahami.

### Remediasi B — Terima perubahan ke HCL

Jika perubahan manual valid, ubah HCL agar desired state baru eksplisit, pin image/provider, lalu jalankan `fmt`, `validate`, dan `plan`. HCL harus menjadi source of truth; jangan mengedit state langsung untuk mengakomodasi perubahan.

### Remediasi C — Lepaskan ownership

`tofu state rm` dapat melepas mapping tanpa menghapus object provider pada banyak provider, tetapi object menjadi unmanaged dan cleanup berikutnya harus jelas:

```bash
# Hanya setelah backup, scope check, lock, dan approval.
tofu state rm docker_container.manual
```

Jangan memakai ini sebagai shortcut. Catat siapa owner baru dan bagaimana object akan dibersihkan.

## 9. Inspeksi dan Backup Evidence

Command inspeksi:

```bash
tofu state list
tofu state show docker_container.manual
tofu output -json
tofu show
```

Untuk backup, gunakan fitur backend/object storage yang disetujui atau snapshot versioning. Jangan memakai `cp` raw state ke directory yang akan di-commit. Setiap backup harus memiliki permission terbatas, retention, dan prosedur restore disposable.

Evidence minimal tanpa secret:

```text
- tofu version
- provider source/version
- backend bucket/key yang tidak mengandung credential
- action plan dan resource address
- hasil state list yang sudah diredáksi
- hasil HTTP/health lokal
- status locking: diverifikasi/belum diverifikasi
- status import/drift: dijalankan/belum dijalankan
```

## 10. Cleanup

Jika resource masih dikelola state, buat dan review destroy plan:

```bash
tofu plan -destroy -out=destroy.tfplan
tofu show -no-color destroy.tfplan
tofu apply destroy.tfplan
```

Jika object telah dilepas dengan `state rm`, cleanup manual memerlukan verifikasi nama/ID dan owner:

```bash
docker ps -a --filter name=sre-tofu-manual-web
docker ps -a --filter name=sre-tofu-manual-web-drifted
```

Hapus hanya object yang dibuat lab dan telah diverifikasi. Bersihkan MinIO/container/volume lab setelah backup/evidence tidak lagi diperlukan dan retention policy mengizinkan:

```bash
docker rm -f sre-tofu-minio
docker volume rm sre-tofu-minio-data
```

Command cleanup di atas **tidak boleh** dijalankan bila nama volume/container tidak diverifikasi sebagai resource lab.

## Failure Modes

### `AccessDenied` atau bucket tidak ditemukan

Periksa endpoint, bucket/key, region, path style, permission, dan credential helper tanpa mencetak credential. Jangan mengganti access policy menjadi public sebagai workaround.

### Backend S3 init gagal karena parameter unsupported

Baca versi OpenTofu/backend documentation. Parameter S3-compatible dan locking dapat berbeda antarversi. Hapus hanya parameter yang memang tidak didukung setelah review; jangan mematikan validation/locking tanpa memahami risiko.

### Lock timeout

Cari writer aktif, cek proses/CI yang menggunakan key sama, tunggu atau hentikan melalui prosedur. Force unlock hanya untuk lock stale yang diverifikasi.

### Import menghasilkan replacement

Bandingkan HCL dengan `tofu state show` dan konfigurasi aktual. Replacement dapat menghapus/recreate object; jangan apply sebelum downtime/data loss disetujui.

### Drift tidak terlihat

Resource mungkin tidak mengelola atribut tersebut, provider tidak refresh seperti yang diharapkan, atau backend/state salah. Verifikasi scope dan provider schema. Plan kosong bukan bukti tidak ada perubahan.

## Acceptance Criteria

- [ ] Backend S3-compatible/MinIO berhasil diinisialisasi, atau keterbatasan endpoint dicatat.
- [ ] Credential tidak ada dalam HCL, Git, evidence, atau command yang disalin.
- [ ] Locking/concurrency behavior diverifikasi, atau statusnya jelas `belum diverifikasi`.
- [ ] Resource manual diimpor hanya bila ownership dan ID terverifikasi, atau walkthrough statis diselesaikan.
- [ ] Drift dibuat pada resource disposable dan plan dibaca, atau runtime belum tersedia dicatat.
- [ ] Remediasi dipilih berdasarkan source of truth, bukan dengan mengedit state secara membabi buta.
- [ ] Backup, inspection, dan cleanup scope terdokumentasi.

## Handoff

- [04 — State, Remote, Import, Drift](../04-state-remote-import-drift.md) menjelaskan konsep dan failure mode.
- Modul 3.2 akan membahas environment isolation, reusable modules, dan secret integration.
- Modul 3.3 akan menghubungkan metadata provisioning OpenTofu ke Ansible lalu k3s.
