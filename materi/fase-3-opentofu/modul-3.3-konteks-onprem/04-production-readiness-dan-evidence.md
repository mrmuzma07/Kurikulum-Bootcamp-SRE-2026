# 04 — Production Readiness dan Evidence

> **Tujuan:** menyusun gate sebelum provisioning on-prem, menghubungkan plan dengan approval dan handoff, serta menyiapkan recovery tanpa menganggap apply sukses sebagai bukti service sehat.

## Tujuan Belajar

- menilai readiness provider, network, host, state, dan process;
- membuat evidence chain yang tidak berisi credential atau raw artifact;
- membedakan drift, replacement, incident, dan recovery;
- menetapkan stop condition sebelum apply, bootstrap, dan k3s change.

## 1. Readiness Gates

### Provider dan state

```text
[ ] OpenTofu/provider version dan lock file direview
[ ] endpoint, CA/TLS, identity, dan permission sesuai environment
[ ] backend bucket/key terisolasi
[ ] state lock dan concurrency policy aktif/terverifikasi
[ ] backup/retention/recovery procedure tersedia
[ ] module version dan commit SHA cocok dengan change
```

### Infrastruktur

```text
[ ] hypervisor capacity dan maintenance window
[ ] VM template/image dan architecture sesuai
[ ] network/VLAN/bridge/route/firewall disetujui
[ ] IP/DNS/hostname reservation tidak collision
[ ] storage class/datastore/capacity dan durability jelas
[ ] time sync dan certificate path tersedia
```

### Process

```text
[ ] plan actions create/update/delete/replace dibaca
[ ] policy checks lulus
[ ] approval sesuai environment tersedia
[ ] handoff owner dan rollback decision ditentukan
[ ] incident/change evidence location memiliki retention policy
```

## 2. Network, IP, DNS, dan Storage

On-prem mengharuskan tim memiliki detail yang sering disediakan otomatis oleh cloud:

| Area | Pertanyaan review |
|---|---|
| Management network | Dari runner/Ansible, apakah endpoint host dapat dijangkau? |
| Node network | Apakah control plane, worker, DNS, registry, dan storage saling menjangkau? |
| IP | Siapa owner allocation, reservation, conflict detection, dan release? |
| DNS | Apakah forward/reverse record, TTL, dan resolver policy siap? |
| Firewall | Port mana yang dibuka antar-node dan dari operator? |
| Storage | Apakah disk/data/backup memiliki kapasitas, IOPS, durability, dan restore test? |
| MTU/route | Apakah overlay/MetalLB/VM bridge menggunakan asumsi MTU yang benar? |

`ping` sukses tidak membuktikan API, SSH, TLS, DNS reverse, atau k3s port sehat. Gunakan check spesifik dan simpan hasil yang telah diredáksi.

## 3. Plan dan Policy Review

Review minimal:

```text
address resource/module
provider alias dan endpoint class
backend key/workspace
image/template/version
network/storage references
create/update/delete/replace
count/for_each address change
dependency dan ordering
secret/state exposure
```

Reject plan bila:

- environment atau directory tidak sesuai;
- backend key shared/unknown;
- provider identity tidak dapat dibuktikan;
- host production muncul pada plan lab;
- replacement tidak memiliki migration plan;
- image/tag mutable tidak disetujui;
- artifact tidak memiliki owner/retention;
- plan dibuat dari commit atau provider lock yang berbeda.

## 4. Evidence Chain

Evidence yang baik menghubungkan:

```text
commit SHA
  → module version
  → provider lock/checksum
  → environment input reference
  → backend bucket/key
  → plan summary dan action review
  → approval/change ticket
  → apply result
  → host metadata handoff
  → readiness result
  → Ansible result
  → k3s health result
```

Evidence tidak harus berupa raw state atau plan binary. Simpan summary/action count, resource address, checksum artifact sesuai policy, command, timestamp, operator/job identity, dan link ke system of record. Redact endpoint/identity bila dianggap sensitif dan jangan menyalin token.

