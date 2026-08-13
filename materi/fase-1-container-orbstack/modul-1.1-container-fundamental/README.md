# Modul 1.1 — Container Fundamental

> **Tujuan akhir:** paham apa sebenarnya container, bisa menulis Dockerfile yang baik (multi-stage, non-root, kecil), dan build image multi-arch untuk ARM64 lalu push ke GitLab Container Registry.

## Capaian Modul (Wajib)

- [ ] Bisa menjelaskan image vs container, layer filesystem, dan namespace & cgroup secara konsep
- [ ] Bisa membaca Dockerfile & memahami tiap instruksi (`FROM`, `RUN`, `COPY`, `CMD`, `ENTRYPOINT`)
- [ ] Bisa menulis Dockerfile best practice: multi-stage build, non-root user, image sekecil mungkin
- [ ] Bisa push & pull image ke GitLab Container Registry
- [ ] Bisa build image multi-arch (`docker buildx`) untuk `linux/arm64` + `linux/amd64`
- [ ] Bisa mendeteksi image amd64-only & tahu konsekuensinya di Mac M5
- [ ] Bisa menjalankan compose stack (app + database) di OrbStack

## Rencana 4 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-konsep-container](01-konsep-container.md) | [Latihan:Konsep](evaluasi/latihan.md) |
| 2 | [02-dockerfile-best-practice](02-dockerfile-best-practice.md) | [LAB-01](lab/LAB-01-containerisasi-app.md) (mulai) |
| 3 | [03-registry-gitlab](03-registry-gitlab.md), [04-multi-arch-arm64](04-multi-arch-arm64.md) | [LAB-01](lab/LAB-01-containerisasi-app.md) (selesai) |
| 4 | [LAB-02](lab/LAB-02-compose-stack.md) + review | [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Fase 0 selesai (Linux/shell nyaman, GitLab repo `sre-bootcamp` ada)
- OrbStack terpasang & jalan (`brew install --cask orbstack`)
- Docker CLI available lewat OrbStack (`docker version` jalan dari Mac)
- Sudah membaca [Fase 1 README](../README.md)

## Deliverables Modul

1. **App web sederhana** (Go/Python/Node) ter-containerisasi dengan **multi-stage build** + **non-root user**.
2. **Image multi-arch** (`arm64` + `amd64`) ter-push ke **GitLab Container Registry**.
3. **Compose stack** (app + database) berjalan di OrbStack.
4. **Repo `sre-bootcamp/m1.1`** berisi Dockerfile, compose file, & bukti push.
5. **Nilai kuis ≥ 80%**

## Cara Memulai

Container bukan sihir — itu **proses Linux yang diisolasi** (namespace) + **dibatasi sumber dayanya** (cgroup), di atas **filesystem berlapis** (layer). Modul ini mulai dari konsep "apa itu container" sebelum menulis Dockerfile. Buka OrbStack, jalankan tiap perintah, dan **inspeksi hasilnya** (`docker inspect`, `dive`). Image multi-arch adalah kunci di Mac M5 — tanpa itu, aplikasi akan lambat atau gagal.

## Kaitan dengan Modul Berikutnya

- Image yang dibuat di sini → di-deploy sebagai **Pod** Kubernetes (Modul 2.1)
- GitLab registry + multi-arch → dipakai **Helm chart** (Fase 5) & **ArgoCD GitOps** (Fase 6)
- Compose stack (app+db) → miniatur arsitektur yang nanti jadi Deployment + StatefulSet di k3s
- Konsep layer & image kecil → dasar **image scanning** Trivy (Fase 8)