# 03 — OpenTofu, Ansible, dan k3s Handoff

> **Tujuan:** merancang handoff yang terurut dari provisioning host sampai cluster k3s, dengan ownership, readiness, secret boundary, dan rollback yang jelas.

## Tujuan Belajar

- memisahkan provisioning, configuration management, dan cluster lifecycle;
- menyusun metadata output non-secret untuk inventory;
- menentukan readiness gate sebelum Ansible dan k3s;
- menjelaskan sequencing, idempotency, failure handling, dan rollback.

## 1. Tiga Boundary Berbeda

```text
OpenTofu/provider
  = object existence dan infrastructure relationship

Ansible
  = OS bootstrap, hardening, package, service, configuration

k3s
  = Kubernetes distribution, server/agent join, cluster operation
```

OpenTofu tidak seharusnya mengedit file daemon, memasang seluruh package OS, atau mencetak kubeconfig. Ansible tidak seharusnya mengambil alih state ownership provider. k3s tidak seharusnya dipasang sebelum host, network, time sync, firewall, dan storage siap.

## 2. Contract Metadata

Output module/root hanya membawa metadata minimum yang diperlukan:

```hcl
output "ansible_inventory_metadata" {
  description = "Metadata non-secret untuk tahap configuration management."

  value = {
    for key, host in module.host : key => {
      hostname         = host.hostname
      address          = host.ip_address
      role             = host.role
      environment      = var.environment
      module_version   = var.module_version
      provisioning_ref = host.id
    }
  }
}
```

Contoh di atas bersifat pola contract. Nama attribute harus disesuaikan dengan provider/module yang benar-benar digunakan. Jangan memasukkan password SSH, private key, token bootstrap, kubeconfig, join token, atau secret provider ke output biasa maupun artifact.

Metadata handoff harus memiliki:

```text
stable host key
hostname/FQDN yang disetujui
management address
role: server/agent/utility
environment
module/provider version
provisioning reference
ownership dan expiry/lifecycle
```

## 3. Readiness Gate Host

Ansible hanya dipanggil setelah pemeriksaan berikut lulus:

| Area | Bukti minimum | Stop condition |
|---|---|---|
| Identity | hostname, role, environment sesuai inventory | host tertukar atau duplicate key |
| Network | management path, node-to-node path, DNS/route | timeout atau IP collision |
| Time | NTP/chrony status dan timezone policy | clock skew mengganggu TLS/etcd |
| Access | SSH/check mode dengan identity dari secret mechanism | credential tidak tersedia atau host unknown |
| OS | distro/version/kernel/resource capacity | image tidak didukung runbook |
| Storage | path, filesystem, capacity, durability | disk penuh atau mount salah |
| Firewall | port matrix disetujui | port k3s tidak dapat berkomunikasi |
| Security | user, patch baseline, audit policy | host belum memenuhi hardening gate |

Readiness adalah gate, bukan komentar. Jika satu check gagal, jangan lanjutkan install k3s untuk node tersebut.

## 4. Sequencing Produksi

```text
1. checkout commit yang disetujui
2. verifikasi environment/backend/provider identity
3. tofu fmt/validate/plan
4. review action dan approval
5. apply pada scope yang disetujui
6. collect metadata non-secret
7. host readiness check
8. Ansible bootstrap/configuration dengan limit dan check mode
9. readiness re-check
10. install k3s server bootstrap
11. join server/agent sesuai quorum dan token boundary
12. kubectl health check, node condition, workload smoke test
13. catat evidence dan promote perubahan berikutnya
```

K3s install/runbook sebenarnya menjadi deliverable Fase 4. Modul ini mengajarkan boundary dan sequencing, bukan mengklaim eksekusi k3s.

## 5. Inventory Handoff

Contoh inventory konseptual tanpa credential:

```yaml
all:
  vars:
    environment: lab
    k3s_version: "<approved-version>"
  hosts:
    cp1:
      ansible_host: <management-address-cp1>
      node_role: server
    worker1:
      ansible_host: <management-address-worker1>
      node_role: agent
```

