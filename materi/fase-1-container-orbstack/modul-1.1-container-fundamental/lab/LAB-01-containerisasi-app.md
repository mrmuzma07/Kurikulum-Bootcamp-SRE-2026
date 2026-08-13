# LAB-01 — Containerisasi Aplikasi Web (Multi-Stage + Non-Root)

> **Target:** containerisasi aplikasi web sederhana dengan Dockerfile **multi-stage**, **non-root user**, image sekecil mungkin, lalu **build multi-arch** dan push ke GitLab Container Registry.

## Latar Belakang
Ini image yang akan dipakai sepanjang sisa bootcamp: di-deploy sebagai Pod (Fase 2), dipackage Helm (Fase 5), dan di-deploy via ArgoCD (Fase 6). Karena itu harus multi-arch (Mac M5 = arm64, production = amd64) dan di registry, bukan cuma lokal.

## Prasyarat
- [ ] Fase 0 selesai (repo `sre-bootcamp` di GitLab, OrbStack jalan)
- [ ] Sudah baca [02-dockerfile-best-practice](../02-dockerfile-best-practice.md) & [04-multi-arch-arm64](../04-multi-arch-arm64.md)
- [ ] PAT GitLab dengan scope `write_registry` (lihat [03-registry-gitlab](../03-registry-gitlab.md))

## Waktu
± 2–3 jam

## Langkah

### 1. Buat Aplikasi (Pilih: Go / Python / Node)

Pilih salah satu bahasa. Contoh **Go** (paling ringan untuk multi-stage):

```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp   # sesuaikan path repo
git switch -c m1.1-lab01
mkdir -p m1.1/app && cd m1.1/app

cat > main.go <<'EOF'
package main

import (
	"fmt"
	"net/http"
	"os"
	"time"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	name := os.Getenv("APP_NAME")
	if name == "" {
		name = "sre-bootcamp"
	}
	start := time.Now()
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "hello from %s (uptime: %s)\n", name, time.Since(start).Round(time.Second))
	})
	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "ok")
	})
	fmt.Println("listening on :" + port)
	if err := http.ListenAndServe(":"+port, nil); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
EOF

cat > go.mod <<'EOF'
module sre-bootcamp/app
go 1.22
EOF
```

> **Python/Node alternatif:** lihat lampiran di bawah (app dengan `/health` endpoint). Pilih sesuai comfort.

### 2. `.dockerignore` & Dockerfile Multi-Stage + Non-Root

```bash
cat > .dockerignore <<'EOF'
.git
*.md
.env
.env.*
.DS_Store
EOF

cat > Dockerfile <<'DOCKERFILE'
# syntax=docker/dockerfile:1
# ---------- Stage 1: builder ----------
FROM golang:1.22-alpine AS builder
WORKDIR /src
# cache dependensi dulu (jarang berubah)
COPY go.mod ./
RUN go mod download
# lalu kode (sering berubah)
COPY . .
# build biner statik kecil; -s -w buang simbol debug
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/app .

# ---------- Stage 2: runtime kecil ----------
FROM gcr.io/distroless/static-debian12:nonroot
# ENV default
ENV APP_NAME=sre-bootcamp PORT=8080
# salin hanya biner dari builder
COPY --from=builder /out/app /app
# distroless:nonroot sudah non-root (UID 65532)
EXPOSE 8080
ENTRYPOINT ["/app"]
DOCKERFILE
```

**Catatan distroless:** tidak ada shell → `docker exec -it` tidak bisa. Untuk lab yang butuh debug, swap ke `alpine` final stage dengan user eksplisit (lihat lampiran). Untuk image produksi, distroless lebih aman & kecil.

### 3. Build & Uji Lokal (arm64)

```bash
# Build (OrbStack default arm64 native — cepat)
docker build -t app:local .
docker images app:local                 # catat ukuran (harus kecil, ~20-30MB)

# Jalankan & uji endpoint
docker run -d -p 8080:8080 --name app1 app:local
curl http://localhost:8080/             # "hello from sre-bootcamp (uptime: ...)"
curl http://localhost:8080/health       # "ok"

# Verifikasi non-root (distroless:nonroot → UID 65532):
docker inspect app1 --format '{{.Config.User}}'      # 65532 atau nonroot
# (exec shell tidak ada di distroless; skip exec id)
docker rm -f app1
```

### 4. Inspeksi Layer & Lint

```bash
docker history app:local                 # lihat layer (sedikit, kecil)
hadolint Dockerfile 2>/dev/null || echo "install hadolint: brew install hadolint"

# (opsional) dive untuk lihat isi layer:
# brew install dive && dive app:local
```

### 5. Build Multi-Arch & Push ke GitLab Registry

```bash
# 1. Login (PAT scope write_registry)
docker login registry.gitlab.com

# 2. Siapkan builder multi-arch
docker buildx create --name multiarch --use 2>/dev/null || docker buildx use multiarch
docker buildx inspect --bootstrap multiarch

# 3. Build multi-arch + push manifest list
#    (lambat karena amd64 lewat QEMU — sabar; atau skip amd64 di laptop, CI akan build ulang di Fase 6)
REG=registry.gitlab.com/<username>/sre-bootcamp/app
docker buildx build --platform linux/arm64,linux/amd64 -t "$REG:v1.0.0" --push .

# 4. Verifikasi manifest list ada di registry
docker buildx imagetools inspect "$REG:v1.0.0" | grep linux/
# harus muncul: linux/arm64, linux/amd64
```

