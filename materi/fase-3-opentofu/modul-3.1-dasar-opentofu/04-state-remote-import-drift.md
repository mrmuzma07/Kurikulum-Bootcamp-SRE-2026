# 04 — State, Remote Backend, Import, dan Drift

> **Tujuan:** memahami state sebagai metadata ownership dan mapping resource, memilih local/remote state, memakai locking, dan memulihkan perbedaan antara konfigurasi, state, dan infrastructure dengan prosedur yang dapat diaudit.

## 1. Apa Itu State?

State menyimpan mapping address HCL ke object provider serta atribut yang dibutuhkan OpenTofu untuk menghitung perubahan. State bukan sekadar cache dan bukan backup aplikasi.

State dapat memuat:

- resource ID dan metadata provider;
- atribut yang digunakan pada output/dependency;
- data dari data source;
- value yang ditandai sensitive.

Karena itu, **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Namun, walaupun configuration tidak memuat secret, provider atau output dapat menyimpan value sensitif di state. Lindungi state dengan access control, encryption at rest/in transit, least privilege, retention, dan audit.

## 2. Local State vs Remote State

### Local state

Default backend menyimpan `terraform.tfstate` di directory project. Kelebihan: mudah untuk lab satu operator. Kekurangan:

- sulit berbagi dan locking;
- rawan hilang jika disk rusak;
- mudah ikut tercopy ke backup/log yang tidak aman;
- tidak cocok untuk concurrent team workflow.

### Remote state

Remote backend menyimpan state pada service terpusat. Kelebihan:

- satu lokasi dengan permission dan audit;
- locking/concurrency control;
- backup dan retention terpusat;
- cocok untuk pipeline dan tim.

Kekurangan: membutuhkan availability backend, credential management, network, migration procedure, dan pemahaman failure mode. Remote state bukan otomatis aman hanya karena berada di object storage.

## 3. S3-Compatible dan MinIO di OrbStack

MinIO dapat dipakai sebagai S3-compatible object storage di lab. Jalankan hanya pada project/network lab disposable dan dokumentasikan image tag yang disetujui. Credential MinIO wajib diberikan melalui mekanisme environment/credential helper dan tidak boleh ditulis pada HCL atau Git.

Contoh backend menggunakan environment-driven credentials:

```hcl
terraform {
  backend "s3" {
    bucket                      = "sre-tofu-state"
    key                         = "modul-3.1/lab-01/terraform.tfstate"
    region                      = "us-east-1"
    endpoint                    = "http://127.0.0.1:9000"
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_requesting_account_id  = true
    use_path_style              = true
  }
}
```

Nilai `endpoint`, bucket, dan key di atas adalah contoh lokal; sesuaikan dengan endpoint lab yang benar. Jangan memasukkan access key/secret key literal. Backend S3 biasanya membaca `AWS_ACCESS_KEY_ID` dan `AWS_SECRET_ACCESS_KEY` atau credential helper sesuai versi/provider/backend. Gunakan cara yang didokumentasikan versi OpenTofu/MinIO yang dipakai.

Pada backend S3-compatible, fitur locking perlu diverifikasi terhadap versi OpenTofu dan backend. Mekanisme legacy DynamoDB locking dan fitur locking native/alternatif tidak boleh dicampur secara membabi buta. Baca `tofu init`/backend documentation versi yang digunakan sebelum mengaktifkan lock.

## 4. Locking dan Concurrency

Tanpa lock, dua apply dapat membaca state yang sama lalu menimpa perubahan satu sama lain. Dampaknya dapat berupa:

- lost update;
- state berisi mapping stale;
- resource orphan atau duplicate;
- apply kedua gagal setelah sebagian perubahan;
- operator tidak tahu plan yang telah disetujui sudah tidak valid.

Aturan lab:

```text
satu state + satu lock = satu perubahan writer pada satu waktu
```

Jangan menjalankan concurrent `tofu apply`, `import`, `state mv`, atau backend migration pada key yang sama. Jika lock timeout, cari proses yang benar-benar berjalan dan ikuti prosedur unlock resmi versi OpenTofu. Jangan menghapus lock object manual hanya untuk membuka jalan.

## 5. Backup dan Permission

Sebelum migrasi backend atau operasi state mutation:

1. hentikan writer lain;
2. catat backend, key, workspace, dan revision;
3. lakukan backup sesuai mekanisme backend;
4. batasi permission backup;
5. simpan checksum/metadata tanpa mencetak credential;
6. uji prosedur restore di environment disposable;
7. siapkan stop condition dan reviewer.

Backup state bukan backup PV/database/image. Backup terpisah diperlukan untuk data aplikasi dan secret store.

## 6. Inspeksi State

Command berikut relatif aman tetapi tetap dapat mencetak data sensitif:

```bash
tofu state list
tofu state show docker_container.web
tofu output -json
tofu show
```

`tofu state show` dipakai dengan resource address yang diketahui. Redact ID/endpoint sensitif saat menyimpan evidence. Jangan mengirim file state ke chat atau commit.

Untuk melihat format plan tanpa apply:

```bash
tofu plan -out=review.tfplan
tofu show -no-color review.tfplan
```

Simpan plan di lokasi yang di-ignore dan hapus sesuai retention policy.

## 7. `tofu import`

Import mengadopsi object provider yang sudah ada ke address resource dalam state. Import tidak otomatis menulis konfigurasi lengkap yang benar; HCL harus dibuat dan disesuaikan.

Pola umum:

