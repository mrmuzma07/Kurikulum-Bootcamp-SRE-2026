# Modul 2.1 — Konsep & k3d untuk Latihan

> **Tujuan akhir:** memahami arsitektur & objek inti Kubernetes, dan menjalankan k3d sebagai "fast lane" untuk latihan harian dengan `kubectl` survival kit — siap mendalami k3s production (Modul 2.2).

## Capaian Modul (Wajib)

- [ ] Bisa menjelaskan arsitektur Kubernetes: control plane, kubelet, container runtime, etcd
- [ ] Bisa membuat & mengelola Pod, Deployment, Service, Ingress, ConfigMap, Secret, PV/PVC
- [ ] Bisa memakai k3d untuk latihan harian: create/delete cluster, multi-node, port mapping, kubeconfig
- [ ] Bisa memakai `kubectl` survival kit: get, describe, logs, exec, port-forward, top
- [ ] Bisa memakai label/selector & namespace untuk mengorganisir workload
- [ ] Bisa menjalankan debug flow dasar (describe → events → logs → exec)

## Rencana 3 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-arsitektur-k8s](01-arsitektur-k8s.md), [02-objek-inti](02-objek-inti.md) (Pod, Deployment) | [LAB-01](lab/LAB-01-pod-deployment-service.md) |
| 2 | [02-objek-inti](02-objek-inti.md) (Service, Ingress, ConfigMap, Secret, PV/PVC), [03-k3d-latihan-harian](03-k3d-latihan-harian.md) | [LAB-02](lab/LAB-02-ingress-configmap-secret.md) |
| 3 | [04-kubectl-survival-kit](04-kubectl-survival-kit.md) | [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Fase 1 selesai (container fundamental + OrbStack + k3d pernah dipakai di Modul 1.2 LAB-01)
- `kubectl` & `k3d` terpasang (`brew install kubectl k3d`)
- Image app di GitLab registry (`registry.gitlab.com/<user>/sre-bootcamp/app:v1.0.0`)
- Sudah membaca [Fase 2 README](../README.md)

## Deliverables Modul

1. **Cluster k3d** multi-node berjalan dengan app ter-deploy (Pod, Deployment, Service, Ingress).
2. **Manifest YAML** ter-commit di repo `sre-bootcamp/m2.1/` (bukan `kubectl create` imperatif saja).
3. **Lab kedua** memakai ConfigMap + Secret + Ingress multi-path, dengan rollout update.
4. **Catatan kubectl** survival kit (cheatsheet pribadi) di `m2.1/kubectl-notes.md`.
5. **Nilai kuis ≥ 80%**

## Cara Memulai

Modul 1.2 LAB-01 sudah membuat Anda merasakan k3d: buat cluster, deploy app, Ingress, akses dari Mac. Sekarang kita **buka isi Kubernetesnya** — apa sebenarnya Pod, Deployment, Service; bagaimana control plane bekerja; dan bagaimana `kubectl` dipakai sehari-hari SRE. Pola berulang: baca konsep → buat manifest YAML → `kubectl apply` di k3d → amati → break → perbaiki. k3d tetap jadi "fast lane"; k3s di VM (Modul 2.2) datang setelah konsep ini solid.

## Kaitan dengan Modul Berikutnya

- **Objek inti di sini** = fondasi untuk semua fase berikutnya: Helm (Fase 5) = template objek ini; GitOps (Fase 6) = automasi `apply`; observability (Fase 7) = scrape Pod/Service.
- **k3d (fast lane) → k3s (production lane):** Modul 2.2 beralih ke k3s di VM OrbStack untuk simulasi on-prem nyata (systemd, static IP, MetalLB). Konsep sama, beda rumah.
- **kubectl survival kit** = alat sepanjang bootcamp; Fase 2.4 mendalami troubleshooting pola (CrashLoopBackOff, OOMKilled, dsb.).
- **Resource requests/limits** (Modul 2.4) lanjutan dari limit container (Fase 1) — sekarang di level orkestrator.