## 5. Drift dan Recovery

Drift dapat berasal dari perubahan manual hypervisor, IP/network, template, guest OS, atau konfigurasi k3s. Flow aman:

```text
1. hentikan writer dan perubahan manual lain
2. verifikasi directory, backend, provider identity, dan state
3. jalankan refresh/plan read-only sesuai versi dan policy
4. klasifikasikan drift: desired, remote, state, atau dependency
5. tentukan reconcile, import, state migration, rollback, atau incident
6. buat plan baru dan review replacement/downtime
7. apply hanya setelah approval dan backup/lock siap
8. validasi host, Ansible, dan k3s setelah reconcile
```

State restore bukan rollback resource otomatis. `state rm` bukan delete remote object. Import bukan bukti configuration sudah cocok. Semua state operation adalah perubahan control plane.

## 6. CloudInit dan Image Readiness

CloudInit/image harus memiliki:

- datasource dan user-data contract;
- hostname/network initialization policy;
- package baseline dan repository availability;
- first boot timeout dan console evidence;
- no-credential logging policy;
- behavior saat clone/reboot;
- ownership setelah Ansible mengambil alih.

Jangan menaruh secret bootstrap pada user-data yang masuk log, metadata service, image snapshot, state, atau backup tanpa mekanisme perlindungan dan rotation.

## 7. Production Promotion

Promotion bukan copy state atau plan:

```text
dev design → disposable simulation → provider test → staging plan
→ approval → staging apply/readiness → production change review
→ production plan baru → approval → controlled apply
→ Ansible/k3s handoff → health/SLO evidence
```

Setiap environment memiliki plan baru pada state/context target. Production tidak boleh menerima plan dari environment lain hanya karena HCL sama.

## Troubleshooting

### Plan kosong padahal host berubah

Periksa provider refresh, state backend/key, workspace, data source, filter, dan identity. Jangan menyimpulkan tidak ada drift sebelum memastikan object yang dibaca benar.

### Plan mengganti seluruh VM

Cari immutable field, template/image ID, network/storage reference, provider version, dan state drift. Stop; replacement host memerlukan migration plan serta koordinasi Ansible/k3s.

### Handoff metadata tidak cocok dengan inventory

Bandingkan stable key, hostname, address, role, environment, module version, dan provisioning reference. Jangan memperbaiki dengan menyalin credential atau mengubah host manual tanpa ownership.

### Evidence mengandung secret

Hentikan publication, quarantine artifact, revoke/rotate melalui owner, audit log/cache/backup, lalu perbaiki masking dan retention. `sensitive` bukan encryption.

## Acceptance Checklist

- [ ] Readiness gate provider, state, network, IP/DNS, storage, image, process, dan approval tersedia.
- [ ] Plan review mencakup replacement, address churn, dependency, dan identity.
- [ ] Evidence chain dapat diikuti tanpa raw credential/state/plan.
- [ ] Drift/recovery flow membedakan state mutation dari remote resource rollback.
- [ ] CloudInit secret exposure dan lifecycle dijelaskan.
- [ ] Promotion membuat plan baru pada context target.

## Catatan SRE

Readiness adalah keputusan operasional yang dapat menghentikan automation. Sistem on-prem yang sehat bukan yang selalu apply, melainkan yang dapat menolak perubahan ketika identity, network, state, recovery, atau evidence tidak memenuhi syarat.

## Kaitan dengan Modul Berikutnya

- [LAB-02 — Handoff ke Ansible dan k3s](lab/LAB-02-handoff-ke-ansible-dan-k3s.md) menerapkan readiness dan evidence review.
- Modul 2.4 menyediakan runbook operasi k3s, backup, dan rolling maintenance.
- Fase 4 — Ansible masih menyusul untuk eksekusi configuration management.
