# 01 — Provider On-Prem dan Boundary

> **Tujuan:** memahami provider sebagai adapter API, memilih boundary ownership, dan menghindari anggapan bahwa semua resource on-prem memiliki lifecycle yang sama.

## Tujuan Belajar

- membedakan provider, resource, data source, CloudInit, dan Ansible;
- membandingkan Proxmox, vSphere, libvirt, serta HTTP/REST secara konseptual;
- menyusun provider version dan identity boundary;
- mendokumentasikan ownership dan failure mode sebelum membuat resource.

## 1. Provider Bukan Infrastruktur

Provider menerjemahkan deklarasi OpenTofu ke API atau library tertentu. Provider bukan hypervisor, bukan backend state, dan bukan jaminan bahwa endpoint aman. Sebelum dipakai, verifikasi:

```text
provider source dan versi
provider/OpenTofu compatibility
endpoint dan TLS/CA policy
identity, permission, dan scope
resource import/update/delete behavior
retry, timeout, locking, dan failure semantics
ARM64/provider binary availability
```

`required_providers` harus dikunci sesuai policy organisasi. Jangan menyalin source atau versi dari contoh tanpa menguji checksum, compatibility, dan behavior provider.

## 2. Pilihan Provider On-Prem

| Pilihan | Cocok untuk | Hal yang harus diverifikasi | Batas contoh |
|---|---|---|---|
| Proxmox provider | VM, template, storage, network pada Proxmox | provider source/version, API endpoint, node/storage mapping, clone semantics, permission | provider dan endpoint tidak tersedia otomatis di Mac |
| vSphere provider | VM/template/network pada vCenter/ESXi | vCenter version, datastore/portgroup, guest customization, permission, provider compatibility | bukan pengganti API vSphere atau capacity planning |
| libvirt provider | VM/QEMU/KVM dan network/storage libvirt | libvirt daemon, qemu architecture, image format, host capability, provider behavior | Apple Silicon/OrbStack tidak otomatis menyediakan KVM/libvirt |
| CloudInit | first-boot seed untuk hostname/user/package minimum | image support, datasource, ordering, idempotency, secret exposure, console evidence | CloudInit bukan provider dan bukan configuration management penuh |
| HTTP/REST | API internal yang tidak memiliki provider khusus | auth chain, TLS, schema, idempotency, retry, pagination, delete semantics, audit | HTTP read success tidak membuktikan resource lifecycle |

Nama provider dan capability harus dipastikan dari dokumentasi versi yang benar-benar digunakan. OpenTofu compatibility tidak boleh diasumsikan hanya karena contoh Terraform terlihat serupa.

## 3. Root Module Mengendalikan Provider

Provider configuration sebaiknya berada di root module/environment. Child module menerima provider melalui caller sehingga endpoint dan identity tidak dipilih diam-diam oleh module.

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    onprem = {
      # Illustrative only: pilih source dan versi setelah compatibility review.
      source  = "example.invalid/onprem/provider"
      version = "~> 1.0"
    }
  }
}

provider "onprem" {
  endpoint = var.api_endpoint
  # Identity berasal dari credential chain yang disetujui; bukan literal di HCL.
}

module "host" {
  source = "../../modules/onprem-host"

  providers = {
    onprem = onprem
  }

  name        = var.host_name
  environment = var.environment
  image       = var.image_reference
}
```

`example.invalid` menandakan placeholder dan bukan provider yang dapat di-install. Jangan menjalankan snippet tersebut sebagai bukti provider nyata.

Provider alias dipakai jika satu root module perlu dua context yang memang disetujui, misalnya management dan workload:

```hcl
provider "onprem" {
  alias    = "management"
  endpoint = var.management_endpoint
}

module "inventory" {
  source = "../../modules/inventory"

  providers = {
    onprem = onprem.management
  }

  environment = var.environment
}
```

Alias bukan security boundary otomatis. Endpoint, identity, permission, backend key, dan plan target tetap harus diperiksa.

## 4. Resource, Data Source, dan Ownership

Resource berarti module memiliki lifecycle object tersebut. Data source membaca object yang dimiliki boundary lain. Contoh konseptual:

```hcl
data "onprem_network" "workload" {
  name = var.shared_network_name
}