**Catatan:** kalau build multi-arch terlalu lambat di laptop, build arm64 lokal dulu (`--platform linux/arm64 -t app:arm64 --load`), dan tandai TODO untuk CI Fase 6 yang akan build multi-arch di runner. Tapi **push ke registry wajib** (setidaknya arm64) untuk bekal Fase 5/6.

```bash
# Alternatif cepat: push arm64 saja dulu
docker buildx build --platform linux/arm64 -t "$REG:v1.0.0-arm64" --push .
```

### 6. Pull dari Registry (Bukti Tersimpan)

```bash
docker rmi app:local
docker pull "$REG:v1.0.0"               # atau :v1.0.0-arm64
docker run --rm -p 8080:8080 "$REG:v1.0.0"
curl http://localhost:8080/health        # ok
```

### 7. Commit & MR ke Repo

```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
git add m1.1/app/
git commit -m "feat(m1.1): containerisasi app web multi-stage + non-root

- Dockerfile multi-stage (golang builder -> distroless runtime)
- non-root user (distroless:nonroot UID 65532)
- .dockerignore
- image multi-arch (arm64+amd64) di GitLab registry

Closes #<issue-m1.1-lab01>"
git push -u origin m1.1-lab01
```

Buat MR di GitLab (source `m1.1-lab01` → `main`), deskripsi `Closes #<N>`, **squash & merge**, delete source branch.

## Acceptance Criteria

- [ ] Dockerfile multi-stage (builder + runtime terpisah)
- [ ] Container jalan sebagai **non-root** (verify via inspect user)
- [ ] Endpoint `/` dan `/health` berfungsi (`curl` balas benar)
- [ ] Image final **kecil** (`docker images` — distroless ~20-30MB, alpine ~15-25MB; bukan golang full ~300MB)
- [ ] `.dockerignore` ada
- [ ] `hadolint Dockerfile` bersih (atau warning minim, didokumentasikan)
- [ ] Image ter-push ke GitLab Container Registry (terlihat di UI Packages & Registries)
- [ ] Manifest list multi-arch ada (`imagetools inspect` menampilkan arm64+amd64) **atau** minimal arm64 ter-push + TODO multi-arch di CI
- [ ] `docker pull` dari registry berhasil & jalan
- [ ] File ter-commit di repo via MR (squash + conventional commit)

## Troubleshooting

| Gejala | Solusi |
|---|---|
| `exec /app: exec format error` | Image salah arch; build dengan `--platform` sesuai host, atau pakai multi-arch manifest |
| Build amd64 sangat lambat | Emulasi QEMU; build arm64 lokal, serahkan multi-arch ke CI (Fase 6) |
| `docker login` denied | PAT scope salah (butuh `write_registry`); atau username = gitlab username (bukan email) |
| `denied: access forbidden` saat push | Registry path salah — harus `registry.gitlab.com/<username>/<project>/...`; project name case-sensitive |
| distroless: `docker exec` tidak bisa | Memang distroless tanpa shell; untuk debug pakai `alpine` final stage sementara |
| `COPY go.mod` gagal tapi file ada | Cek `.dockerignore` tidak mengecualikan go.mod; atau `COPY go.mod ./` (titik di akhir) |
| Image masih besar (~300MB) | Pastikan pakai multi-stage & base kecil; cek `docker history` layer mana yang besar |
| `:latest` di manifest tapi ingin arch lain | `docker run --platform linux/arm64 img` untuk paksa |

## Catatan SRE
- Image ini = unit deploy yang reproducible. Tag semver (`v1.0.0`) immutable — jangan re-push; buat `v1.0.1`.
- Multi-arch adalah **persyaratan** di bootcamp ini: Mac M5 (arm64) untuk dev, target production bisa amd64. Tanpa multi-arch, akan ketemu `exec format error` saat deploy.
- distroless aman tapi susah debug. Di production, pertimbangkan debug sidecar atau `:debug` tag varian untuk troubleshooting.

## Lampiran: Aplikasi Alternatif

<details>
<summary>Python (Flask) — multi-stage</summary>

```python
# app.py
import os, time
from flask import Flask
app = Flask(__name__)
start = time.time()
@app.route("/")
def home():
    return f"hello from {os.getenv('APP_NAME','sre-bootcamp')} (uptime: {int(time.time()-start)}s)\n"
@app.route("/health")
def health():
    return "ok\n"
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.getenv("PORT","8080")))
```

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt
COPY app.py .

FROM python:3.12-slim
RUN useradd -r -u 10001 appuser
USER appuser
ENV PATH=/home/appuser/.local/bin:$PATH
COPY --from=builder /root/.local /home/appuser/.local
COPY --from=builder /app/app.py /app/app.py
ENV APP_NAME=sre-bootcamp PORT=8080
EXPOSE 8080
CMD ["python","/app/app.py"]
```
</details>

<details>
<summary>Node (Express) — multi-stage</summary>

```js
// app.js
const express=require('express'),os=require('os');
const app=express(),start=Date.now();
app.get('/',(_,r)=>r.send(`hello from ${process.env.APP_NAME||'sre-bootcamp'} (uptime: ${Math.floor((Date.now()-start)/1000)}s)\n`));
app.get('/health',(_,r)=>r.send('ok\n'));
app.listen(process.env.PORT||8080);
```

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY app.js .

FROM node:20-alpine
RUN adduser -D -u 10001 appuser
USER appuser
COPY --from=builder /app /app
ENV APP_NAME=sre-bootcamp PORT=8080
EXPOSE 8080
CMD ["node","/app/app.js"]
```
</details>

## Lanjut
[LAB-02 — Compose Stack](LAB-02-compose-stack.md)