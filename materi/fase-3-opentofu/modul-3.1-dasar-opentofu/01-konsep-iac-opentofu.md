# 01 — Konsep IaC dan OpenTofu

> IaC bukan sekadar menulis file konfigurasi. IaC adalah cara mengelola perubahan infrastruktur melalui desired state, review, evidence, dan reconciliation yang dapat diulang.

## Tujuan

- Membedakan imperative dan declarative.
- Memahami desired state, current state, state file, dan dependency graph.
- Menjelaskan idempotency serta batas-batasnya.
- Membandingkan OpenTofu dan Terraform secara faktual tanpa menyederhanakan persoalan kompatibilitas.
- Menghubungkan IaC dengan praktik SRE: change review, blast radius, rollback, dan audit trail.

## 1. Mengapa Infrastructure as Code?

Perubahan manual melalui console atau SSH dapat berhasil sekali, tetapi sulit direproduksi dan diaudit. IaC membuat konfigurasi dapat:

- direview melalui pull request;
- diuji dan divalidasi sebelum perubahan;
- direproduksi pada environment lain;
- dikaitkan dengan versi provider dan input;
- dipulihkan melalui riwayat Git dan backup state.

IaC tidak berarti semua hal harus dipaksa dikelola oleh OpenTofu. Database schema, data aplikasi, sertifikat private, dan konfigurasi runtime tertentu mungkin membutuhkan tool khusus. Tentukan boundary ownership sebelum menulis resource.

## 2. Imperative vs Declarative

### Imperative

Operator menjelaskan langkah atau perintah:

```text
1. Buat network.
2. Jalankan container.
3. Pasang label.
4. Buka port.
```

Contoh imperative tidak otomatis aman jika dijalankan ulang. Perintah kedua dapat membuat duplikasi atau gagal karena object sudah ada.

### Declarative

Operator menjelaskan hasil yang diinginkan:

```hcl
resource "docker_container" "web" {
  name  = "sre-tofu-web"
  image = docker_image.web.image_id

  ports {
    internal = 8080
    external = var.host_port
  }
}
```

OpenTofu membangun graph dependency, membaca state, memeriksa provider, lalu menyusun perubahan menuju desired state. Detail urutan tindakan diserahkan pada engine/provider.

## 3. Tiga Kondisi yang Harus Dibedakan

```text
desired configuration  →  apa yang ditulis dalam HCL
current infrastructure  →  apa yang benar-benar ada pada provider
state file              →  identitas/atribut terakhir yang diketahui OpenTofu
```

State bukan sumber kebenaran tunggal untuk kondisi provider. Provider API dapat berubah di luar OpenTofu, sehingga `plan` dan refresh diperlukan untuk menemukan drift. Sebaliknya, perubahan manual tanpa memperbarui HCL dapat kembali dibuat atau dihapus pada plan berikutnya.

## 4. Idempotency dan Reconciliation

Operasi idempotent menghasilkan keadaan akhir yang sama bila dijalankan ulang dengan input dan kondisi yang sama. OpenTofu berusaha melakukan reconciliation, bukan sekadar mengulang command.

Idempotency dapat terganggu oleh:

- tag image mutable seperti `latest`;
- resource provider yang memiliki default berubah;
- timestamp/random value tanpa lifecycle yang jelas;
- API yang eventual consistent;
- perubahan manual di luar OpenTofu;
- state yang stale, rusak, atau dikunci oleh proses lain.

Karena itu, pin provider dan image tag immutable, review plan, gunakan lock, dan simpan evidence versi.

## 5. OpenTofu vs Terraform

OpenTofu adalah fork open source dari Terraform yang dikembangkan di bawah OpenTofu project dan Linux Foundation. Keduanya menggunakan HCL, model provider/resource, state, serta workflow yang mirip. Banyak konfigurasi dapat menjadi titik awal migrasi, tetapi kompatibilitas harus diverifikasi per versi/provider dan tidak boleh diasumsikan tanpa pengujian.

