# 04 — Multi-Arch & ARM64

> Build image yang jalan di Apple Silicon (arm64) dan server x86 (amd64) — dan waspadai image yang hanya amd64.

## Tujuan
- Paham konsep arsitektur image & kenapa penting di Mac M5
- Bisa build image multi-arch dengan `docker buildx`
- Bisa mendeteksi image amd64-only & tahu konsekuensinya
- Mengerti trade-off build multi-arch di Mac ARM64 (emulasi QEMU)

## 1. Kenapa Multi-Arch Penting di Bootcamp Ini

MacBook Air M5 = **ARM64**. Tapi production on-prem sering **AMD64** (server x86_64). Image yang hanya `amd64`:
- Di Mac M5: jalan lewat **emulasi QEMU** (lambat, kadang gagal) — atau error `exec format error`.
- Di server amd64: jalan normal.

Image yang hanya `arm64`:
- Di Mac M5: cepat.
- Di server amd64: **gagal total** (`exec format error`).

**Solusi: build satu tag yang berisi kedua arsitektur** (manifest list). Pull di masing-masing host otomatis dapat versi yang cocok.

```
app:v1.0.0  (manifest list)
├── linux/arm64  → untuk Mac M5 & server ARM
└── linux/amd64  → untuk server x86 production
```

## 2. Cek Arsitektur Image

```bash
# Lihat manifest & platform image:
docker manifest inspect <image> | grep -E '"architecture"|"os"|"platform"'
# atau
docker buildx imagetools inspect <image>
```

```bash
# Cek host lokal:
uname -m                  # arm64 (Mac M5) / x86_64
docker info | grep -i arch
```

```bash
# Deteksi image amd64-only (bahaya di Mac M5):
docker pull someimage:latest
docker inspect someimage:latest --format '{{.Architecture}}'    # kalau "amd64" saja → waspada
```

## 3. Setup `buildx`

`buildx` = builder modern yang mendukung multi-platform. Cek & buat builder:

```bash
docker buildx version
docker buildx ls
# Buat builder multi-platform:
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap multiarch
```

## 4. Build Multi-Arch

```bash
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  -t registry.gitlab.com/<username>/sre-bootcamp/app:v1.0.0 \
  --push .
```

| Flag | Fungsi |
|---|---|
| `--platform linux/arm64,linux/amd64` | build untuk kedua arsitektur |
| `--push` | langsung push manifest list ke registry (image multi-arch tidak disimpan lokal utuh) |
| `-t` | tag |

**Tanpa `--push`**: multi-arch image tidak bisa dimuat lokal utuh (manifest list). Untuk uji lokal per-arch:
```bash
# Build arm64 saja untuk uji di Mac:
docker buildx build --platform linux/arm64 -t app:arm64 --load .
docker run --rm app:arm64
```

## 5. Emulasi QEMU — Kenapa Lambat

Saat build `amd64` di Mac ARM64, buildx memakai **QEMU** untuk meng-emulasi instruksi amd64. Ini:
- **Lambat** (5–20x lebih lambat) — install paket & compile makan lama.
- Kadang **gagal** untuk instruksi tertentu.

Strategi:
1. **Build arm64 lokal** untuk dev cepat (`--platform linux/arm64 --load`).
2. **Build multi-arch di CI** (GitLab runner amd64 atau multi-arch) — Fase 6 akan setup ini.
3. Hindari build multi-arch penuh di laptop untuk image besar.

```bash
# Cek emulator terpasang:
docker buildx ls
# "desktop-linux" builder OrbStack sudah punya QEMU untuk cross-arch
```

## 6. Mendeteksi Image amd64-only di Wild

Saat pakai image publik (mis. dari Docker Hub), cek dulu apakah multi-arch:

```bash
docker buildx imagetools inspect nginx:alpine | grep -E "linux/"
# linux/amd64, linux/arm64, ...  → aman
# hanya linux/amd64               → waspada di Mac M5
```

Saat menarik image amd64-only di Mac M5:
```bash
docker run --rm --platform linux/amd64 amd64-only-image
# jalan lewat QEMU (lambat) atau warning "requested platform does not match"
```

**Praktik:** selalu cek `imagetools inspect` sebelum pakai image publik di bootcamp. Kalau amd64-only, cari alternatif multi-arch atau build sendiri.

## 7. Tag per-Archipelago vs Satu Tag Multi-Arch

| Pendekatan | Contoh | Kelebihan | Kekurangan |
|---|---|---|---|
| Satu tag multi-arch | `app:v1` (manifest list) | pull otomatis pilih arch | build lebih kompleks |
| Tag per-arch | `app:v1-arm64`, `app:v1-amd64` | sederhana | consumer harus pilih manual |

**Rekomendasi:** satu tag multi-arch + `--push`. Helm/Kubernetes hanya perlu `app:v1`.

## Latihan Cepat (15 menit)

```bash
# 1. Cek arsitektur host
uname -m
docker info | grep -i arch

# 2. Inspeksi image publik multi-arch
docker buildx imagetools inspect alpine:3.20 | grep -E "linux/"

# 3. Buat builder & build arm64 lokal (cepat di Mac)
cd demo  # folder dari topik 02
docker buildx create --name multiarch --use 2>/dev/null || docker buildx use multiarch
docker buildx build --platform linux/arm64 -t demo:arm64 --load .
docker run --rm demo:arm64          # jalan native arm64 (cepat)

# 4. (Opsional, lambat) Build multi-arch push ke registry
# docker buildx build --platform linux/arm64,linux/amd64 \
#   -t registry.gitlab.com/<username>/sre-bootcamp/demo:v0.2 --push .

# 5. Verifikasi manifest list (kalau sudah push)
# docker buildx imagetools inspect registry.gitlab.com/<username>/sre-bootcamp/demo:v0.2 | grep linux/
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Cek arsitektur host | `uname -m`, `docker info \| grep arch` |
| Cek platform image | `docker buildx imagetools inspect <img> \| grep linux/` |
| Buat builder | `docker buildx create --name x --use` |
| Build multi-arch + push | `docker buildx build --platform arm64,amd64 -t img --push .` |
| Build arm64 lokal uji | `docker buildx build --platform linux/arm64 -t img --load .` |
| Jalankan arch spesifik | `docker run --platform linux/amd64 img` |

## Cek Pemahaman

1. Apa yang terjadi saat `docker run` image amd64-only di Mac M5, dan kenapa lambat?
2. Kenapa build multi-arch tanpa `--push` tidak bisa di-`docker run` secara utuh di lokal?
3. Strategi apa untuk image besar agar build multi-arch tidak menghambat development di Mac M5?
4. Bagaimana satu tag `app:v1` bisa melayani host arm64 dan amd64 secara teknis (manifest list)?