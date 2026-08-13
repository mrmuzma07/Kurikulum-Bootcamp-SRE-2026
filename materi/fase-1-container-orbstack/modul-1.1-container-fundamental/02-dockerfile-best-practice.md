# 02 — Dockerfile Best Practice

> Menulis Dockerfile yang kecil, aman, dan efisien: multi-stage, non-root, instruksi terurut.

## Tujuan
- Bisa membaca & menulis Dockerfile dengan instruksi yang tepat
- Bisa menerapkan multi-stage build untuk image kecil
- Bisa menjalankan container sebagai non-root user
- Bisa mengoptimalkan cache layer & ukuran image

## 1. Anatomi Dockerfile

```dockerfile
# Komentar dengan #
FROM golang:1.22-alpine AS builder        # base + stage name
WORKDIR /app                               # direktori kerja di dalam image
COPY go.mod go.sum ./                       # salin file (layer)
RUN go mod download                         # jalankan perintah (layer)
COPY . .                                    # salin sisa kode
RUN CGO_ENABLED=0 go build -o app .         # build biner

FROM alpine:3.20                            # stage final (kecil)
RUN adduser -D appuser                      # buat user non-root
COPY --from=builder /app/app /app/app       # salin biner dari stage builder
USER appuser                                # jalankan sebagai user ini
EXPOSE 8080                                 # dokumentasi port (tidak membuka)
ENTRYPOINT ["/app/app"]                     # perintah wajib (tidak bisa di-override mudah)
CMD ["--port=8080"]                         # argumen default (bisa di-override)
```

## 2. Instruksi Penting

| Instruksi | Fungsi | Catatan |
|---|---|---|
| `FROM` | base image | selalu awal (kecuali ARG) |
| `WORKDIR` | set direktori kerja | lebih baik dari `cd` di `RUN` |
| `COPY` | salin file host→image | prefer dari `ADD` (kecuali tar auto-extract) |
| `RUN` | jalankan perintah build | tiap `RUN` = 1 layer |
| `ENV` | set env var | persist di runtime |
| `EXPOSE` | dokumentasi port | **tidak benar-benar membuka port** — hanya info |
| `USER` | jalankan sebagai user | non-root = aman |
| `ENTRYPOINT` | perintah utama | jarang di-override |
| `CMD` | argumen default | mudah di-override: `docker run image arg` |

**`ENTRYPOINT` vs `CMD`:**
```dockerfile
ENTRYPOINT ["/app/app"]          # biner tetap
CMD ["--help"]                   # default; `docker run image --port=8080` ganti ini
```
Pakai keduanya: `ENTRYPOINT` = program, `CMD` = default args. Ini pola paling fleksibel.

## 3. Multi-Stage Build — Image Kecil

Tanpa multi-stage, image berisi **seluruh toolchain build** (compiler, source, dependency) — besar & tidak aman. Multi-stage: build di stage besar, **salin hanya hasil** ke stage kecil.

```dockerfile
# Stage 1: builder (besar, buang setelah build)
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o app .

# Stage 2: final (kecil, hanya biner)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

| Tanpa multi-stage | Dengan multi-stage |
|---|---|
| `golang` base ~300MB + source | `distroless`/`alpine` ~20MB, hanya biner |
| Toolchain ikut ke produksi (risiko) | hanya runtime minimal |

**Pilih base image kecil:**
- `alpine` (~7MB) — musl libc (waspadai binary yang butuh glibc)
- `distroless` — tanpa shell/package manager (aman, tapi susah debug)
- `-slim` varian (mis. `python:3.12-slim`) — lebih kecil dari full

```bash
# Bandingkan ukuran:
docker images | grep -E "golang|alpine|distroless"
docker history <image-final>     # lihat layer final (sedikit)
```

## 4. Non-Root User — Wajib

Default container jalan sebagai **root** (UID 0). Berbahaya: kalau ada celah, attacker dapat root di container. Best practice: buat user biasa.

```dockerfile
# Alpine
RUN adduser -D -u 10001 appuser
USER appuser

