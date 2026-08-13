# Fase 2 — Kubernetes: k3d, k3s & MetalLB

> **Tujuan fase:** menguasai konsep & operasi Kubernetes, menjalankan cluster multi-node "mirip production" di on-prem (k3s di VM OrbStack), dan meng-expose service dengan LoadBalancer bare-metal (MetalLB) — semua di satu Mac M5.

## Durasi
Minggu 4–5

## Modul di Fase Ini

| # | Modul | Durasi | Status |
|---|---|---|---|
| 2.1 | Konsep & k3d untuk Latihan | 3 hari | ✅ Tersedia |
| 2.2 | k3s untuk Simulasi Production | 3 hari | ✅ Tersedia |
| 2.3 | MetalLB: LoadBalancer Bare-Metal | 2 hari | ✅ Tersedia |
| 2.4 | Operasi & Troubleshooting | 2 hari | ⏳ menyusul |

## Capaian Fase (Wajib)

- [ ] Bisa menjelaskan arsitektur Kubernetes (control plane, kubelet, container runtime, etcd) secara sederhana
- [ ] Bisa membuat & mengelola objek inti: Pod, Deployment, Service, Ingress, ConfigMap, Secret, PV/PVC
- [ ] Bisa memakai k3d sebagai "fast lane" untuk latihan harian (create/delete, multi-node, port mapping)
- [ ] Bisa memakai `kubectl` survival kit: get, describe, logs, exec, port-forward, top, labels/selectors, namespaces
- [ ] Bisa install k3s single-node & multi-node di VM OrbStack (disable komponen yang tidak perlu)
- [ ] Bisa memasang MetalLB (L2 mode) & meng-expose Service `type=LoadBalancer` yang diakses dari Mac
- [ ] Bisa debug pola umum: CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled
- [ ] Bisa backup & restore snapshot etcd k3s

## Cara Belajar

1. Selesaikan **Fase 1** dulu — container & OrbStack adalah fondasi (container = unit yang diorkestrasi K8s).
2. **Dua "lane" sepanjang fase ini:**
   - **k3d** (container) = cepat untuk belajar objek & operasi (Modul 2.1, sebagian 2.4).
   - **k3s di VM OrbStack** = production-like untuk MetalLB, multi-node, etcd, Ansible bootstrap (Modul 2.2–2.4).
3. Mulai dengan manifest manual (`kubectl apply -f`) untuk paham struktur, baru di Fase 5 beralih ke Helm (template).
4. Tiap lab punya **skenario chaos** (matikan node, hapus Pod) — ini inti SRE: tahu sistem lewat kegagalan.
5. Semua image dipakai dari **GitLab registry** (Fase 1) — bukan `:latest` asal; pakai tag semver.
6. Isi [_Notes_] pribadi di akhir modul: 3 hal dipelajari + 1 hal masih bingung.

## Prasyarat

- Fase 1 selesai: image app di GitLab registry, OrbStack jalan dengan resource limit, k3d pernah dipakai (Modul 1.2 LAB-01).
- `kubectl` & `k3d` terpasang (`brew install kubectl k3d`).

## Sumber Tambahan

- **Kubernetes docs** — kubernetes.io/docs (concepts: Pod, Service, Deployment, Ingress, ConfigMap, Secret)
- **k3s docs** — docs.k3s.io (install, architecture, embedded components, uninstall)
- **k3d docs** — k3d.io (cluster create, port mapping, kubeconfig)
- **MetalLB docs** — metallb.unixtower.org (L2/BGP, IPAddressPool, troubleshooting ARP)
- **"Kubernetes The Hard Way"** — github.com/kelseyhightower (memahami komponen dari nol; untuk konteks, bukan diikuti literal di Mac)
- **kubectl Cheat Sheet** — kubernetes.io/docs/reference/kubectl/cheatsheet

## Catatan MacBook Air M5

1. **k3d untuk harian, k3s di VM untuk production-like.** Jangan campur — k3d tidak bisa MetalLB L2 (container, ARP tidak terpropagasi), k3s di VM bisa. Keputusan ini dipakai sepanjang fase.
2. **Resource limit naik.** Cluster multi-node k3s (3 server + worker) + observability butuh banyak RAM. Naikkan limit OrbStack sementara ke ~10–12 GB untuk lab besar, turunkan lagi setelah selesai.
3. **Image multi-arch.** K3s/k3d di M5 = arm64. Image Anda dari Fase 1 sudah multi-arch (arm64+amd64) → langsung jalan. Kalau pakai image publik amd64-only, akan emulasi (lambat) atau gagal.
4. **Bersih-bersih antar lab.** `k3d cluster delete` setelah selesai; VM OrbStack yang tidak terpakai di-stop. SSD Mac terbatas — `docker system prune` rutin.