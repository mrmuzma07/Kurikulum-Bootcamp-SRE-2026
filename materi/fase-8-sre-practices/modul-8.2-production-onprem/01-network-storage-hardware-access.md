# 01 — Network, Storage, Hardware, dan Access On-Prem

## Tujuan

Memetakan dependency yang biasanya disediakan cloud tetapi harus dirancang, dimonitor, dan dipulihkan sendiri pada environment on-prem.

## 1. Network Failure Domain

Topologi minimum harus menunjukkan:

```text
bastion/jump host
  → management VLAN → host/VM/k3s nodes
  → workload VLAN → service/MetalLB address pool
  → storage VLAN → NFS/SAN or local disks
  → internal DNS/NTP/registry/observability dependencies
```

Dokumentasikan static IP pool, VLAN ID, subnet, gateway, DNS internal, DHCP boundary, ARP ownership, MTU, switch path, firewall, and routing. MetalLB L2 advertises an address melalui ARP/NDP; BGP membutuhkan peer, ASN, route policy, dan failure handling. Tidak ada cloud load balancer yang menyembunyikan ARP, route, health, atau capacity failure.

IP pool harus allowlisted, tidak overlap dengan DHCP/host/storage, memiliki owner, reservation, monitoring, dan expiry. Duplicate IP, stale ARP, MTU mismatch, asymmetric route, dan DNS drift adalah failure modes yang harus ada di runbook.

## 2. Storage dan Data

| Pilihan | Kelebihan | Risiko/pertanyaan |
|---|---|---|
| Local PV | latency dan locality baik | node/disk failure, rescheduling terbatas |
| NFS | shared access, sederhana | network/server dependency, locking, throughput |
| SAN | enterprise performance/HA | zoning, multipath, credential/key custody |

StorageClass harus mendokumentasikan provisioner, reclaim policy, binding mode, topology, expansion, snapshot, encryption, IOPS, throughput, capacity alert, dan owner. Backup PV bukan hanya backup object claim; pastikan aplikasi/database konsisten dan restore order diketahui.

## 3. Hardware dan Capacity

Host monitoring mencakup node_exporter atau collector setara, CPU/memory, filesystem/inode, disk SMART/health, temperature bila relevan, NIC errors, packet drops, power, dan capacity headroom. Disk full dapat berasal dari logs, container layers, snapshots, WAL, atau backup staging.

Capacity review harus menghitung steady state, failover headroom, upgrade overhead, blue-green overlap, quorum, storage growth, dan retention observability. `node Ready` tidak membuktikan disk sehat atau aplikasi memiliki capacity.

## 4. Bastion dan SSH

Bastion/jump host menjadi access boundary: allowlist source, MFA atau mekanisme yang disetujui, short-lived access, session audit, time limit, dan recovery owner. SSH hardening meliputi key policy, algorithm, forwarding, account separation, patching, logging, dan break-glass procedure.

Semua perubahan OS dilakukan melalui Ansible. Manual SSH hanya break-glass untuk recovery, harus memiliki approval, command/audit summary redacted, expiry, dan follow-up automation agar desired state kembali konsisten.

## 5. Static dan Runtime Lane

Static lane: gambar as-built dan desired-state topology, dependency map, IP/VLAN table placeholder, storage matrix, capacity model, bastion boundary, dan failure matrix.

Disposable runtime: hanya verify/read-only network, ARP, DNS, MTU, storage, host metrics, dan access recovery pada target yang disetujui. Jangan mengubah switch, firewall, IP pool, storage, atau SSH production.

## Acceptance Criteria

- [ ] Static IP, VLAN, DNS, ARP, MTU, MetalLB L2/BGP, dan no-cloud-LB boundary jelas.
- [ ] Local PV/NFS/SAN, StorageClass, IOPS, capacity, consistency, dan backup dibedakan.
- [ ] Host metrics dan disk failure/capacity signal ada.
- [ ] Bastion, SSH hardening, break-glass, audit, dan Ansible ownership jelas.
- [ ] Topology memiliki as-built, desired-state, owner, revision, last-reviewed, dan expiry.
