# Latihan Modul 3.1 — Dasar OpenTofu

> Kerjakan tanpa credential nyata. Gunakan project disposable, placeholder, dan state lokal/remote lab yang sudah disetujui.

## Petunjuk

- Tulis alasan dan evidence, bukan hanya command.
- Untuk pertanyaan command, jelaskan scope, risiko, dan kondisi berhenti.
- Jika OpenTofu/OrbStack/MinIO tidak tersedia, kerjakan desain dan tandai runtime sebagai belum diverifikasi.
- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Latihan 1 — Imperative ke Declarative

1. Tulis langkah imperative untuk membuat network dan container web.
2. Ubah menjadi desired state declarative: resource apa yang diperlukan, input apa yang divalidasi, dan output apa yang diberikan?
3. Jelaskan mengapa mengulang langkah imperative dapat menghasilkan duplikasi atau error, sedangkan OpenTofu menghitung reconciliation.

**Output:** tabel `langkah imperative → blok HCL → acceptance check`.

## Latihan 2 — Tiga Kondisi Infrastruktur

Buat diagram yang menghubungkan:

```text
desired configuration → current infrastructure → state file → provider API
```

Berikan satu contoh ketika:

- HCL benar tetapi object provider berubah manual;
- state stale tetapi object provider masih sehat;
- plan kosong tetapi service aplikasi tetap unhealthy.

## Latihan 3 — HCL Dasar

Tulis konfigurasi minimal yang memiliki:

- `terraform.required_providers` untuk Docker dengan versi dipin;
- satu `docker_image`;
- satu `docker_container` yang mereferensikan image;
- variable `container_name` dan `host_port` dengan type/default/validation;
- `locals` untuk label;
- output URL dan ID container.

Tambahkan `.gitignore` yang mencegah state, plan binary, dan var file masuk Git. Jelaskan mengapa `.gitignore` bukan secret manager.

## Latihan 4 — Provider, Resource, Data Source

Jelaskan perbedaan ownership untuk:

1. `resource "docker_container" "web"`;
2. `data "docker_network" "existing"`;
3. provider Docker.

Kapan data source berbahaya bila dipakai untuk menghindari state ownership?

## Latihan 5 — Dependency Graph

Diberikan resource:

```hcl
resource "docker_image" "web" {
  name = var.web_image
}

resource "docker_container" "web" {
  image = docker_image.web.image_id
  name  = var.container_name
}
```

1. Tunjukkan edge dependency implicit.
2. Jelaskan kapan `depends_on` diperlukan.
3. Jelaskan risiko `depends_on` yang terlalu banyak.
4. Perintah apa yang menghasilkan graph untuk inspeksi?

## Latihan 6 — Review Workflow

Urutkan dan jelaskan tujuan command berikut:

```bash
tofu apply lab.tfplan
tofu validate
tofu init
tofu plan -out=lab.tfplan
tofu fmt -check
tofu output
tofu state list
```

Tambahkan titik approval dan validasi runtime. Jelaskan mengapa `tofu apply` tanpa plan yang direview bukan default yang baik.

## Latihan 7 — Plan Review dan Blast Radius

Sebuah plan berisi:

```text
+ docker_network.lab
+ docker_image.web
-/+ docker_container.web
- docker_container.unknown
```

1. Apakah plan boleh langsung di-apply? Mengapa?
2. Apa yang harus diverifikasi terkait `-/+`?
3. Bagaimana menyelidiki `docker_container.unknown`?
4. Mengapa `-target` bukan jawaban otomatis?

## Latihan 8 — Variable dan Secret

Bandingkan risiko berikut:

```bash
tofu plan -var='host_port=18080'
tofu plan -var='password=<secret>'
export TF_VAR_password='<secret>'
```

Mana yang aman untuk contoh non-secret? Jelaskan shell history, process listing, state, dan alternatif secret manager/environment policy.

## Latihan 9 — Local vs Remote State

Buat tabel perbandingan local state dan remote S3-compatible/MinIO berdasarkan:

- collaboration;
- locking;
- backup;
- permission;
- failure mode;
- network dependency;
- recovery.

Berikan rekomendasi untuk satu operator lab dan tim on-prem.

