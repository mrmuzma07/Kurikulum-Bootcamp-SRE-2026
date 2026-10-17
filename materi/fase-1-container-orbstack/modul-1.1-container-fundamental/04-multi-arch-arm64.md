# 04 — Multi-Architecture Build: ARM64 (Apple Silicon) vs AMD64 (Server Production)

> **Ringkasan:** Memahami perbedaan arsitektur prosesor antara laptop kerja **Apple Silicon (`linux/arm64`)** dan server data center on-premises (**`linux/amd64`**), menghindari insiden klasik `exec format error`, memahami cara kerja emulasi **QEMU/Rosetta**, serta merakit satu Container Image pintar (**Multi-Arch Manifest List**) menggunakan **`docker buildx`** yang dapat berjalan cepat di kedua dunia.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal MacBook, kamu akan mampu:
- Menjelaskan perbedaan mendasar antara arsitektur prosesor **ARM64 (`aarch64`)** dan **AMD64 (`x86_64`)** menggunakan analogi bahasa mesin.
- Menjelaskan penyebab terjadinya error legendaris **`exec user process caused: exec format error`** saat men-deploy container lintas mesin.
- Memahami bagaimana **Emulasi QEMU / Rosetta 2** bekerja di balik layar OrbStack beserta dampak penurunan performanya saat kompilasi.
- Menginspeksi daftar dukungan arsitektur dari sebuah image publik di internet sebelum memasangnya ke server menggunakan `docker buildx imagetools inspect`.
- Menyiapkan dan menggunakan **`docker buildx`** untuk membangun image Multi-Arch (`linux/arm64,linux/amd64`) dan mengunggahnya ke GitLab Container Registry.
- Menerapkan teknik optimasi **Native Cross-Compilation (`$TARGETARCH`)** agar proses build multi-arch di Mac M5 selesai dalam hitungan detik tanpa terhambat lambatnya emulasi.

---

## 1. Konsep & Arsitektur: Mengapa Container Bisa "Salah Bahasa"?

### Dua Dunia Prosesor: ARM64 vs AMD64
Terdapat satu fakta perangkat keras (*hardware*) yang sangat krusial di industri saat ini:
1. **Laptop Kerja Kita (MacBook Air/Pro Apple Silicon M1–M5):** Menggunakan keping prosesor berarsitektur **ARM64** (sering ditulis juga sebagai **`aarch64`**). Prosesor jenis ini dirancang sangat hemat baterai, dingin, dan kencang.
2. **Server Production On-Premises (Intel Xeon / AMD EPYC):** Mayoritas server fisik yang terpasang di rak data center perusahaan saat ini menggunakan prosesor berarsitektur **AMD64** (sering ditulis sebagai **`x86_64`**).

### Analogi Bahasa Ibu & Kamus Instruksi Mesin (ISA)
Prosesor komputer tidak mengerti bahasa manusia ataupun bahasa pemrograman seperti Go atau C++. Prosesor hanya mengerti kode biner mesin (**Instruction Set Architecture / ISA**).
- Bayangkan prosesor **ARM64 (Mac M5)** adalah seorang koki yang hanya bisa membaca resep dalam **Bahasa Jepang**.
- Sedangkan prosesor **AMD64 (Server Intel)** adalah seorang koki yang hanya bisa membaca resep dalam **Bahasa Jerman**.

Ketika kamu menjalankan `docker build` biasa di laptop Mac M5 milikmu, compiler di dalam container akan menerjemahkan aplikasimu menjadi biner **Bahasa Jepang (`linux/arm64`)**.

```text
  Laptop MacBook M5 (ARM64)                      Server Production Intel (AMD64)
 ┌──────────────────────────┐                   ┌──────────────────────────────┐
 │ docker build -t app:v1 . │                   │ docker run app:v1            │
 │                          │ ── Push & Pull ─► │                              │
 │ Hasil Biner: Bahasa ARM64│                   │ 💥 ERROR: exec format error! │
 │ (Jalan mulus di Mac!)    │                   │ (CPU Intel tidak paham ARM!) │
 └──────────────────────────┘                   └──────────────────────────────┘
```

### Tragedi Klasik: `exec format error`
Jika kamu langsung mengunggah image hasil build standar dari Mac M5 ke registry, lalu menyuruh server production Intel (`amd64`) untuk menjalankannya, container di server production akan **langsung mati seketika** pada detik pertama dengan pesan error:

