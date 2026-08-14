# 02 — Simulasi Lokal: Docker, libvirt, dan Mock

> **Tujuan:** membuat latihan yang cepat dan disposable pada Mac ARM64/OrbStack sambil menjaga batas antara simulasi container, VM libvirt, dan provider on-prem production.

## Tujuan Belajar

- memilih simulator sesuai pertanyaan yang ingin diuji;
- memahami perbedaan container, VM, dan mock contract;
- menyusun predicted plan dan metadata host tanpa credential;
- berpindah ke jalur statis bila runtime tidak tersedia.

## 1. Apa yang Disimulasikan?

| Pertanyaan | Docker | libvirt | Mock/static |
|---|---:|---:|---:|
| module composition dan variable contract | ✅ | ✅ | ✅ |
| labels, port, dependency, output | ✅ | ✅ | ✅ |
| guest kernel/systemd/SSH | ❌ | ✅ bila VM benar-benar berjalan | ❌ |
| CloudInit datasource dan first boot | ❌ | mungkin, perlu runtime | desain saja |
| hypervisor API, datastore, quorum | ❌ | ❌ | ❌ |
| plan address dan ownership | ✅ | ✅ | ✅ |
| network L2 production/MetalLB | ❌ | tidak otomatis | ❌ |

Container Docker dapat menjadi objek yang mewakili service atau host metadata, tetapi bukan bukti bahwa Ansible SSH, CloudInit, systemd, disk attach, atau k3s VM berhasil.

## 2. Preflight ARM64 dan OrbStack

```bash
set -eu

pwd
tofu version
docker context show
docker info --format '{{.OSType}}/{{.Architecture}}'
```

Jika salah satu binary/runtime tidak tersedia, jangan mengarang hasil. Simpan output preflight hanya jika tidak mengandung credential atau endpoint sensitif. Pada Apple Silicon, verifikasi image multi-arch dan jangan menganggap image `amd64` dapat berjalan tanpa emulation/cost.

## 3. Simulasi Docker yang Credential-Free

Root module dari Modul 3.2 dapat memakai child module reusable. Berikut bentuk minimal ilustratif:

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

provider "docker" {}

variable "instances" {
  type = map(object({
    host_port = number
    image     = string
    role      = string
  }))

  validation {
    condition     = alltrue([for item in values(var.instances) : item.host_port >= 1024 && item.host_port <= 65535])
    error_message = "Setiap port harus berada pada rentang non-privileged 1024-65535."
  }
}

resource "docker_image" "web" {
  for_each     = var.instances
  name         = each.value.image
  keep_locally = true
}

resource "docker_container" "web" {
  for_each = var.instances
  name     = "sre-sim-${each.key}"
  image    = docker_image.web[each.key].image_id

  labels = {
    "sre.owner" = "modul-3.3-lab-01"
    "sre.role"  = each.value.role
  }


  ports {
    internal = 80
    external = each.value.host_port
  }
}

output "host_metadata" {
  value = {
    for key, container in docker_container.web : key => {
      name        = container.name
      role        = var.instances[key].role
      environment = "lab"
      endpoint    = "http://127.0.0.1:${var.instances[key].host_port}"
    }
  }
}
```

Output tersebut adalah metadata simulasi. `endpoint` lokal bukan IP server on-prem dan tidak boleh diteruskan sebagai production inventory tanpa mapping yang disetujui.

## 4. Libvirt dan Mock

Libvirt dapat lebih dekat ke VM karena melibatkan image, network, disk, dan guest OS, tetapi hanya bila daemon, QEMU/KVM, architecture, provider, dan image tersedia. Pada Mac Apple Silicon/OrbStack, libvirt/KVM tidak otomatis tersedia. Jangan menulis “VM berhasil” hanya karena HCL berhasil diparse.

Mock provider atau mock API berguna untuk menguji contract dan failure response:

```hcl
variable "host_contract" {
  type = object({
    name        = string
    role        = string
    environment = string
  })
}

