# 03 — `for_each`, `count`, Conditional, dan Data Source

> **Tujuan:** memilih pola instansiasi yang stabil dan memahami kapan OpenTofu memiliki ownership terhadap object.

## Tujuan Belajar

Peserta dapat:

- memilih collection type yang tepat;
- menjelaskan address `for_each` dan `count`;
- mencegah address churn akibat rename key atau perubahan index;
- memakai conditional secara eksplisit;
- membedakan resource ownership dengan data source read-only.

## 1. `for_each` untuk Key yang Stabil

Gunakan `for_each` ketika instance memiliki identitas bisnis yang bermakna. Map key menjadi bagian address dan sebaiknya tidak berubah tanpa alasan.

```hcl
variable "web_instances" {
  description = "Instance web disposable dengan key bisnis stabil."
  type = map(object({
    host_port = number
    image     = string
  }))

  validation {
    condition     = alltrue([for key in keys(var.web_instances) : can(regex("^[a-z0-9][a-z0-9-]{0,30}$", key))])
    error_message = "Key instance harus lowercase dan stabil."
  }
}

module "web" {
  source   = "../../modules/web-server"
  for_each = var.web_instances

  name        = "sre-${each.key}"
  image       = each.value.image
  host_port   = each.value.host_port
  environment = var.environment
}

output "web_urls" {
  value = {
    for key, instance in module.web : key => instance.http_url
  }
}
```

Address yang dihasilkan:

```text
module.web["public"].docker_container.this
module.web["admin"].docker_container.this
```

Mengganti key `public` menjadi `frontend` bukan rename kosmetik. OpenTofu dapat melihat destroy `public` dan create `frontend`, kecuali perpindahan address direncanakan dan disetujui menggunakan mekanisme state yang tepat. Jangan mengandalkan `state mv` tanpa backup dan plan.

## 2. `count` untuk Instance Homogen

`count` cocok ketika instance benar-benar identik dan identitas index dapat diterima.

```hcl
resource "docker_container" "worker" {
  count = var.worker_count

  name  = "sre-worker-${count.index}"
  image = var.worker_image

  labels {
    label = "managed_by"
    value = "opentofu"
  }
}
```

Address berbentuk `docker_container.worker[0]`, `[1]`, dan seterusnya. Menghapus item di tengah list dapat menggeser index dan menyebabkan replacement pada instance lain. Jika instance memiliki nama, role, port, atau lifecycle berbeda, pilih `for_each` dengan map.

Validation count:

```hcl
variable "worker_count" {
  type    = number
  default = 2

  validation {
    condition     = var.worker_count >= 0 && var.worker_count <= 5
    error_message = "worker_count lab harus berada pada 0 sampai 5."
  }
}
```

Jangan memakai `count = var.enabled ? 1 : 0` untuk menyembunyikan perubahan berbahaya tanpa menjelaskan bahwa `false` menghapus resource. Conditional adalah keputusan lifecycle.

## 3. Conditional Resource

Conditional dapat dibuat melalui `for_each` pada map kosong atau `count` 0/1. Pilih pola yang membuat address dan plan mudah dibaca.

```hcl
variable "enable_probe" {
  description = "Buat probe HTTP disposable."
  type        = bool
  default     = true
}

resource "docker_container" "probe" {
  count = var.enable_probe ? 1 : 0

  name  = "sre-${var.environment}-probe"
  image = var.probe_image

  depends_on = [module.web]
}

output "probe_name" {
  value     = var.enable_probe ? docker_container.probe[0].name : null
  sensitive = false
}
```

Jika conditional membungkus module atau resource yang output-nya dipakai banyak tempat, pastikan caller menangani `null`/empty collection. Jangan memakai conditional untuk memilih credential atau endpoint secara diam-diam; environment boundary harus eksplisit.

Alternatif map untuk address bernama:

```hcl
resource "docker_container" "probe" {
  for_each = var.enable_probe ? { http = true } : {}

  name  = "sre-${var.environment}-probe"
  image = var.probe_image
}
```

## 4. Data Source dan Ownership

`data` membaca object yang dikelola boundary lain. Ia tidak memberikan ownership untuk update/delete.

```hcl
data "docker_network" "shared" {
  name = var.shared_network_name
}

resource "docker_container" "web" {
  name  = var.name
  image = var.image

  networks_advanced {
    name = data.docker_network.shared.name
  }
}
```

Sebelum memakai data source, dokumentasikan:

- siapa owner network;
- lifecycle dan availability contract;
- permission read-only yang diperlukan;
- apa yang terjadi jika object hilang;
- apakah nama lookup stabil atau bisa menunjuk object berbeda.