```text
exec /app/server: exec format error
```
Pesan `exec format error` adalah teriakan Kernel Linux yang mengatakan: *"File program ini memang ada dan memiliki izin eksekusi, tetapi kode instruksi mesin di dalamnya ditulis untuk jenis keping prosesor yang berbeda!"*

---

## 2. Mengenal Emulasi (QEMU & Rosetta): Saat Mac Menjalankan Image AMD64

Sekarang mari kita balik pertanyaannya:
Bagaimana jika kamu mengunduh sebuah image tua dari internet yang **hanya dibuat untuk Intel (`linux/amd64`)**, lalu kamu jalankan di atas MacBook M5 (`arm64`) milikmu? Mengapa terkadang image tersebut tetap bisa menyala dan tidak langsung error?

### Peran Penerjemah Simultan (QEMU / Rosetta 2)
Aplikasi virtualisasi modern di macOS seperti **OrbStack** telah dilengkapi dengan teknologi penerjemah instruksi otomatis bernama **Rosetta 2** dan **QEMU User-Mode Emulation**.

> 🗣️ **Analogi Penerjemah Bahasa Simultan:**
> Ketika Mac M5 (yang berbahasa Jepang/ARM64) dipaksa menjalankan program berbahasa Jerman (`linux/amd64`), OrbStack memanggil seorang **Juru Bahasa (Emulator)** untuk berdiri di tengah-tengah.
> Setiap kali program `amd64` mengeluarkan satu kalimat perintah, juru bahasa harus menerjemahkannya terlebih dahulu ke bahasa `arm64`, baru kemudian dikerjakan oleh prosesor Mac M5.

### Konsekuensi & Bahaya Mengandalkan Emulasi:
Meskipun terlihat ajaib karena aplikasi tetap bisa jalan, mengandalkan emulasi memiliki **3 harga mahal**:
1. **Lambat & Menguras CPU/Baterai:** Karena setiap instruksi harus diterjemahkan secara *live*, kecepatan eksekusi (terutama saat melakukan kompilasi kode, kompresi, atau enkripsi) bisa **melambat 5x hingga 20x lipat**!
2. **Pemakaian RAM Lebih Boros:** Proses penerjemahan membutuhkan ruang memori tambahan di latar belakang.
3. **Risiko Crash pada Aplikasi Kompleks:** Beberapa aplikasi tingkat rendah (seperti database lawas, mesin Java tertentu, atau program yang memakai instruksi spesifik CPU seperti `AVX2`) bisa tiba-tiba mati mendadak (*Segmentation Fault*) saat dijalankan di atas emulator.

**Kesimpulan SRE:** Emulasi sangat membantu untuk keadaan darurat di laptop lokal, namun untuk beban kerja nyata dan kecepatan maksimal, **selalu gunakan image yang sesuai dengan arsitektur asli mesin (*native*)!**

---

## 3. Solusi Standar Industri: OCI Multi-Arch Image (Manifest List)

Lalu bagaimana caranya agar tim kita tidak pusing? Apakah kita harus membuat dua nama image yang berbeda seperti `app:v1-arm64` untuk di laptop Mac dan `app:v1-amd64` untuk di server kantor?
Jika kita membedakan namanya, maka file konfigurasi Kubernetes dan Helm kita harus terus-menerus diubah setiap kali pindah lingkungan. Itu sangat merepotkan!

Solusi modern dari standar OCI adalah **Multi-Architecture Image** (secara teknis disebut **Manifest List** atau **OCI Image Index**).

```text
              Tag Tunggal: registry.gitlab.com/budi/sre/app:v1.0.0
                                       │
                                       ▼
                    ┌─────────────────────────────────────┐
                    │   MANIFEST LIST (Katalog Indeks)    │
                    ├─────────────────────────────────────┤
                    │ • Jika OS/Arch = linux/arm64 ───────┼──► 📦 Image Layer ARM64
                    │ • Jika OS/Arch = linux/amd64 ───────┼──► 📦 Image Layer AMD64
                    └─────────────────────────────────────┘
                               ▲               ▲
                               │               │
         Mac M5 melakukan pull ┘               └── Server Intel melakukan pull
        (Otomatis dapat ARM64!)                   (Otomatis dapat AMD64!)
```