## Latihan 10 — Backend MinIO

Tulis backend block S3-compatible dengan placeholder untuk bucket, key, endpoint, dan region. Jelaskan:

- mengapa backend block tidak menerima variable biasa;
- dari mana credential seharusnya dibaca;
- mengapa access key/secret key tidak boleh literal;
- parameter compatibility yang harus diverifikasi per versi.

## Latihan 11 — Locking dan Concurrency

Dua operator menjalankan apply bersamaan terhadap backend key yang sama.

1. Jelaskan lost update dan state inconsistency.
2. Apa yang harus dilakukan ketika memperoleh state lock error?
3. Kapan force unlock dapat dipertimbangkan?
4. Mengapa menghapus lock object manual berbahaya?

## Latihan 12 — State Inspection

Tuliskan command untuk:

- melihat semua address state;
- melihat detail resource tertentu;
- melihat output JSON.

Tentukan bagaimana evidence disimpan tanpa mengekspos sensitive value. Jelaskan mengapa `sensitive = true` hanya menyamarkan tampilan.

## Latihan 13 — Import

Sebuah container Nginx dibuat manual oleh tim lab. Buat runbook import yang mencakup:

1. verifikasi owner/name/port/ID;
2. verifikasi backend/workspace;
3. backup state;
4. HCL resource block;
5. command import;
6. state show;
7. plan setelah import;
8. acceptance dan stop condition.

Jelaskan mengapa import tidak otomatis menghasilkan konfigurasi HCL lengkap.

## Latihan 14 — `state mv` vs `state rm`

Bandingkan dua operasi berikut:

```bash
tofu state mv docker_container.web module.web.docker_container.this
tofu state rm docker_container.orphan
```

Untuk masing-masing, jelaskan dampak pada object provider, state mapping, risiko, backup, dan alasan yang valid. Berikan satu alasan yang **tidak valid**.

## Latihan 15 — Drift Detection

Object provider berubah manual: port/label/image tidak lagi sama dengan HCL.

Tulis flow diagnosis dari verifikasi scope sampai validasi remediation. Pilih dua opsi:

- mengembalikan object ke desired state;
- menerima perubahan dengan mengubah HCL.

Jelaskan kapan setiap opsi lebih tepat.

## Latihan 16 — Recovery State

Directory project berpindah dari local backend ke MinIO, lalu `tofu init` meminta migrasi.

Buat checklist:

- backup;
- writer/lock;
- destination bucket/key;
- migration approval;
- inspect state;
- plan;
- rollback/recovery.

Apa yang tidak boleh dilakukan dengan copy state manual?

## Latihan 17 — Handoff OpenTofu → Ansible → k3s

Rancang output OpenTofu yang akan dipakai Fase 4 untuk bootstrap host/IP, tanpa menyimpan credential.

Jelaskan:

- metadata apa yang boleh menjadi output;
- bagaimana Ansible menerima inventory;
- mengapa OpenTofu tidak menggantikan configuration management;
- bagaimana hasil bootstrap menjadi prasyarat instalasi k3s;
- boundary ownership antara provisioning, OS config, cluster, dan aplikasi.

## Latihan 18 — Incident Drill

`tofu plan` akan mengganti container production-like karena provider version berubah. Operator ingin memakai `-auto-approve` dan `state rm` agar plan kosong.

Tulis respons incident yang mencakup:

- tindakan segera;
- evidence yang dikumpulkan;
- hal yang dilarang;
- diagnosis provider/lock/state;
- approval dan rollback;
- komunikasi ke owner.

## Rubrik

| Area | Bobot |
|---|---:|
| Konsep IaC, desired/current/state, idempotency | 20 |
| HCL/provider/resource/dependency | 20 |
| Workflow dan plan review | 15 |
| Remote state, locking, backup, recovery | 20 |
| Import, state operation, drift | 15 |
| Secret safety dan handoff Ansible/k3s | 10 |
| **Total** | **100** |

Target kelulusan: **80/100**. Jawaban yang berisi credential nyata, destroy tanpa scope, atau menghapus lock/state tanpa verifikasi tidak lulus meskipun command teknisnya benar.
