# Latihan Modul 3.2 — Modul dan Pola Produksi

> Kerjakan dengan resource disposable, placeholder, dan evidence yang sudah diredáksi. Jangan gunakan credential nyata.

## Petunjuk Umum

- Jelaskan alasan, ownership, blast radius, dan stop condition; jangan hanya menyalin command.
- Bedakan konfigurasi, state, provider object, plan artifact, dan runtime health.
- Jika OpenTofu, Docker, OrbStack, MinIO, Vault, SOPS, atau CI tidak tersedia, kerjakan desain dan tandai runtime **belum diverifikasi**.
- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Latihan 1 — Contract Module

Rancang child module `web-server` dengan:

1. input `name`, `image`, `host_port`, `environment`, dan `labels`;
2. type, description, default/validation yang tepat;
3. output minimum untuk runtime check dan handoff;
4. provider requirement dan cara provider diteruskan caller;
5. ownership boundary yang tidak ambigu.

**Output:** tabel input/output/provider/owner/failure mode.

## Latihan 2 — Root dan Child Module

Gambarkan hubungan:

```text
environments/dev/main.tf
  → module.web source ../../modules/web-server
      → docker_image.this
      → docker_container.this
```

Jelaskan command yang dijalankan dari root module dan mengapa menjalankan dari directory parent dapat menghasilkan context salah.

## Latihan 3 — Review Abstraksi

Sebuah module menerima 35 variable untuk Docker, Proxmox, vSphere, dan libvirt sekaligus.

1. Identifikasi minimal empat failure mode.
2. Usulkan pemisahan module yang lebih mudah diuji.
3. Tentukan kapan abstraction cukup stabil untuk dipakai lintas environment.

## Latihan 4 — Layout Environment

Buat layout `environments/dev`, `environments/staging`, dan `environments/prod`.

Tulis backend key yang berbeda untuk tiap environment dan jelaskan:

- permission/identity;
- locking/concurrency;
- backup/recovery;
- approval;
- provider endpoint;
- cara mencegah accidental prod apply.

Gunakan placeholder endpoint/bucket, bukan credential.

## Latihan 5 — Workspace versus Directory

Bandingkan workspace dan directory per environment dalam tabel berdasarkan:

- visibility context;
- topology difference;
- permission boundary;
- state key;
- blast radius;
- recovery;
- penggunaan yang direkomendasikan.

Berikan rekomendasi untuk tim on-prem dengan environment production yang memerlukan approval berbeda.

## Latihan 6 — Provider Alias

Tuliskan contoh root module yang memiliki provider alias `docker.lab` dan `docker.shared`, lalu meneruskan hanya `docker.lab` ke child module web.

Jelaskan:

1. mengapa alias dibutuhkan;
2. bagaimana memastikan module tidak menyentuh endpoint salah;
3. mengapa alias bukan security boundary otomatis.

## Latihan 7 — `for_each` Stable Key

Diberikan:

```hcl
variable "web_instances" {
  type = map(object({
    host_port = number
    image     = string
  }))
}

module "web" {
  source   = "./modules/web-server"
  for_each = var.web_instances

  name      = "sre-${each.key}"
  image     = each.value.image
  host_port = each.value.host_port
}
```

1. Tulis address untuk key `public` dan `admin`.
2. Jelaskan dampak rename `admin` menjadi `backoffice`.
3. Tulis checklist sebelum state migration.
4. Mengapa key dari ID resource yang belum diketahui berbahaya?

## Latihan 8 — `count` dan Index Churn

Sebuah resource memakai `count = 3` dan nama berdasarkan `count.index`. Operator menghapus item index 1 dari input list.

1. Prediksi address yang dapat berubah.
2. Jelaskan risiko replacement.
3. Rekomendasikan `for_each` map bila instance memiliki role/nama.
4. Tulis acceptance check untuk memastikan tidak ada delete tak terduga.

## Latihan 9 — Conditional

Tulis resource probe yang hanya dibuat ketika `enable_probe = true`. Jelaskan:

- address saat true/false;
- dampak false terhadap lifecycle;
- cara output menangani resource yang tidak ada;
- alasan conditional tidak boleh diam-diam memilih endpoint production.

## Latihan 10 — Data Source dan Ownership

Bandingkan:

```hcl
data "docker_network" "shared" {
  name = var.shared_network_name
}
```