Data source tidak boleh dipakai untuk menghindari state ownership terhadap resource yang sebenarnya dikelola project ini. Jika module harus membuat network, gunakan `resource`; jika network shared dikelola platform team, gunakan data source dan contract.

## 5. Collection Type dan Validation

Gunakan type yang memperlihatkan bentuk data:

```hcl
variable "ports" {
  type = set(number)
  default = [8080, 8081]

  validation {
    condition     = alltrue([for port in var.ports : port >= 1024 && port <= 65535])
    error_message = "Semua port harus non-privileged dan valid."
  }
}
```

`set` tidak menjamin index; cocok untuk nilai unik tanpa urutan penting. `list` mempertahankan index tetapi rawan churn jika item di tengah dihapus. `map(object(...))` cocok untuk key stabil dan metadata per instance. Jangan mengubah jenis collection tanpa review karena address dan diff dapat berubah.

## 6. Dependency dan Lifecycle

Referensi seperti `docker_image.web.image_id` membuat dependency implicit. Gunakan `depends_on` hanya ketika dependency tidak terlihat dari expression, misalnya side effect provider atau prerequisite eksternal yang memang telah disepakati.

```hcl
module "web" {
  source = "../../modules/web-server"

  name  = "sre-web"
  image = docker_image.web.image_id
}
```

`depends_on` yang terlalu luas membuat graph serial, plan kurang informatif, dan replacement lebih besar. `lifecycle { create_before_destroy = true }` tidak selalu menghindari downtime; nama/port/provider constraint dapat mencegah coexistence. Uji pada disposable environment.

## 7. Address Review

Sebelum apply, cari perubahan berikut:

```text
+  module.web["new"].docker_container.this
-  module.web["old"].docker_container.this
-/+ module.web["public"].docker_container.this
```

Pertanyaan review:

1. Apakah key baru/rename benar-benar dimaksudkan?
2. Apakah port/nama dapat coexist saat replacement?
3. Apakah deletion menyentuh object yang owner-nya benar?
4. Apakah `count` index bergeser?
5. Apakah data source menunjuk object yang benar?
6. Apakah `depends_on` menyebabkan graph terlalu luas?

Gunakan `tofu graph` untuk memahami dependency, tetapi graph bukan pengganti plan review dan runtime validation.

## Troubleshooting

### `Invalid for_each argument`

Key `for_each` harus diketahui sebelum apply dan tidak boleh bergantung pada value yang belum diketahui. Bentuk input sebagai map key yang diketahui; jangan memakai ID resource yang baru dibuat sebagai key.

### Semua instance terganti setelah list berubah

Kemungkinan `count` index bergeser atau input immutable berubah. Bandingkan address dan pertimbangkan map `for_each` dengan key stabil. Migration address harus direncanakan, bukan dilakukan dengan edit state manual.

### Output conditional error index out of range

Resource dengan `count = 0` tidak memiliki `[0]`. Gunakan conditional yang sama atau `try(...)` dengan pemahaman terhadap null, lalu tambahkan acceptance test.

### Data source menemukan object yang salah

Nama lookup tidak cukup stabil atau environment/backend salah. Verifikasi provider endpoint, directory, identity, owner, dan identifier. Jangan membuat resource duplicate sebagai workaround.

### `depends_on` menghasilkan replacement besar

Periksa apakah dependency seharusnya implicit atau hanya pada resource tertentu. Kurangi edge setelah review plan dan test disposable.

## Acceptance Checklist

- [ ] `for_each` memakai map key stabil dan address dicatat.
- [ ] `count` hanya dipakai untuk instance homogen; index churn dijelaskan.
- [ ] Conditional menjelaskan dampak `false` terhadap lifecycle.
- [ ] Data source memiliki owner/read-only contract.
- [ ] Collection type dan validation sesuai semantics.
- [ ] Dependency graph tidak dibuat serial tanpa alasan.
- [ ] Plan review mencari create/delete/replace dan address change.
- [ ] Contoh tidak memuat secret atau resource production.

## Catatan SRE

Resource address adalah identitas operasional. Rename key, perubahan index, dan conditional false dapat menghasilkan deletion walaupun diff HCL tampak kecil. Perlakukan address sebagai API state dan review seperti database migration.

## Kaitan dengan Modul Berikutnya

- [04 — Secret Handling dan CI Plan](04-secret-handling-dan-ci-plan.md) membahas bagaimana plan dan input ini diproses oleh CI.
- [LAB-01 — Modul Reusable Web Server](lab/LAB-01-modul-reusable-web-server.md) mempraktikkan `for_each` pada resource disposable.
- Modul 3.3 — Konteks On-Prem masih menyusul; modul tersebut akan menerapkan pola collection pada provider on-prem.