| Aspek | OpenTofu | Terraform |
|---|---|---|
| Proyek | Open source, dikembangkan komunitas di bawah Linux Foundation | Produk HashiCorp dengan komponen open-source dan layanan komersial |
| Lisensi | Mozilla Public License 2.0 (MPL-2.0) | HashiCorp Business Source License (BUSL) untuk versi yang tercakup; detail versi tetap perlu dibaca |
| Komunitas | Ekosistem OpenTofu, kontribusi publik, kompatibilitas dengan banyak provider | Ekosistem HashiCorp Terraform dan Terraform Registry yang sangat luas |
| Operasional | CLI dan konsep state/provider yang familiar bagi pengguna Terraform | CLI, Terraform Registry, dan layanan HCP Terraform/Enterprise sesuai pilihan organisasi |
| Risiko migrasi | Periksa fitur, versi, provider, backend, dan policy yang digunakan | Periksa ketentuan lisensi dan model layanan organisasi |

Lisensi bukan satu-satunya kriteria. Nilai juga governance, dukungan provider, security advisory, skill tim, policy organisasi, dan kebutuhan enterprise. Jangan menyalin binary, provider, atau module tanpa memeriksa lisensi dan provenance.

## 6. Provider sebagai Boundary API

Provider menerjemahkan resource HCL menjadi operasi API Docker, Proxmox, vSphere, libvirt, Kubernetes, atau service internal. Provider version harus dipin di `required_providers` dan dikunci melalui `.terraform.lock.hcl`/lock file yang direview.

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
```

`required_version` di atas hanya contoh kompatibilitas dan harus disesuaikan dengan binary yang telah diuji. Jangan menyalin provider dari sumber tidak terpercaya.

## 7. IaC dalam Siklus SRE

```text
request/change ticket
  → HCL change
  → fmt/validate
  → plan + peer review
  → approved apply
  → health/evidence
  → observe/drift detection
  → incident learning/runbook
```

Plan adalah artefak review, bukan approval otomatis. Apply yang sukses juga bukan bukti service sehat; lakukan validasi aplikasi dan observability.

## Acceptance Checklist

- [ ] Bisa menjelaskan tiga perbedaan imperative dan declarative dengan contoh.
- [ ] Bisa menggambar hubungan HCL → plan → provider API → state.
- [ ] Bisa menyebut minimal tiga penyebab idempotency gagal.
- [ ] Bisa menjelaskan mengapa OpenTofu/Terraform tidak otomatis mengelola data aplikasi.
- [ ] Bisa menunjukkan lokasi untuk memeriksa provider version tanpa mengungkap credential.

## Troubleshooting

### `tofu plan` ingin mengganti resource yang tampak tidak berubah

Periksa provider version, lock file, default provider, perubahan manual, dan atribut yang memang `ForceNew`. Jangan memakai `-replace` atau mengedit state untuk menghilangkan diff tanpa memahami penyebab.

### Provider tidak dapat diinisialisasi

Periksa `tofu version`, `required_providers`, lock file, network/registry mirror, arsitektur `darwin_arm64`, dan pesan error lengkap. Jangan mengunduh binary acak atau memasukkan credential ke command line.

### Plan kosong padahal object provider berubah

Pastikan perubahan berada dalam boundary resource yang dikelola, state mengarah ke backend yang benar, dan refresh provider dapat membaca object tersebut. Plan kosong bukan bukti semua sistem eksternal identik.

## Catatan SRE

Keberhasilan IaC diukur dari perubahan yang dapat dipahami dan dibatasi, bukan jumlah resource yang berhasil dibuat. Change kecil dengan evidence kuat lebih baik daripada apply besar yang tidak dapat dijelaskan.

## Kaitan dengan Modul Berikutnya

- Sintaks HCL dan dependency graph dibedah di [02 — HCL, Resource, Variable, Output, Provider](02-hcl-resource-variable-output-provider.md).
- Workflow eksekusi dan approval dibahas di [03 — Workflow OpenTofu](03-workflow-init-plan-apply-destroy.md).
- State, import, dan drift dibahas di [04 — State, Remote, Import, Drift](04-state-remote-import-drift.md).