`<management-address-...>` dan `<approved-version>` adalah placeholder, bukan nilai siap pakai. SSH user, private key, k3s token, dan kubeconfig disediakan oleh secret mechanism/runner yang terpisah dan tidak dimasukkan ke repository.

## 6. Ansible Boundary

Fase 4 dapat memiliki role seperti:

```text
common/
  package baseline
  time sync
  user and SSH policy
  firewall
  kernel/sysctl
k3s/
  server bootstrap
  additional server join
  agent join
  version pin
  health checks
```

Ansible playbook harus idempotent, mendukung `--check`/dry-run bila module memungkinkan, menggunakan `--limit` untuk blast radius, dan berhenti pada failed host. Jangan menyalin secret ke output debug. Jangan menganggap `changed=0` berarti k3s sehat tanpa health check cluster.

## 7. k3s Boundary dan Quorum

Sebelum install, tentukan:

- berapa server dan agent;
- embedded etcd atau datastore eksternal;
- server quorum dan failure tolerance;
- network antar-node dan API access;
- disable component yang diperlukan, misalnya servicelb/Traefik sesuai desain;
- snapshot/backup procedure;
- version pin, upgrade, rollback, dan stop condition.

Modul 2.2 dan 2.4 membahas topologi, HA, disable komponen, snapshot, drain, dan upgrade. Jangan menjalankan `k3s server --cluster-reset` atau restore snapshot pada cluster aktif sebagai shortcut troubleshooting.

## 8. Failure Handling

### Provisioning sukses, SSH gagal

Jangan meneruskan Ansible. Periksa IP, route, DNS, firewall, guest agent, console, dan user policy. Object provider tetap harus direkonsiliasi; jangan destroy otomatis tanpa review.

### Satu host bootstrap gagal

Gunakan `--limit` pada host yang disetujui, simpan evidence, perbaiki root cause, lalu ulangi check/readiness. Jangan mengulang semua cluster secara membabi buta.

### Server k3s join gagal

Stop join berikutnya. Periksa version, hostname, time sync, network, token lifecycle, API endpoint, dan quorum. Token tidak boleh dicetak ke log atau disimpan di output.

### OpenTofu plan ingin mengganti host yang sudah dikonfigurasi

Hentikan promotion. Review address, immutable field, image/template, state drift, dan dampak terhadap cluster. Replacement host harus memiliki migration/cordon/drain plan dari owner k3s, bukan sekadar apply.

## Acceptance Checklist

- [ ] Tiga boundary ownership dan handoff sequence tertulis.
- [ ] Output/inventory hanya metadata non-secret.
- [ ] Readiness gate host meliputi network, time, SSH, OS, storage, firewall, dan security.
- [ ] Ansible memakai limit/check/idempotency dan tidak mengambil state provider.
- [ ] k3s readiness mencakup version, quorum, network, backup, dan rollback.
- [ ] Failure mode memiliki stop condition dan tidak membocorkan token/kubeconfig.
- [ ] Handoff runtime belum diklaim tanpa evidence.

## Catatan SRE

Handoff adalah kontrak lintas tim. Metadata yang tidak stabil membuat inventory berubah tanpa kontrol; metadata yang terlalu banyak dapat membocorkan secret atau memperluas ownership. Kirim minimum yang diperlukan, validasi readiness pada boundary, dan simpan evidence setiap transisi.

## Kaitan dengan Modul Berikutnya

- [04 — Production Readiness dan Evidence](04-production-readiness-dan-evidence.md) memperluas gate menjadi evidence chain dan recovery.
- [LAB-02 — Handoff ke Ansible dan k3s](lab/LAB-02-handoff-ke-ansible-dan-k3s.md) menjalankan review statis/disposable.
- Fase 4 — Ansible masih menyusul dan akan menangani playbook nyata.
