# Latihan Modul 8.2 — Production On-Prem

## Hari 1 — Network, Storage, Hardware

1. Gambar as-built dan desired-state topology dengan static IP, VLAN, DNS, ARP, MTU, MetalLB, bastion, storage, dan observability.
2. Bandingkan Local PV, NFS, dan SAN untuk tiga workload berbeda.
3. Buat capacity model yang mencakup failure headroom, upgrade overhead, retention, dan blue-green overlap.

## Hari 2 — Access dan Backup

1. Tulis bastion/SSH hardening dan break-glass boundary.
2. Pisahkan backup Kubernetes objects, PV/application data, dan etcd.
3. Definisikan consistency, encryption/key custody, retention, restore order, RPO, dan RTO.

## Hari 3 — Patching dan Upgrade

1. Buat Ansible patch plan dengan `--limit`, serial, approval, maintenance window, dan recovery.
2. Tulis controlled k3s upgrade one-node-at-a-time dengan quorum/PDB review.
3. Buat post-check sebelum menyatakan cluster siap.

## Hari 4 — Security dan DR

1. Buat RBAC, NetworkPolicy, secret rotation, dan Trivy exception review.
2. Tulis DR path OpenTofu → Ansible → k3s → Helm/GitOps → restore → observability/SLO.
3. Buat evidence matrix yang membedakan artifact tersedia dari recovery tervalidasi.

## Hari 5 — Peer Review

Review seluruh output dengan checklist acceptance criteria. Jika runtime tidak dijalankan atau tool/context unavailable, tulis status **belum diverifikasi**.

**Lulus:** minimal 80% dan tidak ada pelanggaran guardrail secret, backup, restore, atau production scope.
