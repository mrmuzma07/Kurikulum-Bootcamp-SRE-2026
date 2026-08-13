# 03 — Registry: Push & Pull ke GitLab Container Registry

> Menyimpan image di tempat terpusat agar bisa dipull dari server lain, CI, dan cluster.

## Tujuan
- Paham konsep registry & tag image
- Bisa login, tag, push, pull image ke GitLab Container Registry
- Bisa menjelaskan format alamat registry GitLab & OCI

## 1. Kenapa Registry?

Build image di laptop hanya berguna di laptop itu. Untuk deploy ke server/CI/cluster, image harus ada di **registry** — server penyimpanan image.

```
Build (laptop) ──push──► [Registry] ──pull──► Server/CI/Cluster
                        GitLab Registry
                        Docker Hub
                        GHCR
```

| Registry | Catatan |
|---|---|
| **GitLab Container Registry** | terintegrasi repo GitLab; ada per project; cocok bootcamp |
| Docker Hub | publik default, rate-limit di free |
| GHCR | GitHub |
| Private (Harbor, Quay) | on-prem production |

## 2. Tag Image — Alamat Image

Format: `registry / project / image : tag`

```
registry.gitlab.com  /  username/sre-bootcamp  /  app  :  v1.0.0
└────── host ────────┘  └──── path (repo) ─────┘  image  └ tag ┘
```

- **Tanpa host** (`app:v1`) → default Docker Hub.
- **Tag** = versi. Hindari `:latest` di produksi (tidak reproducible — menunjuk image yang berubah-ubah).
- **Tag semver** (`v1.2.3`) atau git SHA (`abc1234`) → reproducible & traceable.

```bash
# Build dengan tag lengkap:
docker build -t registry.gitlab.com/<username>/sre-bootcamp/app:v1.0.0 .
# Tag image yang sudah ada:
docker tag demo:v1 registry.gitlab.com/<username>/sre-bootcamp/app:v1.0.0
```

## 3. Login ke GitLab Registry

GitLab registry memakai token (bukan password GitLab langsung).

```bash
# Opsi A — personal access token (PAT) dengan scope read_registry/write_registry
# Buat di GitLab: User Settings → Access Tokens → scope: read_registry + write_registry
docker login registry.gitlab.com
# Username: <gitlab-username>
# Password: <PAT>

# Opsi B — via CI job token (otomatis di pipeline, Fase 6)
# CI_REGISTRY_USER=gitlab-ci-token
# CI_JOB_TOKEN=...
docker login -u gitlab-ci-token -p "$CI_JOB_TOKEN" registry.gitlab.com
```

Verifikasi:
```bash
cat ~/.docker/config.json | grep gitlab        # ada entry auth (jangan commit file ini!)
```

## 4. Push & Pull

```bash
# Push
docker push registry.gitlab.com/<username>/sre-bootcamp/app:v1.0.0

# Pull (dari server lain / CI / cluster)
docker pull registry.gitlab.com/<username>/sre-bootcamp/app:v1.0.0

# Jalankan dari registry
docker run -d -p 8080:8080 registry.gitlab.com/<username>/sre-bootcamp/app:v1.0.0
```

Di GitLab UI: **Packages & Registries → Container Registry** melihat daftar image & tag.

## 5. Mengelola Tag & Bersih-bersih

```bash
# Hapus tag dari registry (hati-hati — bisa break deployment yang pakai):
# GitLab UI: Container Registry → image → tag → Delete
# atau via API (lihat docs GitLab)
```

Praktik baik:
- Tag `v1.0.0` immutable — jangan re-push ke tag sama (gunakan `v1.0.1`).
- Hapus tag lama secara berkala (disk registry berbayar/terbatas).
- Tag `latest` boleh untuk dev, **tidak untuk prod**.

## 6. Format OCI & Multi-arch Manifest

Image modern pakai format **OCI (Open Container Initiative)**. Image multi-arch = satu tag berisi **manifest list** yang menunjuk beberapa image per-arch:

```
registry.../app:v1
   ├── manifest list
   │     ├── linux/arm64 → sha256:aaa...  (image arm64)
   │     └── linux/amd64 → sha256:bbb...  (image amd64)
   └── saat pull di arm64 → otomatis pilih arm64
```

Ini dibahas detail di topik [04-multi-arch-arm64](04-multi-arch-arm64.md). Intinya: registry mendukung multi-arch, dan `pull` di Mac M5 otomatis dapat arm64.

## Latihan Cepat (15 menit)

```bash
# 1. Login (butuh PAT — buat dulu di GitLab)
docker login registry.gitlab.com

# 2. Build & tag image demo dari topik 02
docker build -t registry.gitlab.com/<username>/sre-bootcamp/demo:v0.1 demo/

# 3. Push
docker push registry.gitlab.com/<username>/sre-bootcamp/demo:v0.1

# 4. Hapus lokal, lalu pull dari registry (buktikan image tersimpan)
docker rmi registry.gitlab.com/<username>/sre-bootcamp/demo:v0.1
docker pull registry.gitlab.com/<username>/sre-bootcamp/demo:v0.1
docker run --rm registry.gitlab.com/<username>/sre-bootcamp/demo:v0.1

# 5. Cek di GitLab UI: Packages & Registries → Container Registry
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Tag image | `docker tag src dst` |
| Build dengan tag | `docker build -t reg/path:tag .` |
| Login registry | `docker login registry.gitlab.com` (PAT) |
| Push | `docker push reg/path:tag` |
| Pull | `docker pull reg/path:tag` |
| Lihat di UI | GitLab → Packages & Registries → Container Registry |

## Cek Pemahaman

1. Kenapa `:latest` tidak disarankan di produksi, padahal namanya "terbaru"?
2. Bagaimana CI di GitLab bisa push tanpa PAT manual? (petunjuk: job token)
3. Apa format alamat image `nginx` saja (tanpa host), dan ke mana ia menunjuk?
4. Saat `docker pull` di Mac M5, bagaimana registry tahu harus kirim versi arm64?