resource "onprem_vm" "this" {
  name       = var.name
  network_id = data.onprem_network.workload.id
  image      = var.image_reference
}
```

Sebelum memakai data source, dokumentasikan owner network, identifier stability, permission read, availability, dan failure mode bila network hilang. Jangan memakai data source untuk menyamarkan ownership yang seharusnya berada pada project.

## 5. CloudInit sebagai Guest Initialization

CloudInit sering dipakai untuk first boot:

```yaml
# Contoh non-secret; jangan menaruh password, token, private key, atau kubeconfig.
#cloud-config
hostname: sre-node-placeholder
package_update: false
packages:
  - curl
  - ca-certificates
runcmd:
  - [ sh, -c, "printf '%s\\n' 'cloud-init marker' > /var/lib/sre-bootstrap-marker" ]
```

Sebelum dipakai, verifikasi apakah image mendukung CloudInit, datasource yang digunakan, urutan network, console/log exposure, dan perilaku saat instance reboot. User-data dapat masuk metadata, state, log, atau backup; perlakukan sebagai data sensitif bila berisi credential.

## 6. Boundary dan Failure Mode

| Boundary | Owner | Contoh failure | Stop condition |
|---|---|---|---|
| API provider | platform/IaC | timeout, permission, schema mismatch | jangan retry mutasi tanpa mengetahui status remote |
| VM/network/storage | virtualization team | resource penuh, IP collision, datastore unavailable | hentikan apply dan cek object/provider state |
| guest init | OS/platform | CloudInit gagal atau image tidak cocok | jangan lanjut bootstrap sebelum console/readiness check |
| configuration | Ansible owner | package/service/hardening gagal | batasi host, kumpulkan check mode/evidence |
| k3s | cluster owner | token/version/network/quorum issue | ikuti runbook k3s, jangan reset cluster aktif |

### Provider `apply` timeout

Jangan langsung menjalankan apply ulang. Periksa task/job pada hypervisor, object inventory, provider log yang telah diredáksi, dan state. Jika object sudah dibuat tetapi state belum tersimpan, lakukan recovery/import hanya dengan prosedur terverifikasi, backup, lock, dan approval.

### Provider menghapus resource saat input berubah

Baca `plan` dan `replace` action. Pastikan perubahan template, disk, network, atau name memang diizinkan. Jangan memakai `-target` sebagai cara permanen untuk menyembunyikan dependency atau replacement.

### Endpoint atau identity salah

Stop sebelum mutation. Periksa directory, backend key, provider alias, endpoint, identity context, dan environment allowlist. Jangan mencetak credential saat debugging.

## Acceptance Checklist

- [ ] Provider dipilih berdasarkan API, version, identity, resource behavior, dan failure mode.
- [ ] CloudInit dijelaskan sebagai guest initialization, bukan provider atau pengganti Ansible.
- [ ] Root module mengendalikan provider; child module tidak memilih endpoint production secara tersembunyi.
- [ ] Resource ownership dan data source boundary terdokumentasi.
- [ ] Semua contoh credential-free dan placeholder diberi label.
- [ ] Provider/runtime yang belum diuji ditandai belum diverifikasi.

## Catatan SRE

Provider adalah komponen control plane. Bug atau ketidakcocokan provider dapat membuat plan terlihat valid tetapi gagal pada API, atau lebih buruk: mengubah object yang salah. Pin versi, review plan, batasi identity, dan uji failure semantics pada resource disposable sebelum memperluas scope.

## Kaitan dengan Modul Berikutnya

- [02 — Simulasi Lokal](02-simulasi-lokal-docker-libvirt-mock.md) membatasi apa yang dapat dibuktikan Docker/libvirt/mock.
- [03 — OpenTofu, Ansible, dan k3s](03-opentofu-ansible-k3s-handoff.md) meneruskan metadata non-secret ke boundary berikutnya.