# Debian/Ubuntu
RUN groupadd -r app && useradd -r -g app -u 10001 appuser
USER appuser
```

**UID eksplisit** (`-u 10001`) lebih baik dari nama — konsisten lintas image, mudah di-mapping Kubernetes `runAsUser`.

```bash
# Verifikasi jalan sebagai non-root:
docker run --rm <image> id      # uid=10001 (bukan 0)
```

**Catatan:** port < 1024 butuh root. Kalau app di port 80/443 & non-root, bind ke 8080/8443 lalu reverse proxy (Modul 0.2) atau port mapping.

## 5. Optimasi Cache Layer

Docker cache layer per-instruksi. Kalau instruksi berubah, layer itu & **semua di bawahnya** di-rebuild. Urutan penting.

```dockerfile
# BURUK — COPY . . di atas → tiap ubah kode, `go mod download` rebuild (cache miss)
COPY . .
RUN go mod download
RUN go build

# BAIK — dependensi dulu (jarang berubah) → cache bertahan saat kode berubah
COPY go.mod go.sum ./
RUN go mod download        # cache bertahan selama go.mod tidak berubah
COPY . .
RUN go build
```

Aturan: **yang jarang berubah di atas, yang sering berubah (kode) di bawah.**

```bash
# .dockerignore — jangan kirim sampah ke build context (mempercepat & kecil)
echo -e ".git\ntarget/\n*.md\n.DS_Store\n.env" > .dockerignore
```

## 6. Menggabung RUN & Bersih-bersih

Seperti topik 01: instalasi + bersih di satu `RUN`.

```dockerfile
# BAIK
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*

# flag --no-install-recommends: tidak install paket rekomendasi (lebih kecil)
```

## 7. Linter Dockerfile — hadolint

```bash
brew install hadolint
hadolint Dockerfile           # cek best practice otomatis
```

Linter menangkap: `apt-get upgrade` (tidak efektif di container), `:latest` tag (tidak reproducible), `ADD` untuk file biasa (pakai `COPY`), dll.

## Latihan Cepat (20 menit)

```bash
# 1. Buat app Go sederhana
mkdir demo && cd demo
cat > main.go <<'EOF'
package main
import ("fmt"; "net/http"; "os")
func main() {
    port := os.Getenv("PORT")
    if port == "" { port = "8080" }
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "hello from %s\n", os.Getenv("APP_NAME"))
    })
    http.ListenAndServe(":"+port, nil)
}
EOF
cat > go.mod <<'EOF'
module demo
go 1.22
EOF

# 2. Dockerfile multi-stage + non-root
cat > Dockerfile <<'EOF'
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o app .

FROM alpine:3.20
RUN adduser -D -u 10001 appuser
ENV APP_NAME=demo PORT=8080
COPY --from=builder /app/app /app/app
USER appuser
EXPOSE 8080
ENTRYPOINT ["/app/app"]
EOF

echo -e ".git\n*.md" > .dockerignore

# 3. Build & jalankan
docker build -t demo:v1 .
docker images demo:v1              # catat ukuran
docker run -d -p 8080:8080 --name demo1 demo:v1
curl http://localhost:8080/         # "hello from demo"
docker exec demo1 id                 # uid=10001 (non-root!) 
docker rm -f demo1

# 4. Bandingkan dengan build tanpa multi-stage (opsional: build golang full)
# docker build -f Dockerfile.bad -t demo-bad .   # kalau punya versi non-multistage

# 5. Lint
hadolint Dockerfile
```

## Ringkasan

| Mau... | Praktik |
|---|---|
| Image kecil | multi-stage + base kecil (alpine/distroless/slim) |
| Aman | non-root `USER`, UID eksplisit |
| Cache efisien | dependensi atas, kode bawah + `.dockerignore` |
| Layer hemat | gabung `RUN` + bersih-bersih di akhir (`&& rm -rf`) |
| ENTRYPOINT/CMD | ENTRYPOINT=program, CMD=default args |
| Cek best practice | `hadolint Dockerfile` |
| Cek ukuran layer | `docker history <image>` |

## Cek Pemahaman

1. Kenapa `COPY . .` di awal Dockerfile membatalkan cache `go mod download`?
2. Apa keuntungan distroless/alpine vs image full, dan apa trade-off-nya (debug)?
3. Kenapa non-root user penting, dan kenapa pakai UID eksplisit (mis. 10001) bukan nama?
4. Beda `EXPOSE 8080` di Dockerfile vs `-p 8080:8080` di `docker run`?
5. Bagaimana multi-stage build membuat image final lebih kecil secara teknis (layer)?