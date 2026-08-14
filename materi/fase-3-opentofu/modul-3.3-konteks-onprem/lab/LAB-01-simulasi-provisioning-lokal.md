# LAB-01 — Simulasi Provisioning Lokal

> **Mode utama:** static lane. **Mode opsional:** runtime disposable bila `tofu` dan Docker-compatible runtime tersedia serta preflight lulus.

## Tujuan

- membuat simulasi host/service dengan `for_each` dan key stabil;
- meninjau provider, state address, labels, port, dan metadata output;
- membedakan evidence HCL/plan dari bukti VM, SSH, CloudInit, dan k3s.

## Scope dan Guardrail

Scope hanya direktori lab disposable. Jangan memakai endpoint Proxmox/vSphere, backend production, state production, atau credential nyata. **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Jangan menjalankan `-auto-approve` sebagai default dan jangan menyimpan state/plan binary di Git.

## Prasyarat

- Modul 3.1 dan 3.2 telah dibaca.
- `tofu`, Docker/OrbStack, dan provider Docker hanya dipakai jika benar-benar tersedia.
- Bila runtime tidak tersedia, ikuti seluruh static lane dan tandai runtime sebagai **belum diverifikasi**.

## Topologi Simulasi

```text
Mac ARM64
  → OrbStack/Docker-compatible runtime (opsional)
  → container disposable: public, admin
  → output metadata non-secret
  → predicted Ansible inventory (bukan inventory production)
```

## Bagian A — Static Lane (Wajib)

### 1. Buat layout review

```text
lab-01/
├── main.tf
├── variables.tf
├── outputs.tf
└── README-review.md
```

Gunakan pola HCL dari [02 — Simulasi Lokal](../02-simulasi-lokal-docker-libvirt-mock.md). Jika menyalin, ubah nama lab/label dan gunakan image yang disetujui. Jangan memasukkan access key, password, private key, token, kubeconfig, atau endpoint credential.

### 2. Prediksi address

Dengan input map berikut:

```hcl
instances = {
  public = {
    host_port = 18080
    image     = "<approved-multi-arch-image>"
    role      = "utility"
  }
  admin = {
    host_port = 18081
    image     = "<approved-multi-arch-image>"
    role      = "utility"
  }
}
```

Predicted address:

```text
docker_image.web["public"]
docker_image.web["admin"]
docker_container.web["public"]
docker_container.web["admin"]
```

Tulis action (`create`, `read`, atau `no-op`), owner, dan alasan perubahan key pada tabel review. Placeholder image bukan bukti image tersedia.

### 3. Tinjau contract output

Output minimum yang boleh diteruskan:

```text
name
role
environment
endpoint simulasi
provisioning reference bila ada
```

Tolak output yang memuat token, password, private key, kubeconfig, credential provider, atau raw state.

### 4. Static acceptance

- [ ] HCL/resource/module shape terbaca.
- [ ] `for_each` memakai key stabil.
- [ ] Port berada di rentang non-privileged dan tidak collision.
- [ ] Label ownership dan environment ada.
- [ ] Output non-secret.
- [ ] Batas container versus VM tertulis.
- [ ] Runtime belum diverifikasi bila command tidak dijalankan.

## Bagian B — Runtime Disposable (Opsional)

Jalankan hanya pada project yang scope-nya telah diverifikasi:

```bash
set -eu

pwd
tofu version
docker context show
docker info --format '{{.OSType}}/{{.Architecture}}'
tofu fmt -check
tofu init
tofu validate
tofu plan -out=lab-01.tfplan
tofu show -no-color lab-01.tfplan
```

Review plan sebelum mutation. Jika disetujui untuk lab disposable, apply menggunakan plan yang sama:

```bash
tofu apply lab-01.tfplan
tofu output
```

Lakukan health check hanya pada port lab yang diketahui. Jangan menyebut hasil ini sebagai bukti VM boot, SSH, CloudInit, systemd, storage, hypervisor, atau k3s.

Cleanup memerlukan scope dan approval yang sama. Buat plan cleanup, review action, dan hapus hanya object berlabel lab ini. Jangan menghapus object di luar scope.

## Evidence yang Dikumpulkan

Simpan summary yang sudah diredáksi:

```text
commit/reference materi
provider source/version dan lock status
backend/context class (tanpa credential)
plan action count dan address
labels/scope
output metadata non-secret
runtime status: executed atau belum diverifikasi
```

Jangan menyimpan `terraform.tfstate`, raw plan, provider credential, atau shell history yang berisi secret.

## Troubleshooting

### `tofu` atau Docker tidak ditemukan

Pindah ke static lane. Catat command yang tidak dapat dijalankan dan jangan mengganti dengan klaim berhasil.

### Provider Docker gagal init

Periksa Docker context, daemon/runtime, architecture, provider version, dan scope. Jangan mengarahkan provider ke endpoint production.

### Port sudah dipakai

Pilih port lab non-privileged yang disetujui dan update plan. Jangan menghentikan service yang tidak dimiliki lab.

### Key map berubah

Review address churn dan replacement sebelum apply. Gunakan key stabil; jangan memakai index list bila identitas instance bermakna.

## Pertanyaan Refleksi

1. Evidence apa yang membuktikan module contract tetapi tidak membuktikan VM?
2. Mengapa `127.0.0.1:18080` tidak boleh langsung menjadi `ansible_host` production?
3. Apa yang harus dilakukan bila apply timeout tetapi container ternyata sudah dibuat?
4. Metadata apa yang harus ditolak karena termasuk credential?

## Kaitan Berikutnya

Gunakan output non-secret untuk [LAB-02 — Handoff ke Ansible dan k3s](LAB-02-handoff-ke-ansible-dan-k3s.md). Handoff nyata dan playbook idempotent tetap menjadi materi Fase 4 — Ansible yang masih menyusul.

## Catatan SRE

Lab ini menguji bentuk perubahan dan contract, bukan representasi penuh failure domain on-prem. Nilai lab berasal dari review scope, plan, address, output, dan evidence—bukan dari banyaknya object yang dibuat.
