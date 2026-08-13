# Modul 1.2 — OrbStack sebagai Lab Harian

> **Tujuan akhir:** menjadikan OrbStack rumah harian untuk eksperimen container & Kubernetes — memahami machine Linux, resource limit, perbedaan dengan Docker Desktop, dan menjalankan k3d sebagai jembatan cepat menuju Kubernetes (Fase 2).

## Capaian Modul (Wajib)

- [ ] Bisa membuat & mengelola OrbStack Machine Linux (akses SSH, filesystem sharing)
- [ ] Bisa menjelaskan networking mode OrbStack (bagaimana container/VM dapat IP stabil)
- [ ] Bisa mengatur resource limit (memori/CPU) agar Mac tetap responsif
- [ ] Bisa menjelaskan perbedaan OrbStack vs Docker Desktop & kenapa OrbStack dipilih di Apple Silicon
- [ ] Bisa install & menjalankan k3d cluster di atas OrbStack (create/delete, multi-node, port mapping)
- [ ] Bisa memutuskan kapan pakai k3d (latihan harian) vs VM OrbStack + k3s (simulasi production)

## Rencana 1 Hari

| Sesi | Materi | Lab/Evaluasi |
|---|---|---|
| Pagi | [01-orbstack-machine](01-orbstack-machine.md), [02-resource-limit-vs-docker-desktop](02-resource-limit-vs-docker-desktop.md) | [Latihan:OrbStack](evaluasi/latihan.md) |
| Sore | [03-k3d-on-orbstack](03-k3d-on-orbstack.md) | [LAB-01](lab/LAB-01-k3d-cluster.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 1.1 selesai (container fundamental, image, compose)
- OrbStack terpasang & jalan (`orb version`, `docker version`)
- Sudah membaca [Fase 1 README](../README.md)

## Deliverables Modul

1. **OrbStack Machine `devbox`** (Ubuntu) bisa di-SSH & dipakai sebagai lingkungan eksperimen stabil.
2. **Resource limit** di-set (mis. 8GB RAM) & terverifikasi Mac tetap responsif.
3. **Cluster k3d** (single & multi-node) berjalan di OrbStack dengan app sederhana ter-deploy.
4. **Repo `sre-bootcamp/m1.2`** berisi catatan resource, perbandingan OrbStack vs Docker Desktop, & bukti k3d.
5. **Nilai kuis ≥ 80%**

## Cara Memulai

Modul 1.1 mengajarkan **container**; modul ini mengajarkan **rumah tempat container itu hidup** — OrbStack. Bedanya: Modul 1.1 pakai `docker run` lewat OrbStack tanpa peduli detail platform; sekarang kita buka "mesin" di baliknya: Machine Linux, networking, resource limit, dan k3d sebagai jembatan cepat ke Kubernetes. Ini modul 1 hari — padat tapi praktis. Buka OrbStack, ikuti tiap langkah, dan **bandingkan langsung** dengan Docker Desktop kalau pernah pakai.

## Kaitan dengan Modul Berikutnya

- **k3d di sini** = "fast lane" untuk latihan Kubernetes harian (Fase 2 akan pakai k3d untuk Modul 2.1).
- **VM OrbStack + k3s** = "production-like lane" untuk simulasi on-prem nyata (Modul 2.2 & Fase 4 Ansible).
- **Resource limit** kebiasaan ini penting agar laptop tidak hang saat menjalankan banyak VM/cluster di fase selanjutnya.
- **Networking stabil OrbStack** = fondasi simulasi static IP & MetalLB (Modul 2.3).

Paham kapan k3d vs k3s = keputusan kunci yang dipakai sepanjang Fase 2–9.