### Cara Kerja Manifest List
Saat kita merakit image dengan teknik Multi-Arch:
1. Kita merakit dua varian image sekaligus (satu untuk `linux/arm64` dan satu lagi untuk `linux/amd64`).
2. Kita mengikat kedua varian tersebut di bawah **SATU nama tag yang sama** (misal `app:v1.0.0`) menggunakan sebuah katalog indeks bernama **Manifest List**.
3. Ketika kamu mengetik `docker pull app:v1.0.0` di Mac M5, Docker otomatis melirik katalog tersebut dan hanya mengunduh paket `linux/arm64`.
4. Ketika server production Intel mengetik perintah `docker pull app:v1.0.0` yang sama persis, server tersebut otomatis mengunduh paket `linux/amd64`!

---

## 4. Cara Mendeteksi Arsitektur Host & Image di Alam Liar

Sebagai seorang SRE, sebelum kamu menggunakan image apa pun dari Docker Hub atau internet, kamu wajib memeriksa terlebih dahulu apakah image tersebut sudah mendukung multi-arch atau belum.

### 1. Memeriksa Arsitektur Mesin Kita Sendiri
```bash
# Cek arsitektur Kernel OS:
uname -m
# Output di Mac M5: arm64 (atau aarch64 di dalam VM Linux)
# Output di Server Intel: x86_64

# Cek arsitektur mesin Docker (OrbStack):
docker info --format 'OS={{.OSType}} | Arch={{.Architecture}}'
# Output: OS=linux | Arch=aarch64
```

### 2. Menginspeksi Daftar Arsitektur Image di Registry (Tanpa Perlu Pull!)
Gunakan perintah sakti `docker buildx imagetools inspect` untuk mengintip isi katalog sebuah image langsung di server registry:

```bash
docker buildx imagetools inspect alpine:3.20
```

*Contoh tampilan output:*
```text
Name:      docker.io/library/alpine:3.20
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:beefdbd8a1da6d2915566fde36db9db0b524eb737fc57cd1367effd16dc0d06d

Manifests:
  Name:        docker.io/library/alpine:3.20@sha256:1e42bbe2508154c9126d48c2b8a75420c3544343bf86fd041fb7527e017a4b4a
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/amd64

  Name:        docker.io/library/alpine:3.20@sha256:33735bd63cf84d7e388d9f6d297d348c523c044410f553bd878c6d7829612735
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/arm64/v8
...
```
> Lihat daftar `Platform:` di atas! Karena terdapat `linux/amd64` dan `linux/arm64/v8`, maka image `alpine:3.20` **100% aman dan native** untuk dipakai baik di MacBook M5 maupun di server Intel production.

### 3. Memeriksa Arsitektur Image yang Sudah Terunduh di Lokal
Jika image sudah ada di laptopmu, gunakan `docker inspect`:
```bash
docker inspect alpine:3.20 --format 'Arsitektur Image Lokal: {{.Architecture}}'
```
Jika di Mac M5 hasilnya tertulis `amd64`, berarti image tersebut **belum mendukung ARM64** dan akan berjalan lambat menggunakan emulasi!

---

## 5. Menguasai `docker buildx`: Merakit Image Multi-Arch

Perintah `docker build` klasik hanya dirancang untuk merakit satu arsitektur pada satu waktu. Untuk merakit beberapa arsitektur sekaligus, kita menggunakan generasi terbaru mesin perakit Docker bernama **`docker buildx`**.

### Langkah 1: Menyiapkan Builder Multi-Platform
Secara default, OrbStack sudah menyertakan `buildx`. Namun, untuk merakit multi-arch dengan fitur penuh (driver `docker-container`), kita perlu membuat satu *builder instance* khusus:

```bash
# 1. Lihat daftar builder yang ada saat ini
docker buildx ls

# 2. Buat builder baru bernama 'sre-multiarch' dan langsung aktifkan (--use)
docker buildx create --name sre-multiarch --driver docker-container --use

# 3. Nyalakan dan verifikasi kesiapan mesin builder tersebut
docker buildx inspect --bootstrap
```
Pada bagian `Platforms:` di output perintah terakhir, kamu akan melihat daftar panjang: `linux/arm64, linux/amd64, ...`. Artinya mesin perakitmu siap memproduksi image untuk berbagai jenis prosesor!

---

