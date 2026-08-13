# Latihan — Modul 1.1 Container Fundamental

Lakukan latihan di **OrbStack** (`docker` dari Mac). Target: pemahaman konsep + muscle memory.

> **Aturan:** kerjakan di terminal. Catat output penting di `m1.1/lab/log-latihan.md` di repo `sre-bootcamp`.

---

## Hari 1 — Konsep Container

### 1.1 Image vs Container
```bash
# 1. Pull & lihat image
docker pull alpine:3.20
docker images alpine

# 2. Jalankan container, amati
docker run -d --name c1 alpine:3.20 sleep 3600
docker ps
docker exec c1 ps aux              # di dalam: sleep = PID 1
docker rm -f c1
```
Catat: kenapa `ps aux` di dalam container menunjukkan sedikit proses, padahal host punya ratusan?

### 1.2 Layer Filesystem
```bash
# 1. Lihat layer alpine & nginx
docker history alpine:3.20 | head
docker pull nginx:alpine
docker history nginx:alpine | head

# 2. Bandingkan image kecil vs besar
docker pull golang:1.22
docker pull golang:1.22-alpine
docker images | grep golang         # beda ukuran berapa?
```
Jelaskan kenapa `golang:1.22-alpine` jauh lebih kecil dari `golang:1.22`.

### 1.3 Writable Layer & Volume
```bash
# 1. Tulis file, hapus container, cek hilang
docker run --rm alpine sh -c 'echo data > /tmp/x && cat /tmp/x'
docker run --rm alpine cat /tmp/x 2>&1 || true   # tidak ada (container baru)

# 2. Volume: data bertahan
docker volume create t1
docker run --rm -v t1:/d alpine sh -c 'echo hello > /d/x'
docker run --rm -v t1:/d alpine cat /d/x
docker volume rm t1
```

### 1.4 Namespace & Cgroup
```bash
# 1. Bukti container = proses host dengan namespace beda
docker run -d --name p1 alpine:3.20 sleep 3600
HPID=$(docker inspect -f '{{.State.Pid}}' p1)
ps -p $HPID -o pid,comm
ls -l /proc/$HPID/ns/ 2>/dev/null | head      # namespace file
docker rm -f p1

# 2. Limit memori & simulasikan OOM
docker run -d --name lim --memory=30m alpine:3.20 sleep 3600
docker exec lim sh -c 'dd if=/dev/zero bs=1M count=20 2>/dev/null | head -c 25m > /tmp/big' 2>&1 || true
sleep 2
docker inspect lim --format '{{.State.OOMKilled}} {{.State.ExitCode}}'   # true, 137
docker rm -f lim
```
Catat: exit code 137 = 128 + 9 (SIGKILL oleh OOM killer). Ini akan muncul lagi di Kubernetes OOMKilled.

---

## Hari 2 — Dockerfile Best Practice

### 2.1 Membaca Dockerfile
Kerjakan di folder `demo` dari topik 02 (atau buat baru). Jawab di laporan:
```bash
# Lihat history layer image demo
docker build -t demo:v1 . 2>/dev/null || docker build -t demo:v1 demo/
docker history demo:v1
```
1. Berapa layer? Mana yang terbesar?
2. Kalau `RUN go mod download` dipindah ke **atas** `COPY . .`, apa efeknya saat kode berubah?

### 2.2 Non-root & Multi-stage
```bash
# 1. Verifikasi image demo jalan non-root
docker run --rm demo:v1 id 2>/dev/null || echo "distroless tanpa shell — cek inspect"
docker inspect demo:v1 --format '{{.Config.User}}'

# 2. Bandingkan: build Dockerfile BURUK (full base, root) vs BAIK (multi-stage)
# Buat Dockerfile.bad:
cat > Dockerfile.bad <<'EOF'
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o app .
CMD ["/app/app"]
EOF
docker build -f Dockerfile.bad -t demo-bad:v1 .
docker images | grep -E "demo:v1|demo-bad"
```
Catat perbedaan ukuran & user antara `demo:v1` (baik) vs `demo-bad:v1` (buruk).

### 2.3 Cache Layer
```bash
# 1. Build pertama (catat waktu)
time docker build -t demo:cache1 .

# 2. Ubah kode (bukan go.mod), build lagi (harus cepat — cache bertahan)
echo "// touch" >> main.go   # atau app.go
time docker build -t demo:cache2 .
```
Bandingkan waktu. Kalau cache bekerja, build kedua jauh lebih cepat (layer `go mod download` reuse).

### 2.4 Lint
```bash
hadolint Dockerfile 2>/dev/null || echo "brew install hadolint"
```
Perbaiki warning penting, catat mana yang diabaikan & kenapa.

---

## Hari 3 — Registry & Multi-Arch

### 3.1 Registry
```bash
# 1. Login GitLab (butuh PAT)
docker login registry.gitlab.com

# 2. Tag & push
REG=registry.gitlab.com/<username>/sre-bootcamp/demo
docker tag demo:v1 "$REG:latihan"
docker push "$REG:latihan"

# 3. Hapus lokal, pull bukti tersimpan
docker rmi "$REG:latihan"
docker pull "$REG:latihan"
```

### 3.2 Multi-Arch
```bash
# 1. Cek arsitektur host
uname -m

# 2. Inspeksi image publik multi-arch
docker buildx imagetools inspect alpine:3.20 | grep linux/

# 3. Build arm64 lokal (cepat)
docker buildx create --name ma --use 2>/dev/null || true
docker buildx build --platform linux/arm64 -t demo:arm64 --load .
docker run --rm demo:arm64

# 4. (Opsional lambat) Build multi-arch push
# docker buildx build --platform linux/arm64,linux/amd64 -t "$REG:multi" --push .
# docker buildx imagetools inspect "$REG:multi" | grep linux/
```

### 3.3 Deteksi amd64-only
```bash
# Cari image publik yang hanya amd64 (latih kebiasaan cek)
docker buildx imagetools inspect <some-old-image> 2>/dev/null | grep linux/ || echo "hanya amd64?"
```

---

## Hari 4 — Integrasi & Compose

### 4.1 Compose Stack
Kerjakan [LAB-02](../lab/LAB-02-compose-stack.md) dan catat di laporan:
1. Bagaimana app menemukan DB tanpa tahu IP-nya? (nama service)
2. Apa yang terjadi pada data saat `docker compose down` (tanpa `-v`) vs dengan `-v`?
3. Saat `docker compose stop db`, apa status app & kenapa?

### 4.2 Soal Refleksi
Tulis jawaban singkat di `m1.1/lab/log-latihan.md`:
1. Seorang rekan bilang "container itu VM kecil." Koreksi pernyataan ini dengan satu kalimat.
2. Anda menarik image `someapp:latest` di Mac M5 dan dapat `exec format error`. Apa dua kemungkinan penyebab & cara cek?
3. Kenapa `docker history` penting sebelum menerima image publik ke produksi?
4. Compose `depends_on` tanpa `condition: service_healthy` tidak menjamin DB siap — jelaskan kenapa & solusinya.

---

## Catatan Performa

- [ ] Semua latihan di terminal OrbStack
- [ ] Output penting disimpan di `m1.1/lab/log-latihan.md` di repo
- [ ] Bisa menjelaskan image vs container, layer, namespace, cgroup
- [ ] Bisa menulis Dockerfile multi-stage + non-root tanpa lihat contoh
- [ ] Bisa build & push multi-arch ke registry
- [ ] Bisa menjalankan compose stack & menjelaskan network/volume/healthcheck-nya