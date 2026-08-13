# 03 — Workflow `init → plan → apply → destroy`

> Workflow OpenTofu yang aman memisahkan syntax check, dependency initialization, plan review, approved change, validation, dan cleanup.

## Tujuan

- Menjalankan workflow dasar pada project disposable.
- Membedakan command read-only dari command yang mengubah infrastructure/state.
- Membaca action pada plan sebelum apply.
- Menghindari apply/destroy pada project atau context yang salah.

## 1. Verifikasi Tool dan Runtime

Pada Mac Apple Silicon:

```bash
tofu version
docker version
docker context show
docker info --format '{{.OSType}}/{{.Architecture}}'
```

Expected architecture untuk lab adalah `darwin_arm64` pada CLI dan `linux/arm64` pada runtime/container image bila native ARM dipakai. Bila hanya tersedia image `amd64`, catat penggunaan emulasi dan dampaknya; jangan menganggap hasil performa setara.

Command ini read-only terhadap infrastructure. Jika `tofu` belum ada:

```bash
brew install opentofu
```

Gunakan versi package yang telah disetujui dan catat hasil `tofu version` sebagai evidence. Jangan mengunduh executable dari URL acak.

## 2. Struktur Project dan `.gitignore`

```text
lab-tofu/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── terraform.tfvars.example
└── .gitignore
```

