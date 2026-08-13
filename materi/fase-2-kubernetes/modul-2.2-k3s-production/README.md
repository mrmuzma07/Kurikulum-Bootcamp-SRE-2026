# Modul 2.2 — k3s untuk Simulasi Production

> **Tujuan akhir:** memasang k3s di VM OrbStack sebagai "production lane" — single-node & multi-node (HA) — menyiapkan topologi on-prem (static IP, disable komponen yang tidak perlu) sebelum MetalLB dipasang di Modul 2.3.

## Capaian Modul (Wajib)

- [ ] Bisa menjelaskan k3s & kenapa cocok untuk on-prem (ringan, 1 biner, embedded etcd)
- [ ] Bisa install k3s single-node di VM OrbStack & `kubectl` dari Mac via kubeconfig
- [ ] Bisa install k3s multi-node HA (`--cluster-init` + join agent/server) di beberapa VM
- [ ] Bisa menjelaskan embedded etcd HA & quorum (berapa server minimal, dampak kehilangan node)
- [ ] Bisa menonaktifkan komponen bawaan (`--disable traefik`, `--disable servicelb`) & tahu kenapa
- [ ] Bisa menjelaskan Traefik (bawaan) vs ingress controller alternatif & trade-off-nya
- [ ] Bisa merancang topologi on-prem (static IP, external LoadBalancer) di OrbStack sebagai simulasi

## Rencana 3 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-k3s-arsitektur-install](01-k3s-arsitektur-install.md) | [LAB-01](lab/LAB-01-k3s-single-node-vm.md) |
| 2 | [02-k3s-multi-node-ha](02-k3s-multi-node-ha.md), [03-disable-komponen-ingress](03-disable-komponen-ingress.md) | [LAB-02](lab/LAB-02-k3s-multi-node-topologi.md) |
| 3 | [04-topologi-onprem](04-topologi-onprem.md) | [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 2.1 selesai (paham arsitektur K8s, objek inti, kubectl survival kit)
- OrbStack Machine `devbox` (Ubuntu) pernah dipakai (Modul 1.2)
- OrbStack resource limit cukup untuk multi-VM (~10–12 GB sementara untuk 3 VM)
- IP OrbStack Machine stabil (`orb ip`) — fondasi static IP
- Sudah membaca [Fase 2 README](../README.md)

## Deliverables Modul

1. **VM OrbStack** ≥3 (mis. `k3s-cp1`, `k3s-w1`, `k3s-w2`) dengan IP stabil, bisa di-SSH dari Mac.
2. **Cluster k3s HA** (3 server, embedded etcd) berjalan; `kubectl` dari Mac via kubeconfig.
3. **Komponen disable** terbukti: Traefik dan/atau servicelb off (siap MetalLB di 2.3).
4. **Diagram topologi** (di `m2.2/topologi.md`) — static IP, peran node, jalur akses dari Mac.
5. **Nilai kuis ≥ 80%**

## Cara Memulai

Modul 2.1 memakai **k3d** (k3s dalam container) untuk cepat belajar objek. Sekarang beralih ke **k3s di VM OrbStack** — "production lane": install k3s di Ubuntu VM sungguhan (systemd, static IP), persis seperti server on-prem. Konsep objeknya sama persis (Pod, Deployment, Service — topik 02.1 berlaku), tapi *rumahnya* berbeda: bukan container, melainkan VM dengan kernel & init system penuh. Inilah yang memungkinkan MetalLB L2 (Modul 2.3), Ansible bootstrap (Fase 4), dan etcd backup nyata (Modul 2.4).

## Kaitan dengan Modul Berikutnya

- **k3s di sini → MetalLB (2.3):** kita disable `servicelb` (klipper) sekarang agar MetalLB bisa mengambil alih Service `type:LoadBalancer`. Tanpa disable ini, dua LB bentrok.
- **Topologi on-prem di sini → Ansible (Fase 4):** install k3s multi-node secara manual di lab ini akan terasa berulang; Fase 4 meng-automasi dengan Ansible (inventory, role, bootstrap).
- **HA etcd di sini → backup (2.4):** cluster HA butuh backup etcd (snapshot). Single-node juga bisa; multi-node HA = lebih realistis.
- **k3d vs k3s (final):** sekarang Anda punya kedua lane. Fase 6 GitOps akan pakai k3d "staging" + k3s "production" — paham perbedaan adalah keputusan kunci.