### Langkah 2: Memahami Perbedaan Flag `--load` vs `--push`
Saat menggunakan `docker buildx build`, ada satu aturan teknis yang sangat penting untuk dipahami:
**Penyimpanan image lokal Docker biasa (`docker images`) belum bisa menampung dua arsitektur berbeda dalam satu nama tag yang sama secara bersamaan.**

Oleh karena itu, kita punya 2 mode kerja:

#### Mode A: Uji Coba Cepat di Laptop Lokal (`--platform linux/arm64 --load`)
Saat sedang menulis kode dan ingin mengetesnya cepat di Mac M5 milikmu sendiri, rakitlah untuk `linux/arm64` saja dan gunakan flag **`--load`** agar hasilnya langsung masuk ke daftar `docker images` lokalmu:
```bash
docker buildx build \
  --platform linux/arm64 \
  -t demo-app:uji-lokal \
  --load .
```

#### Mode B: Rilis Multi-Arch Resmi ke Registry (`--platform linux/arm64,linux/amd64 --push`)
Saat kode sudah siap dirilis ke GitLab Registry agar bisa dipakai oleh seluruh tim dan server production, sebutkan kedua platform dipisahkan koma (tanpa spasi!) dan gunakan flag **`--push`** agar Buildx langsung merakit katalog *Manifest List* dan mengirimkannya ke server GitLab:
```bash
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  -t registry.gitlab.com/<username>/sre-bootcamp/demo-app:v1.0.0 \
  --push .
```

---

## 6. Teknik Rahasia SRE: Cross-Compilation Cepat Tanpa Emulasi QEMU!

### Masalah Kecepatan Saat Build Multi-Arch
Ketika kamu menjalankan perintah `--platform linux/arm64,linux/amd64` di MacBook M5 pada Dockerfile biasa:
- Cabang `linux/arm64` akan selesai dirakit dalam **3 detik** (karena jalan *native* di CPU M5).
- Namun cabang `linux/amd64` harus menjalankan seluruh container builder di atas **Emulator QEMU**! Jika aplikasimu besar, cabang `amd64` bisa memakan waktu **3 sampai 10 menit**!

### Solusi Pintar: Gunakan Variabel Otomatis `$BUILDPLATFORM` dan `$TARGETARCH`
Tahukah kamu bahwa compiler modern seperti **Go (Golang)** sebenarnya bisa berjalan secara *native* di prosesor ARM64, tetapi menghasilkan file biner untuk prosesor AMD64 hanya dengan mengubah variabel `GOARCH=amd64`? Proses ini disebut **Native Cross-Compilation** dan berjalan **100% secepat kilat tanpa butuh emulasi QEMU sama sekali!**

Buildx menyediakan variabel otomatis (*built-in ARG*) yang bisa kita panggil di dalam Dockerfile:
- **`$BUILDPLATFORM`**: Arsitektur asli mesin tempat kita sedang melakukan build (yaitu `linux/arm64` di Mac M5).
- **`$TARGETOS`** & **`$TARGETARCH`**: Sistem operasi dan arsitektur tujuan yang sedang diminta oleh flag `--platform` (akan bernilai `arm64` pada putaran pertama, dan bernilai `amd64` pada putaran kedua).

Perhatikan Dockerfile Multi-Stage super cepat berikut ini:

```dockerfile
# ==============================================================================
# STAGE 1: Paksa stage builder SELALU jalan native di CPU asli ($BUILDPLATFORM)!
# ==============================================================================
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder

# Panggil variabel otomatis dari Buildx:
ARG TARGETOS
ARG TARGETARCH

WORKDIR /build
COPY go.mod ./
RUN go mod download
COPY . .

# Compiler Go jalan kencang di ARM64, tapi mencetak biner sesuai TARGETARCH!
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -ldflags="-s -w" -o /build/server .

# ==============================================================================
# STAGE 2: Stage final otomatis mengikuti masing-masing arsitektur target
# ==============================================================================
FROM alpine:3.20
RUN adduser -D -u 10001 sreuser
WORKDIR /app
COPY --from=builder /build/server /app/server
USER 10001
ENTRYPOINT ["/app/server"]
```

Dengan menambahkan `--platform=$BUILDPLATFORM` pada baris pertama `FROM`, merakit image untuk `arm64` DAN `amd64` sekaligus di MacBook M5 kini selesai hanya dalam **hitungan detik**!

---

## 7. Hands-On Mandiri Step-by-Step (20 Menit)