Contoh `.gitignore` minimal:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfplan
crash.log
crash.*.log
```

`.gitignore` mengurangi risiko commit tidak sengaja, tetapi bukan secret manager. State tetap harus diproteksi melalui permission, backend policy, encryption, dan access control.

## 3. `tofu init`

```bash
tofu init
```

`init` mengunduh provider/module, membuat direktori `.terraform`, dan membaca backend. Periksa provider source, version, lock file, serta backend yang digunakan sebelum lanjut.

Jika project berpindah backend atau directory state, `init` dapat meminta migrasi. Hentikan dan review backup/scope sebelum menjawab prompt migrasi. Jangan memakai `-reconfigure` untuk mengabaikan warning tanpa memahami konsekuensi; jangan memakai `-migrate-state` pada state penting tanpa approval dan backup.

## 4. `tofu fmt` dan `tofu validate`

```bash
tofu fmt -check
tofu fmt -diff
tofu validate
```

- `fmt -check` mendeteksi file yang belum terformat.
- `fmt -diff` membantu review perubahan format.
- `validate` memeriksa sintaks dan konsistensi internal module; ia tidak membuktikan provider API atau runtime sehat.

Jika format belum sesuai, jalankan `tofu fmt` pada project lab lalu review diff Git. Jangan menjalankan formatter pada directory production yang tidak sedang direview.

## 5. `tofu plan`

```bash
tofu plan -out=lab.tfplan
```

Plan membaca configuration, state, dan provider refresh lalu menampilkan action:

```text
+ create
~ update in-place
-/+ replace
- destroy
```

Review minimal:

- workspace/backend dan project yang benar;
- resource address dan jumlah perubahan;
- perubahan `destroy` atau `replace`;
- port, network, volume, dan image tag;
- output yang mungkin memuat value sensitif;
- dependency dan blast radius;
- hasil dibandingkan change ticket/acceptance criteria.

Plan binary dapat mengandung data sensitif dari state. Simpan hanya di directory lab yang di-ignore dan hapus sesuai retention policy. Jangan mengunggah `lab.tfplan` ke issue publik.

Untuk menghasilkan plan tanpa membuat file binary:

```bash
tofu plan
```

Jangan menjadikan `plan` sebagai formal approval otomatis. Approval datang dari reviewer yang memahami action dan scope.

## 6. `tofu apply`

Default yang aman adalah menerapkan plan setelah review:

```bash
tofu apply lab.tfplan
```

Atau:

```bash
tofu apply
```

Command kedua menyusun ulang plan dan meminta konfirmasi. Pada production, gunakan process approval dan policy yang berlaku. Jangan menjadikan `-auto-approve` default.

Untuk lab disposable, `-auto-approve` hanya boleh dipakai jika dokumentasi secara eksplisit membatasi project, resource, identity, network, stop condition, dan cleanup. Contoh tidak diberikan sebagai command copy-paste agar peserta tidak menganggapnya sebagai normal workflow.

Jika apply berhenti di tengah jalan, jangan langsung mengulang atau menghapus state. Baca error provider, periksa object aktual dan state, lalu jalankan plan baru setelah penyebab dipahami.

## 7. Output dan Validasi Setelah Apply

```bash
tofu output
tofu output -json
tofu state list
tofu state show docker_container.web
```

`tofu output -json` dapat menampilkan value yang tidak terlihat pada output biasa; redirect hanya ke lokasi lokal yang aman dan jangan commit. Untuk output sensitive, `<sensitive>` adalah perlindungan presentasi, bukan enkripsi state.

Validasi runtime lab:

```bash
docker ps --filter name=sre-tofu-web
curl --fail --silent --show-error http://127.0.0.1:<port>
```

Ganti `<port>` dengan variable lab yang disetujui. `curl` hanya membuktikan HTTP path lokal, bukan health semua dependency.

## 8. Refresh dan Reconcile

Perubahan manual pada provider dapat ditemukan pada plan berikutnya:

```bash
tofu plan
```

Pada versi OpenTofu yang dipakai, jangan mengandalkan command lama yang mengubah state tanpa perubahan infrastructure. Perlakukan refresh/reconcile sebagai aktivitas yang perlu dipahami dari output versi CLI. Plan normal adalah titik awal review drift.

## 9. Variable File

Gunakan file var non-secret:

```bash
tofu plan -var-file=lab.auto.tfvars
```

Pastikan `lab.auto.tfvars` berada di `.gitignore` bila berisi nilai lokal atau data sensitif. Untuk parameter non-secret sederhana:

```bash
tofu plan -var='host_port=18080'
```

Jangan memasukkan password/token ke argumen command line. Gunakan environment variable atau secret manager sesuai kebijakan organisasi. Ingat bahwa `TF_VAR_*` dapat terlihat dalam environment process dan harus memiliki permission yang sesuai.

## 10. `tofu destroy` yang Terbatas

Destroy adalah command berisiko karena menghapus object yang dikelola state:

```bash
tofu plan -destroy -out=destroy.tfplan
tofu apply destroy.tfplan
```

Sebelum plan destroy:

```text
[ ] directory project lab sudah diverifikasi
[ ] backend/workspace sudah diverifikasi
[ ] current owner dan resource address sudah dicatat
[ ] tidak ada production credential/provider alias
[ ] data yang dibutuhkan sudah dibackup
[ ] scope destroy hanya resource disposable
[ ] reviewer menyetujui plan
```

`tofu destroy` tanpa target dapat menghapus seluruh resource dalam state. Jangan memberikan `-target` sebagai jalan pintas untuk cleanup tanpa memahami dependency; target juga dapat menghasilkan state parsial. Bila harus melakukan targeted destroy pada lab, tulis alasan dan verifikasi plan sebelum apply.

## 11. Kategori Command

| Kategori | Contoh | Risiko |
|---|---|---|
| Read-only/low risk | `fmt -check`, `validate`, `plan`, `state list/show`, `graph` | Membaca config/state/provider; output dapat sensitif |
| State mutation | `state mv`, `state rm`, `import`, backend migration | Dapat membuat state tidak konsisten atau kehilangan ownership |
| Infrastructure mutation | `apply`, `destroy` | Membuat, mengubah, atau menghapus object provider |

## Acceptance Checklist

- [ ] `tofu fmt -check` dan `tofu validate` berhasil pada project lab atau kegagalan dicatat.
- [ ] Plan dibaca per resource address dan action.
- [ ] Apply hanya dilakukan setelah scope/backend diverifikasi.
- [ ] HTTP dan state diverifikasi setelah perubahan.
- [ ] Destroy menggunakan plan terpisah dan hanya project lab.
- [ ] Tidak ada plan binary, tfvars, state, atau credential masuk Git.

## Troubleshooting

### `Backend initialization required`

Jalankan `tofu init` dari directory root yang benar. Periksa backend block dan jangan memindahkan state manual dengan copy file tanpa prosedur migrasi.

### Plan meminta `-/+ replace`

Baca atribut yang berubah, lifecycle provider, dan dependency. Replacement berarti object lama dihapus/dibuat ulang; review downtime dan data loss sebelum apply.

### Apply gagal karena nama/port sudah ada

Identifikasi owner object. Jangan menghapus object manual yang tidak dibuat lab. Ubah nama/port lab atau lakukan import jika object memang sengaja diadopsi dan ownership disetujui.

### `tofu destroy` melihat lebih banyak resource dari dugaan

Berhenti. Periksa backend, workspace, state list, module directory, dan plan destroy. Jangan meneruskan hanya karena prompt konfirmasi muncul.

## Catatan SRE

Workflow yang benar bukan berarti selalu apply. Keputusan terbaik dapat berupa tidak melakukan perubahan setelah plan menunjukkan blast radius lebih besar dari manfaatnya.

## Kaitan dengan Modul Berikutnya

- [04 — State, Remote, Import, Drift](04-state-remote-import-drift.md) menjelaskan lifecycle state dan operasi adopsi resource.
- [LAB-01 Docker Web Server](lab/LAB-01-tofu-docker-web-server.md) mempraktikkan workflow ini.
- [LAB-02 MinIO, Import & Drift](lab/LAB-02-state-minio-import-drift.md) menambahkan backend remote dan concurrency.
