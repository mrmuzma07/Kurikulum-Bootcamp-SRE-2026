# Fase 1 — Container & OrbStack

> **Tujuan fase:** memahami apa itu container, membuat image yang baik & multi-arch, dan menjadikan OrbStack rumah harian untuk eksperimen container sebelum masuk Kubernetes di Fase 2.

## Durasi
Minggu 3

## Modul di Fase Ini

| # | Modul | Durasi | Status |
|---|---|---|---|
| 1.1 | Container Fundamental | 4 hari | ✅ Tersedia |
| 1.2 | OrbStack sebagai Lab Harian | 1 hari | ⏳ |

## Capaian Fase (Wajib)

- [ ] Bisa menjelaskan konsep image vs container, layer, namespace & cgroup secara sederhana
- [ ] Bisa menulis Dockerfile best practice: multi-stage build, non-root user, image kecil
- [ ] Bisa push/pull image ke GitLab Container Registry
- [ ] Bisa build image multi-arch (`buildx`) untuk ARM64 — waspada image amd64-only
- [ ] Bisa menjalankan compose stack (app + database) di OrbStack
- [ ] Bisa membandingkan OrbStack vs Docker Desktop & tahu kenapa OrbStack dipilih di Apple Silicon

## Cara Belajar

1. Selesaikan **Modul 0.1–0.3** dulu (Fase 0) — butuh terminal nyaman + repo GitLab.
2. Tiap modul punya **materi** + **lab** + **evaluasi**.
3. Praktik di **OrbStack** (bukan Docker Desktop) sejak awal — build & run semua image di sini.
4. Semua image yang dibuat **multi-arch** & di-push ke GitLab registry (bekal Fase 5 Helm & Fase 6 GitOps).
5. Isi [_Notes_](https://notes.example.com) pribadi di akhir modul: 3 hal dipelajari + 1 hal masih bingung.

## Sumber Tambahan

- **Docker docs** — docs.docker.com (Dockerfile reference, buildx)
- **OrbStack docs** — docs.orbstack.dev (machine, networking, resource limit)
- **GitLab Container Registry** — docs.gitlab.com/ee/user/packages/container_registry
- **"Containers From Scratch"** — youtube.com/LizRice (mengerti namespace/cgroup via kode)
- **Dive** — github.com/wagoodman/dive (inspeksi layer image)

## Catatan MacBook Air M5

1. **ARM64 everywhere** — Mac M5 = ARM64. Image yang hanya `linux/amd64` akan jalan lewat emulasi (lambat) atau gagal. Build multi-arch wajib.
2. **OrbStack > Docker Desktop** — native ARM64, hemat RAM. Set limit memori (mis. 8 GB) agar Mac tetap responsif.
3. **Emulasi QEMU** — build untuk `amd64` di Mac ARM64 pakai QEMU (lambat). Untuk CI nyata, pakai runner amd64 (Fase 6).