Mari kita praktikkan langsung pembuatan aplikasi Go yang menampilkan arsitektur CPU tempat ia berjalan, merakitnya menjadi **Multi-Arch Image**, mengunggahnya ke GitLab Container Registry, dan menguji menjalankannya dalam dua mode arsitektur berbeda!

### Langkah Pelaksanaan

```bash
# --- LANGKAH 1: Siapkan Direktori & Variabel Username GitLab ---
# Ganti dengan username GitLab milikmu (huruf kecil semua):
export GITLAB_USER="username_anda"

cd ~
mkdir -p lab-multiarch && cd lab-multiarch

# --- LANGKAH 2: Buat Aplikasi Go yang Melaporkan Arsitektur CPU ---
cat << 'EOF' > go.mod
module cek-arch

go 1.22
EOF

cat << 'EOF' > main.go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Printf("Halo SRE! Program ini berjalan pada OS: %s | Arsitektur CPU: %s\n", runtime.GOOS, runtime.GOARCH)
}
EOF

# --- LANGKAH 3: Buat Dockerfile Multi-Arch Cepat (Dengan $BUILDPLATFORM) ---
cat << 'EOF' > Dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder
ARG TARGETOS
ARG TARGETARCH
WORKDIR /build
COPY go.mod main.go ./
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -ldflags="-s -w" -o /build/cek-arch .

FROM alpine:3.20
RUN adduser -D -u 10001 sreuser
COPY --from=builder /build/cek-arch /usr/local/bin/cek-arch
USER 10001
ENTRYPOINT ["/usr/local/bin/cek-arch"]
EOF

# --- LANGKAH 4: Siapkan Builder Buildx ---
docker buildx create --name sre-multiarch --driver docker-container --use 2>/dev/null || docker buildx use sre-multiarch

# --- LANGKAH 5: Build Multi-Arch (ARM64 + AMD64) & Push ke GitLab ---
# Pastikan kamu sudah 'docker login registry.gitlab.com' dari materi 03!
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  -t "registry.gitlab.com/${GITLAB_USER}/sre-bootcamp/cek-arch:v1.0.0" \
  --push .

# --- LANGKAH 6: Inspeksi Manifest List di GitLab Registry ---
docker buildx imagetools inspect "registry.gitlab.com/${GITLAB_USER}/sre-bootcamp/cek-arch:v1.0.0"
```

Perhatikan hasil langkah ke-6: di bawah satu tag `cek-arch:v1.0.0`, kini terdaftar dua manifest (`linux/arm64` dan `linux/amd64`)!

Sekarang, mari kita buktikan bahwa tag tunggal tersebut bisa melayani kedua jenis prosesor:

```bash
# 1. Jalankan secara normal di Mac M5 (Otomatis memilih varian native arm64):
docker run --rm "registry.gitlab.com/${GITLAB_USER}/sre-bootcamp/cek-arch:v1.0.0"
# Output: Halo SRE! Program ini berjalan pada OS: linux | Arsitektur CPU: arm64

# 2. Paksa Docker menarik dan menjalankan varian server Intel (linux/amd64):
docker run --rm --platform linux/amd64 "registry.gitlab.com/${GITLAB_USER}/sre-bootcamp/cek-arch:v1.0.0"
# Output: Halo SRE! Program ini berjalan pada OS: linux | Arsitektur CPU: amd64

# --- LANGKAH 7: Bersihkan Folder Latihan ---
cd ~ && rm -rf lab-multiarch
```

---

## 8. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 masalah khas lintas arsitektur yang wajib kamu kenali beserta cara memperbaikinya:

### Kasus 1: Error `docker exporter does not currently support exporting manifest lists`
- **Gejala:** Kamu menjalankan `docker buildx build --platform linux/arm64,linux/amd64 -t app:v1 .` tanpa menyertakan flag `--push`, atau justru menyertakan flag `--load`.
- **Penyebab:** Kamu meminta Buildx merakit 2 arsitektur sekaligus, namun kamu tidak menyuruhnya mengirim hasilnya ke Registry (`--push`). Penyimpanan lokal Docker standar menolak menerima 2 arsitektur dalam 1 tag.
- **Solusi:**
  - Jika ingin merakit 2 arsitektur (`arm64,amd64`), **wajib tambahkan `--push`** (dengan alamat tag registry lengkap).
  - Jika hanya ingin menguji di laptop lokal tanpa push ke internet (`--load`), sebutkan **1 platform saja**: `--platform linux/arm64 --load`.