dengan resource network yang dibuat module.

Jelaskan owner, failure mode jika object hilang, permission, dan kapan data source menjadi cara berbahaya untuk menghindari state ownership.

## Latihan 11 — Dependency Graph

Gunakan module web dan image resource.

1. Tunjukkan dependency implicit dari image ke container.
2. Jelaskan kapan `depends_on` diperlukan.
3. Mengapa `depends_on` terlalu luas dapat memperbesar plan/replacement?
4. Tulis command untuk menghasilkan graph.

## Latihan 12 — Backend/State Isolation

Dua directory menggunakan:

```hcl
backend "s3" {
  bucket = "sre-tofu-state"
  key    = "terraform.tfstate"
}
```

1. Apa risiko konfigurasi ini?
2. Perbaiki menjadi key `dev` dan `staging`.
3. Tulis preflight sebelum `tofu init`.
4. Apa yang dilakukan jika state terlihat kosong?

## Latihan 13 — Promotion

Susun flow promotion dari commit module ke `dev`, `staging`, lalu production on-prem. Wajib memuat:

- `fmt`, `validate`, `plan`;
- plan artifact dan metadata;
- approval;
- plan baru pada target backend;
- health check;
- rollback;
- handoff Ansible.

Jelaskan mengapa plan binary dev tidak boleh diterapkan ke staging.

## Latihan 14 — Secret Boundary

Bandingkan risiko:

```text
variable default berisi password
-var='password=<secret>'
TF_VAR_password dari secret helper
Vault/SOPS/secret manager pada runner
```

Jelaskan shell history, process listing, state, log, artifact, rotation, dan audit. Jangan menulis nilai secret.

## Latihan 15 — `sensitive` dan State

Jelaskan mengapa:

```hcl
output "credential" {
  value     = var.credential
  sensitive = true
}
```

belum cukup untuk menjamin secret aman. Berikan minimal lima lokasi exposure dan lima control.

## Latihan 16 — CI Guardrail

Tulis pseudo-pipeline yang:

1. fail jika path bukan environment yang diizinkan;
2. menjalankan `tofu fmt -check`, `tofu validate`, `tofu plan`;
3. menolak `-auto-approve` sebagai default;
4. menyimpan artifact secara terbatas;
5. meminta approval sebelum apply;
6. memvalidasi backend key dan commit sebelum apply.

## Latihan 17 — Handoff OpenTofu → Ansible

Rancang output non-secret untuk inventory:

```text
hostname, address, role, environment, module_version
```

Jelaskan boundary OpenTofu, Ansible, k3s, Helm, dan GitOps. Apa yang tidak boleh menjadi output biasa?

## Latihan 18 — Incident Drill Environment Salah

CI yang seharusnya plan `dev` ternyata berjalan dari `environments/prod` dan menunjukkan delete/replace.

Tulis respons:

- tindakan segera;
- evidence yang dikumpulkan;
- siapa yang dihubungi;
- hal yang dilarang;
- cara memperbaiki pipeline;
- kondisi restart.

## Latihan 19 — Incident Drill Secret Exposure

Plan artifact CI tampak mengandung nilai sensitif.

Tulis runbook revoke/rotate, quarantine artifact, audit log/cache, komunikasi owner, dan perubahan prevention. Jangan menyalin nilai yang terlihat.

## Latihan 20 — Static versus Runtime Evidence

Buat tabel untuk status berikut:

| Check | Command/design | Evidence | Status |
|---|---|---|---|
| module contract | ... | ... | ... |
| `tofu validate` | ... | ... | ... |
| remote state | ... | ... | ... |
| Vault/SOPS | ... | ... | ... |
| promotion | ... | ... | ... |

Pastikan status “belum diverifikasi” tidak ditulis sebagai “berhasil”.

## Rubrik

| Area | Bobot |
|---|---:|
| Module contract, composition, provider passing | 20 |
| Environment/workspace/state isolation | 20 |
| `for_each`, `count`, conditional, data source | 20 |
| Secret safety dan state/artifact exposure | 15 |
| CI plan, approval, promotion | 15 |
| Handoff Ansible dan operational evidence | 10 |
| **Total** | **100** |

Target kelulusan: **80/100**. Credential nyata, secret commit, apply/destroy pada scope salah, atau state mutation tanpa backup/approval dapat menggugurkan submission.