output "predicted_host" {
  value = {
    name        = var.host_contract.name
    role        = var.host_contract.role
    environment = var.host_contract.environment
    status      = "predicted-only"
  }
}
```

Mock tidak membuktikan provider schema, hypervisor behavior, network reachability, atau Ansible connectivity.

## 5. Predicted Plan

Sebelum runtime, buat tabel address dan ownership:

| Address | Predicted action | Owner | Evidence |
|---|---|---|---|
| `docker_image.web["public"]` | create/read | LAB-01 module | plan atau HCL review |
| `docker_container.web["public"]` | create | LAB-01 module | plan + container check |
| `module.host["worker-1"]` | predicted create | environment root | plan address |
| `data.onprem_network.workload` | read | network boundary lain | provider read evidence |

Key map harus stabil. Rename `worker-1` menjadi `worker-a` adalah address change; review replacement sebelum mutation.

## 6. Jalur Runtime Disposable

Jika `tofu` dan Docker-compatible runtime tersedia:

```bash
set -eu

pwd
tofu fmt -check
tofu init
tofu validate
tofu plan -out=lab-03-03.tfplan
tofu show -no-color lab-03-03.tfplan
```

Apply hanya setelah scope disposable, backend, provider context, labels, port, plan, dan approval diverifikasi:

```bash
tofu apply lab-03-03.tfplan
tofu output
# health check hanya pada endpoint lab yang diketahui
```

Cleanup harus memakai destroy plan terpisah dan scope yang sama. Jangan memasukkan plan binary atau state ke Git.

## 7. Jalur Statis

Jika runtime tidak tersedia:

1. review `required_providers`, variable type, validation, address, dan output;
2. buat predicted plan tanpa menulis raw plan artifact;
3. pastikan output tidak memuat credential;
4. dokumentasikan perbedaan container versus VM;
5. tandai `tofu init/validate/plan/apply`, Docker, libvirt, mock API, dan HTTP check sebagai **belum diverifikasi**;
6. lanjutkan ke desain handoff dengan metadata contoh non-secret.

## Troubleshooting

### Image tidak mendukung ARM64

Periksa manifest image dan architecture runtime. Pilih tag multi-arch yang disetujui atau gunakan emulation dengan catatan performa. Jangan mengganti tag menjadi `latest` untuk menghilangkan error reproduksi.

### Docker output terlihat seperti host IP

Bedakan port localhost, container address, VM address, dan IP management. Output Docker tidak otomatis menjadi inventory Ansible.

### Libvirt provider gagal di-init

Periksa daemon, socket, QEMU architecture, provider version, image format, dan permission tanpa mencetak credential. Lanjutkan static lane bila host tidak mendukung.

### Mock plan terlalu optimistis

Tambahkan failure cases: timeout, duplicate name, IP unavailable, quota, permission denied, and stale object. Mock hanya menguji contract, bukan semantics provider nyata.

## Acceptance Checklist

- [ ] Pertanyaan simulasi dan batas buktinya tertulis.
- [ ] HCL menggunakan value non-secret dan image/port yang dapat direview.
- [ ] Predicted address, owner, dan action terdokumentasi.
- [ ] Container tidak disebut sebagai VM production.
- [ ] ARM64/OrbStack dan libvirt limitation dijelaskan.
- [ ] Runtime yang tidak tersedia ditandai belum diverifikasi.

## Catatan SRE

Simulator mempercepat feedback, bukan menghapus perbedaan failure domain. Uji contract di lokal, lalu validasi behavior provider, hypervisor, network, guest OS, dan Ansible pada environment yang merepresentasikan production sebelum mengubah deployment model.

## Kaitan dengan Modul Berikutnya

- [LAB-01 — Simulasi Provisioning Lokal](lab/LAB-01-simulasi-provisioning-lokal.md) mempraktikkan predicted plan dan metadata.
- [03 — OpenTofu, Ansible, dan k3s](03-opentofu-ansible-k3s-handoff.md) menentukan batas setelah metadata dibuat.