---

### Kasus 2: Salah Mengunduh Binary Pihak Ketiga di Dalam `RUN curl/wget`
- **Gejala:** Di dalam Dockerfile, kamu mengunduh alat CLI (misalnya `kubectl` atau `helm`) menggunakan perintah `RUN curl -LO https://.../linux/amd64/kubectl`. Saat image di-build untuk `linux/arm64`, proses build sukses, tetapi saat container dijalankan dan memanggil `kubectl`, muncul **`exec format error`**!
- **Penyebab:** URL unduhan di dalam perintah `curl` tersebut masih diketik mati (*hardcoded*) ke kata `amd64`! Sehingga di dalam image `arm64` pun yang terunduh tetaplah file biner `amd64`.
- **Solusi SRE:** Manfaatkan variabel `ARG TARGETARCH` agar URL unduhan selalu menyesuaikan dengan arsitektur yang sedang dirakit:
  ```dockerfile
  ARG TARGETARCH
  RUN curl -sSLo /usr/local/bin/mytool "https://example.com/releases/mytool-linux-${TARGETARCH}" \
      && chmod +x /usr/local/bin/mytool
  ```

---

### Kasus 3: Peringatan `The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)`
- **Gejala:** Muncul baris peringatan kuning saat menjalankan `docker run` pada sebuah image publik.
- **Penyebab:** Pembuat image tersebut hanya menyediakan versi `amd64` dan belum pernah mem-build versi `arm64`.
- **Tindakan SRE:**
  1. Periksa dengan `docker buildx imagetools inspect <image>` apakah ada tag versi lebih baru dari image tersebut yang sudah mendukung `linux/arm64`.
  2. Jika proyek tersebut *open-source* dan menyediakan `Dockerfile`, kamu bisa mem-build ulang sendiri untuk `linux/arm64` di lokal agar berjalan kencang tanpa membebani CPU Mac milikmu.

---

## 9. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Perintah Multi-Arch (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Cek arsitektur CPU komputer host** | `uname -m` |
| **Intip dukungan platform image di registry** | `docker buildx imagetools inspect <nama-image>` |
| **Cek arsitektur image yang sudah di lokal** | `docker inspect <nama-image> --format '{{.Architecture}}'` |
| **Buat & aktifkan builder multi-arch** | `docker buildx create --name multiarch --use` |
| **Build cepat 1 arsitektur untuk tes lokal** | `docker buildx build --platform linux/arm64 -t app:dev --load .` |
| **Build multi-arch & unggah ke registry** | `docker buildx build --platform linux/arm64,linux/amd64 -t <reg>/app:v1 --push .` |
| **Uji jalankan varian arsitektur tertentu** | `docker run --rm --platform linux/amd64 <nama-image>` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan reflektif berikut untuk memastikan kamu siap menghadapi tantangan laboratorium Modul 1.1:

1. **Mengapa sebuah file biner hasil kompilasi C/Go yang dirakit di atas prosesor Apple Silicon (`arm64`) akan mengeluarkan pesan `exec format error` ketika dijalankan secara langsung di server berprosesor Intel Xeon (`amd64`)?**
2. **Apa yang terjadi di balik layar ketika kamu menjalankan sebuah image yang hanya mendukung `linux/amd64` di atas laptop MacBook M5 milikmu? Apa dampak negatifnya terhadap kecepatan dan pemakaian sumber daya?**
3. **Bagaimana sebuah tag tunggal seperti `nginx:1.27-alpine` bisa memberikan file image `arm64` saat ditarik oleh MacBook, namun memberikan file image `amd64` saat ditarik oleh server Intel?**
4. **Mengapa menambahkan `--platform=$BUILDPLATFORM` pada stage builder (Stage 1) dan menggunakan variabel `GOARCH=$TARGETARCH` dapat mempercepat waktu pembuatan image Multi-Arch secara drastis dibandingkan membiarkan Buildx menjalankan emulasi penuh?**

---
*Selamat! Kamu telah menuntaskan seluruh fondasi teori Modul 1.1. Sekarang saatnya mempraktikkan keahlianmu di skenario dunia nyata melalui laboratorium: [lab/LAB-01-containerisasi-app.md](lab/LAB-01-containerisasi-app.md) dan [lab/LAB-02-compose-stack.md](lab/LAB-02-compose-stack.md)!*