```bash
tofu import docker_container.manual <resource-id-yang-sudah-diverifikasi>
tofu state show docker_container.manual
tofu plan
```

Guardrail import:

- pastikan object dibuat manual oleh tim dan ownership disetujui;
- verifikasi backend/workspace/project;
- gunakan ID yang tidak mengandung credential;
- buat backup state sebelum import bila policy mewajibkan;
- tulis resource block minimal sebelum atau segera setelah import;
- plan harus menunjukkan no-op atau perubahan yang memang disengaja;
- jangan import object production ke state lab.

Syntax ID provider berbeda-beda. Jangan menyalin `<resource-id-yang-sudah-diverifikasi>` tanpa menggantinya dengan ID lab yang benar.

## 8. `tofu state mv`

`state mv` mengubah address ownership tanpa memindahkan object provider:

```bash
tofu state mv docker_container.web module.web.docker_container.this
tofu plan
```

Gunakan untuk refactor address yang telah direncanakan. Backup state, lock state, pastikan source/destination benar, lalu review plan. Jangan menggunakan `state mv` untuk menyembunyikan drift atau membuat plan terlihat kosong.

Versi OpenTofu dapat memiliki flag tambahan untuk state backend. Ikuti `tofu state mv -help` dari versi yang dipakai.

## 9. `tofu state rm`

`state rm` menghapus mapping dari state tetapi biasanya tidak menghapus object provider. Pada plan berikutnya object dapat dianggap unmanaged dan berisiko dibuat ulang/tertinggal.

```bash
tofu state rm docker_container.orphan
```

Ini adalah operasi berisiko. Sebelum menjalankan:

```text
[ ] object dan address terverifikasi
[ ] alasan ownership ditulis
[ ] state backup selesai
[ ] lock aktif / tidak ada writer lain
[ ] dampak orphan dan cleanup dipahami
[ ] reviewer menyetujui
```

Jangan memakai `state rm` sebagai cara cepat memperbaiki plan yang tidak dipahami.

## 10. Drift Detection

Drift terjadi ketika current infrastructure berubah di luar desired configuration/state workflow, misalnya port, label, image, ukuran disk, atau object dihapus manual.

Flow:

```text
1. Verifikasi backend/workspace dan scope.
2. Jalankan tofu plan read-only.
3. Kelompokkan perbedaan: manual change, provider default, version change, atau state issue.
4. Tentukan source of truth: terima perubahan ke HCL atau remediasi ke desired state.
5. Review plan baru.
6. Apply hanya setelah approval.
7. Validasi provider dan service.
8. Catat root cause dan pencegahan.
```

### Remediasi menerima perubahan manual

Jika perubahan manual valid dan harus dipertahankan, ubah HCL sesuai keadaan yang diinginkan, pin version, lalu review plan. Jangan mengedit state untuk mencerminkan perubahan tanpa HCL.

### Remediasi mengembalikan desired state

Jika perubahan manual tidak disetujui, plan dapat mengembalikan atribut ke HCL. Review downtime, replacement, data loss, dan owner perubahan sebelum apply. Production remediation perlu change window.

## 11. State Migration

Migrasi local → remote atau perubahan key/backend memerlukan:

```text
backup → lock/stop writer → verify destination → init migration prompt → inspect state → plan → approval
```

Jangan menjalankan migrasi backend pada directory yang salah. Jangan menghapus local backup sampai remote state tervalidasi dan retention policy terpenuhi.

## Acceptance Checklist

- [ ] Bisa menjelaskan mengapa remote state bukan otomatis secret manager.
- [ ] Bisa menjelaskan lost update akibat concurrent apply.
- [ ] Bisa melakukan `state list/show` tanpa menyalin secret ke evidence.
- [ ] Bisa menjelaskan perbedaan import, state mv, dan state rm.
- [ ] Bisa membedakan drift yang diterima dan drift yang harus diremediasi.
- [ ] Bisa menulis runbook backup/migration dengan stop condition.

## Troubleshooting

### `Error acquiring the state lock`

Jangan langsung force unlock. Pastikan tidak ada apply/import aktif, identifikasi lock owner sesuai backend, dan ikuti prosedur resmi. Force unlock hanya dengan lock ID yang diverifikasi dan approval.

### Import berhasil tetapi plan membuat banyak perubahan

Konfigurasi belum merepresentasikan object aktual, default provider berubah, atau import ID tidak tepat. Bandingkan `state show`, dokumentasi provider, dan HCL. Jangan menghapus state hanya untuk meniadakan diff.

### Remote state tidak ditemukan

Periksa endpoint, bucket/key, region/path style, credential helper, network, dan permission. Jangan mencetak environment variable credential saat debugging.

### State lokal dan remote berbeda

Hentikan writer, identifikasi backend aktif dari `tofu init` dan directory, lalu jangan menggabungkan file state manual. Gunakan prosedur migration/backup yang terdokumentasi.

## Catatan SRE

State adalah inventory ownership. Semua operasi state adalah perubahan control plane dan harus diperlakukan seperti perubahan database: ada backup, lock, audit, rollback decision, dan validasi.

## Kaitan dengan Modul Berikutnya

- [LAB-02 State MinIO, Import & Drift](lab/LAB-02-state-minio-import-drift.md) memberi walkthrough disposable.
- Modul 3.2 membahas environment isolation, module composition, dan secret integration yang lebih production-ready.
- Modul 3.3 membahas provider on-prem dan handoff OpenTofu → Ansible → k3s.
