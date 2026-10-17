# 02 — Dockerfile Best Practice: Multi-Stage, Non-Root & Optimasi Layer

> **Ringkasan:** Menguasai seni meracik Container Image berstandar production: memahami cara kerja *Build Context* dan setiap instruksi di dalam `Dockerfile`, memangkas ukuran image hingga 95% dengan teknik **Multi-Stage Build**, menutup celah keamanan dengan **Non-Root User**, mempercepat waktu kompilasi menggunakan **Layer Caching**, serta mengaudit kualitas Dockerfile secara otomatis dengan **Hadolint**.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Menjelaskan peran **Dockerfile** dan mekanisme kerja **Build Context** serta **`.dockerignore`**.
- Membedakan fungsi setiap instruksi utama (`FROM`, `WORKDIR`, `COPY` vs `ADD`, `RUN`, `ARG` vs `ENV`, `EXPOSE`, `ENTRYPOINT` vs `CMD`).
- Memahami mengapa penulisan *Exec Form* (`["./app"]`) wajib digunakan dibandingkan *Shell Form* demi kelancaran *Graceful Shutdown* (`SIGTERM`).
- Menerapkan **Multi-Stage Build** untuk memisahkan lingkungan kompilasi (*builder*) dari lingkungan produksi (*runtime*).
- Memilih pondasi **Base Image** yang tepat antara varian *Full*, *-slim*, *Alpine*, dan *Distroless*.
- Mengonfigurasi container agar berjalan menggunakan **Non-Root User** dengan UID numerik eksplisit.
- Mengurutkan instruksi Dockerfile secara strategis agar **Layer Cache** bekerja maksimal.
- Menguji dan memvalidasi kualitas Dockerfile menggunakan linter standar industri (**`hadolint`**).

---

## 1. Konsep & Arsitektur: Apa Itu Dockerfile & Build Context?

### Analogi Buku Resep Otomatis
Jika **Container Image** adalah kue yang sudah jadi dan siap disajikan, maka **Dockerfile** adalah **Lembar Resep Masakan** berisi urutan instruksi teks langkah demi langkah.
Saat kamu menjalankan perintah `docker build`, mesin Docker (BuildKit) membaca resep tersebut dari baris paling atas hingga baris paling bawah untuk merakit sebuah Image secara otomatis dan konsisten (*reproducible*).

---

### Rahasia Tanda Titik (`.`) pada `docker build`: Mengenal *Build Context*
Ketika membuat image, kita selalu mengetik perintah seperti ini:

```bash
docker build -t aplikasi-ku:v1 .
```

**Pertanyaan penting:** Apa arti tanda titik (`.`) di ujung paling kanan perintah tersebut?
Banyak pemula mengira tanda titik itu artinya *"cari file bernama Dockerfile di folder ini"*. **Itu anggapan yang kurang tepat!**

Tanda titik (`.`) tersebut menunjuk ke **Build Context (Konteks Pembangunan)**. Artinya:
1. Sebelum proses pembuatan image dimulai, Docker CLI akan **mengemas seluruh file dan subfolder yang ada di direktori `.` (saat ini)** lalu mengirimkannya ke mesin pembuat image (*Docker Daemon / BuildKit*).
2. Jika di dalam folder kerjamu terdapat folder `.git` sebesar 500 MB, folder `node_modules` sebesar 1 GB, atau video rekaman layar, maka setiap kali kamu mengetik `docker build`, Docker akan mengangkut beban 1,5 GB tersebut terlebih dahulu! Proses build akan terasa sangat lambat.

```text
  Folder Kerja di Laptop (Build Context)            Mesin Pembuat Image (BuildKit)
 ┌──────────────────────────────────────┐         ┌──────────────────────────────┐
 │ 📄 main.go          ──(Dikirim)──────────────► │                              │
 │ 📄 Dockerfile       ──(Dikirim)──────────────► │    Merakit Lapisan Image     │
 │ 📁 .git (500 MB)    ──(DICEGAH!)─┐             │    Sesuai Isi Dockerfile     │
 │ 📄 .env (Password!) ──(DICEGAH!)─┼─ Oleh       │                              │
 └──────────────────────────────────┼──.dockerignore─────────────────────────────┘
                                    ▼
                              [ Dibuang ]
```

### Pertahanan Pertama: File `.dockerignore`
Persis seperti `.gitignore` pada Git, kita **WAJIB** selalu membuat file bernama **`.dockerignore`** di folder proyek sebelum menjalankan `docker build`. Fungsinya adalah mencegat file sampah, dokumentasi, dan kredensial rahasia agar tidak ikut terlempar ke dalam *Build Context* maupun tidak sengaja tersalin lewat perintah `COPY . .`.

Contoh isi `.dockerignore` standar SRE:
```dockerignore
# Abaikan folder riwayat Git dan konfigurasi lokal
.git
.gitignore

# Abaikan file kredensial dan rahasia!
.env
.env.*
*.pem
*.key

# Abaikan dokumentasi yang tidak dibutuhkan oleh biner aplikasi
README.md
docs/

# Abaikan file sampah sistem operasi macOS & editor
.DS_Store
.vscode/
.idea/

# Abaikan hasil kompilasi lokal / dependensi mentah
node_modules/
__pycache__/
*.log
```

---

## 2. Membedah Anatomi & Instruksi Penting Dockerfile

Mari kita lihat sebuah contoh `Dockerfile` lengkap, lalu kita bedah fungsi setiap kata kuncinya secara mendalam:

```dockerfile
# Baris diawali tanda pagar (#) adalah komentar
FROM python:3.12-slim-bookworm

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PORT=8080

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --system --uid 10001 appuser
USER 10001

EXPOSE 8080

ENTRYPOINT ["python"]
CMD ["main.py"]
```

---

### Kamus Instruksi Dockerfile untuk Pemula

#### 1. `FROM` (Menentukan Pondasi Dasar)
Setiap Dockerfile **wajib** diawali dengan `FROM` (hanya boleh didahului oleh `ARG`). Instruksi ini menentukan sistem operasi atau lingkungan dasar apa yang kita gunakan sebagai lantai pertama.
- ❌ **Hindari tag `:latest`** (misal `FROM ubuntu:latest` atau `FROM node`). Mengapa? Karena bulan depan isi `:latest` bisa berubah drastis ke versi mayor baru dan merusak aplikasimu secara tiba-tiba!
- ✅ **Gunakan versi spesifik yang terkunci (Pinned Version):** Misal `FROM alpine:3.20` atau `FROM golang:1.22.5-alpine3.20`.

#### 2. `WORKDIR` (Mengatur Folder Kerja Aktif)
Berfungsi seperti perintah `mkdir -p` sekaligus `cd` di dalam image. Semua instruksi `RUN`, `COPY`, `ENTRYPOINT`, dan `CMD` di bawahnya akan dieksekusi di dalam folder ini.
- ❌ Jangan pernah menulis `RUN cd /app && ...` karena di instruksi baris berikutnya posisi folder akan kembali ke `/`!
- ✅ Selalu gunakan `WORKDIR /app`.

#### 3. `COPY` vs `ADD` (Menyalin File dari Host ke Dalam Image)
Keduanya berfungsi menyalin file dari komputer lokalmu ke dalam lapisan image. Lalu apa bedanya?
- **`COPY` (Standar Wajib):** Tugasnya murni hanya menyalin file dan folder lokal secara jujur dan transparan.
- **`ADD` (Hindari kecuali darurat):** Memiliki fitur tersembunyi—bisa mengunduh file dari URL internet dan otomatis mengekstrak file arsip `.tar.gz`. Perilaku otomatis yang tersembunyi ini sering memicu celah keamanan (*zip slip*) atau membengkakkan layer. **Aturan SRE: Selalu gunakan `COPY`!**

#### 4. `RUN` (Menjalankan Perintah Saat Proses Perakitan Image)
Digunakan untuk menginstal paket tambahan, membuat user, atau mengompilasi kode. Ingat kembali materi sebelumnya: **Setiap satu baris `RUN` menciptakan satu layer baru!**

#### 5. `ARG` vs `ENV` (Variabel Build vs Variabel Runtime)
Pemula sering bingung membedakan keduanya:

```text
 [ Proses docker build ] ──────────────────────► [ Container Berjalan (docker run) ]
  └── ARG hidup di sini saja!                     └── ENV hidup di saat build DAN saat jalan!
```
- **`ARG` (Build Argument):** Variabel sementara yang hanya ada **selama image sedang dirakit** (misal: `ARG APP_VERSION=1.0.0`). Begitu image jadi dan dijalankan sebagai container, variabel `ARG` ini lenyap.
- **`ENV` (Environment Variable):** Variabel lingkungan yang tetap hidup dan bisa dibaca oleh kodingan aplikasimu **saat container sedang berjalan di production** (misal: `ENV PORT=8080`).

#### 6. `EXPOSE` (Sekadar Dokumentasi, Bukan Pembuka Port!)
> 💡 **Membongkar Salah Kaprah Terbesar Pemula:**
> Menulis `EXPOSE 8080` di dalam Dockerfile **TIDAK otomatis membuka atau mempublikasikan port 8080 ke luar!**
> `EXPOSE` hanyalah secarik catatan metadata (stiker informasi) dari pembuat image kepada pengguna image bahwa *"Aplikasi di dalam kotak ini mendengarkan di port 8080"*. Untuk benar-benar menghubungkan port tersebut ke laptopmu, kamu tetap wajib mengetik flag `-p 8080:8080` saat menjalankan `docker run`.

---

### 7. `ENTRYPOINT` vs `CMD` (Serta Mengapa *Exec Form* Sangat Krusial!)

Kedua instruksi ini menentukan **program apa yang akan dijalankan sebagai proses utama (PID 1)** saat container dinyalakan.

#### A. Perbedaan Peran `ENTRYPOINT` dan `CMD`
- **`ENTRYPOINT`**: Mengunci program utama yang wajib dijalankan oleh container. Tidak mudah ditimpa begitu saja saat orang mengetik argumen tambahan di `docker run`.
- **`CMD`**: Menyediakan argumen bawaan (*default arguments*) untuk `ENTRYPOINT` tersebut, yang sangat mudah diubah/ditimpa saat menjalankan `docker run`.

*Pola Kombinasi Terbaik:*
```dockerfile
ENTRYPOINT ["/usr/bin/curl"]
CMD ["-I", "https://example.com"]
```
- Jika dijalankan biasa `docker run my-curl` ➔ mengeksekusi `curl -I https://example.com`.
- Jika dijalankan dengan argumen baru `docker run my-curl https://google.com` ➔ `CMD` lama otomatis diganti menjadi `curl https://google.com`!

#### B. Bahaya *Shell Form* vs Keunggulan *Exec Form* (Konteks SRE & Kubernetes)
Ada dua cara menulis `ENTRYPOINT` atau `CMD`:

| Bentuk Penulisan | Sintaks di Dockerfile | Proses apa yang menjadi PID 1 di Container? |
|---|---|---|
| **Shell Form (❌ Buruk)** | `ENTRYPOINT /app/server` | `/bin/sh -c "/app/server"` (Shell yang menjadi PID 1!) |
| **Exec Form (✅ Wajib)** | `ENTRYPOINT ["/app/server"]` | `/app/server` (Aplikasi langsung menjadi PID 1!) |

**Mengapa SRE sangat melarang *Shell Form* untuk aplikasi utama?**
Ketika Kubernetes atau Docker ingin mematikan container secara sopan (*Graceful Shutdown*), sistem operasi mengirimkan sinyal **`SIGTERM`** ke proses nomor **PID 1** agar aplikasi punya waktu 30 detik untuk menyelesaikan transaksi database yang sedang berjalan sebelum benar-benar mati.
- Jika kamu memakai **Shell Form**, yang menjadi PID 1 adalah program `/bin/sh`. Sayangnya, `/bin/sh` **tidak meneruskan sinyal `SIGTERM` tersebut ke aplikasimu**!
- Akibatnya: Aplikasimu tidak tahu kalau ia diminta berhenti. Setelah menunggu 10–30 detik tanpa respon, sistem akan kehilangan kesabaran dan menembak mati aplikasimu secara paksa dengan **`SIGKILL` (Exit Code 137)** di tengah-tengah transaksi pelanggan!
- Dengan **Exec Form** (`["/app/server"]`), aplikasimu langsung menduduki kursi PID 1 dan menerima sinyal `SIGTERM` dengan mulus.

---

## 3. Best Practice #1: Multi-Stage Build (Rahasia Image Kecil & Aman)

### Masalah Dockerfile Tradisional (Single-Stage)
Bayangkan kamu membeli sebuah **kue ulang tahun**. Apakah kamu ingin penjual kue mengirimkan kue tersebut lengkap dengan **kompor oven, tabung gas, karung tepung, cangkang telur pecah, dan panci kotor** di dalam kotak kue yang sama? Tentu tidak! Kotaknya akan menjadi sangat berat, kotor, dan berbahaya.

Namun, itulah yang terjadi jika kita membangun aplikasi (seperti Go, Rust, Java, atau TypeScript/React) menggunakan Dockerfile biasa (*Single-Stage*):
- Untuk mengompilasi 1 file kode Go menjadi program biner berukuran **12 MB**, kita membutuhkan image `golang` yang berisi compiler, SDK, dan alat sistem seberat **~800 MB**.
- Jika kita langsung memakai image tersebut ke server production, kita membawa seluruh alat kompilasi (bahkan source code mentah!) ke server production.

### Solusi: Multi-Stage Build (Memisahkan Pabrik dari Kotak Pengiriman)
Sejak Docker memperkenalkan fitur **Multi-Stage Build**, kita bisa menggunakan beberapa instruksi `FROM` di dalam satu file `Dockerfile` yang sama!

```text
 ┌───────────────────────────────────────────────┐
 │ STAGE 1: BUILDER (Dapur / Pabrik Perakitan)   │
 │ Base: golang:1.22-alpine (~300 MB)            │
 │ - Unduh library                               │
 │ - Kompilasi source code ──► Hasil: /app/biner │
 └───────────────────────┬───────────────────────┘
                         │
                         │ COPY --from=builder /app/biner
                         │ (Hanya ambil kue matangnya, buang dapurnya!)
                         ▼
 ┌───────────────────────────────────────────────┐
 │ STAGE 2: FINAL RUNTIME (Kotak Pengiriman)     │
 │ Base: alpine:3.20 (~7.8 MB)                   │
 │ - Hanya berisi /app/biner (12 MB)             │
 │ Total Ukuran Akhir: ~20 MB! (Hemat >90%)      │
 └───────────────────────────────────────────────┘
```

Perhatikan betapa elegan penulisannya:

```dockerfile
# ==============================================================================
# STAGE 1: Tahap Kompilasi (Diberi nama alias 'builder')
# ==============================================================================
FROM golang:1.22-alpine AS builder

WORKDIR /build
COPY go.mod ./
RUN go mod download

COPY . .
# Kompilasi menjadi satu file biner statis mandiri tanpa simbol debug (-s -w)
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /build/myapp .

# ==============================================================================
# STAGE 2: Tahap Produksi Akhir (Bersih, Ramping & Aman)
# ==============================================================================
FROM alpine:3.20

WORKDIR /app
# Salin HANYA file biner matang dari stage 'builder' di atas!
COPY --from=builder /build/myapp /app/myapp

USER 10001
ENTRYPOINT ["/app/myapp"]
```

### Mengapa SRE Sangat Mencintai Multi-Stage Build?
1. **Keamanan Tingkat Tinggi (Minimal Attack Surface):** Di dalam image final tidak ada compiler `gcc`, tidak ada `git`, tidak ada `curl` atau `wget`, dan tidak ada kode sumber mentah. Jika peretas berhasil menyusup ke aplikasi, mereka tidak memiliki peralatan apa pun di dalam container untuk mengunduh malware tambahan!
2. **Kecepatan Deployment & Auto-Scaling:** Mengunduh (*pull*) image berukuran 15 MB ke 50 node Kubernetes hanya memakan waktu 1 detik, dibandingkan mengunduh image 1 GB yang memakan waktu bermenit-menit saat server sedang kebanjiran trafik.
3. **Hemat Biaya Penyimpanan Registry & Disk Node.**

---

### Panduan Memilih Base Image untuk Stage Final

| Jenis Base Image | Ukuran Rata-rata | Karakteristik & Kegunaan | Trade-Off (Kelebihan vs Kekurangan) |
|---|---|---|---|
| **Full OS** (`ubuntu:24.04`, `debian:bookworm`) | 75 MB – 120 MB | Lengkap dengan library `glibc` standar dan package manager `apt`. | Sangat kompatibel untuk semua aplikasi, namun ukurannya cukup gemuk. |
| **Varian `-slim`** (`python:3.12-slim`, `debian:bookworm-slim`) | 40 MB – 130 MB | Versi diet dari Debian; membuang manual docs dan paket non-esensial. | **Titik tengah terbaik** untuk aplikasi Python, Node.js, atau Ruby. |
| **Alpine Linux** (`alpine:3.20`) | **~7.8 MB** | Super kecil, punya shell `sh` dan package manager `apk`. | Menggunakan pustaka **`musl libc`** (bukan `glibc` standar). Kadang memicu masalah pada library C/Python tertentu. |
| **Google Distroless** (`gcr.io/distroless/static-debian12`) | **~2 MB – 20 MB** | **Tanpa Shell (`sh`/`bash`) sama sekali**, tanpa `ls`, tanpa package manager! Hanya berisi file sistem paling dasar untuk menjalankan biner. | **Paling aman di dunia** (peretas tidak bisa melakukan *remote shell*), tapi kamu tidak bisa melakukan `docker exec -it <nama> sh` untuk debugging manual! |

---

## 4. Best Practice #2: Keamanan Non-Root User (Wajib!)

### Bahaya Default Container: Berjalan Sebagai `root` (UID 0)
Secara default, jika kamu tidak menuliskan instruksi `USER` di dalam Dockerfile, maka proses di dalam container akan dijalankan sebagai **`root` (UID 0)**.

Mengapa ini sangat berbahaya di lingkungan production?
Ingat bahwa container berbagi **Kernel yang sama** dengan server Host utama! **`UID 0` (root) di dalam container adalah `UID 0` yang sama di mata Kernel Linux Host.**
Jika aplikasimu memiliki celah keamanan atau salah konfigurasi (misalnya volume host ter-mount secara sembarangan, atau ada bug pada kernel), penyerang yang berhasil menguasai proses di dalam container akan langsung memiliki hak akses **Superuser `root`** di server fisik aslimu!

### Solusi: Buat dan Gunakan User Biasa (Non-Root)
Terapkan prinsip **Least Privilege** (Hak Akses Seminimal Mungkin). Buat pengguna khusus tanpa hak administrator dan tanpa akses login shell:

#### Cara Membuat Non-Root User di Alpine Linux:
```dockerfile
FROM alpine:3.20
# -D = Jangan beri password, -u 10001 = Tetapkan nomor UID eksplisit 10001
RUN adduser -D -u 10001 appuser
USER 10001
```

#### Cara Membuat Non-Root User di Debian / Ubuntu (`-slim`):
```dockerfile
FROM debian:bookworm-slim
# --system = user sistem, --no-create-home = tanpa folder home, --uid 10001
RUN groupadd --system --gid 10001 appgroup \
    && useradd --system --uid 10001 --gid 10001 --no-create-home --shell /sbin/nologin appuser
USER 10001:10001
```

> 🔑 **Mengapa SRE Memilih Angka `USER 10001` Dibanding Nama `USER appuser`?**
> 1. **Kenapa angka di atas 10.000 (misal `10001`)?** Di sistem operasi Linux host, UID nomor `1` sampai `999` dipakai oleh daemon sistem, dan UID `1000` sampai `9999` biasanya dipakai oleh akun manusia biasa (admin/staf). Memakai angka `10001` menjamin bahwa UID container tidak akan pernah kebetulan bentrok dengan UID user manusia yang ada di server host!
> 2. **Kenapa tulis angkanya (`USER 10001`) bukan namanya (`USER appuser`)?** Saat nanti kamu men-deploy container ke **Kubernetes** dengan kebijakan keamanan tinggi (`securityContext.runAsNonRoot: true`), Kubernetes harus memastikan container tidak berjalan sebagai UID 0. Jika kamu menulis nama teks `USER appuser`, Kubernetes tidak tahu `appuser` itu nomor berapa sebelum menyalakan container, sehingga **Kubernetes akan menolak menjalankan Pod-mu** (`CreateContainerConfigError: container has runAsNonRoot and image has non-numeric user`)! Menulis angka `USER 10001` membuat Kubernetes langsung percaya dan mengizinkannya jalan.

### Aturan Penting Non-Root: Jangan Pakai Port di Bawah 1024!
Di Linux, nomor port `1` sampai `1023` (seperti port `80` untuk HTTP dan `443` untuk HTTPS) disebut **Privileged Ports** dan hanya boleh diikat oleh `root`.
Jika container-mu berjalan sebagai user `10001` dan mencoba menyalakan server di port `80`, aplikasi akan langsung mati dengan pesan error: **`bind: permission denied`**.
- **Solusi:** Selalu jalankan aplikasimu di port tinggi seperti **`8080`**, **`3000`**, atau **`8443`**, lalu biarkan Reverse Proxy (Caddy/Nginx) atau Service Kubernetes yang meneruskan trafik dari port 80 ke 8080!

---

## 5. Best Practice #3: Optimasi Layer Caching (Build Instan!)

Pernahkah kamu mengalami: hanya mengubah 1 huruf di kode program, tapi saat menjalankan `docker build`, Docker mengunduh ulang ratusan library dari internet selama 5 menit?
Itu terjadi karena kamu salah menyusun urutan baris di dalam Dockerfile!

### Bagaimana Mekanisme Cache Docker Bekerja?
Docker memeriksa setiap baris instruksi dari atas ke bawah:
1. Untuk instruksi `RUN`, Docker melihat apakah teks perintahnya berubah. Jika sama persis dengan build sebelumnya, Docker langsung mengambil hasil dari **Cache** (0.0 detik!).
2. Untuk instruksi `COPY`, Docker menghitung *checksum* (sidik jari isi file) dari file yang disalin.
3. **ATURAN RANTAI (Domino Effect):** Begitu **SATU lapisan mengalami perubahan** (Cache Miss), maka lapisan tersebut **DAN SELURUH LAPISAN DI BAWAHNYA otomatis hangus cache-nya** dan wajib dijalankan ulang dari awal!

### Perbandingan Urutan Buruk vs Urutan Emas

```text
 ❌ URUTAN BURUK (Lambat!)                    ✅ URUTAN EMAS SRE (Cepat!)
 ┌─────────────────────────────────┐          ┌─────────────────────────────────┐
 │ 1. FROM golang:1.22-alpine      │ (Cache)  │ 1. FROM golang:1.22-alpine      │ (Cache)
 ├─────────────────────────────────┤          ├─────────────────────────────────┤
 │ 2. WORKDIR /app                 │ (Cache)  │ 2. WORKDIR /app                 │ (Cache)
 ├─────────────────────────────────┤          ├─────────────────────────────────┤
 │ 3. COPY . .  ◄── (main.go ubah!)│ ❌ MISS! │ 3. COPY go.mod go.sum ./        │ (Cache Aman!)
 ├─────────────────────────────────┤          ├─────────────────────────────────┤
 │ 4. RUN go mod download (Unduh!) │ ❌ ULANG!│ 4. RUN go mod download          │ (Cache Aman!)
 ├─────────────────────────────────┤          ├─────────────────────────────────┤
 │ 5. RUN go build                 │ ❌ ULANG!│ 5. COPY . .  ◄── (main.go ubah!)│ ❌ MISS!
 └─────────────────────────────────┘          ├─────────────────────────────────┤
                                              │ 6. RUN go build                 │ ❌ ULANG!
                                              └─────────────────────────────────┘
```

- Pada **Urutan Buruk**, karena `COPY . .` ditaruh di baris ke-3, setiap kali kamu mengedit `main.go`, langkah ke-3 berubah. Akibatnya langkah ke-4 (`go mod download` / `npm install` / `pip install`) ikut hangus dan harus mengunduh ulang seluruh dependensi dari internet!
- Pada **Urutan Emas**, kita menyalin file daftar dependensi (`go.mod`, `package.json`, atau `requirements.txt`) terlebih dahulu, lalu mengunduh dependensinya. Selama kamu tidak menambah library baru, langkah 3 dan 4 akan selalu diambil dari Cache secara instan (0.0 detik) meskipun kamu mengedit `main.go` ribuan kali!

> 📌 **Prinsip Emas Urutan Dockerfile:**
> Letakkan instruksi yang **paling jarang berubah di bagian paling atas**, dan letakkan instruksi yang **paling sering berubah (kode sumber aplikasi) di bagian paling bawah**.

---

## 6. Best Practice #4: Audit Otomatis dengan `hadolint`

Manusia bisa lupa, tetapi linter tidak pernah lupa. Di dunia SRE, sebelum sebuah `Dockerfile` diizinkan masuk ke branch `main` di GitLab, file tersebut harus diperiksa oleh alat bernama **Hadolint (Haskell Dockerfile Linter)**.

Hadolint memeriksa Dockerfile milikmu dan bahkan memeriksa praktik terbaik perintah shell di dalam instruksi `RUN` (menggunakan ShellCheck di balik layar).

### Instalasi & Penggunaan di macOS
```bash
# 1. Instal hadolint melalui Homebrew
brew install hadolint

# 2. Jalankan pemeriksaan pada Dockerfile
hadolint Dockerfile
```

Jika Dockerfile kamu mengandung praktik buruk, `hadolint` akan memberikan peringatan lengkap dengan kode aturannya, misalnya:
- `DL3007 warning: Using latest is prone to errors if the image will ever update.` (Jangan pakai tag `:latest`).
- `DL3008 warning: Pin versions in apt get install.` (Kunci versi paket atau gunakan praktik bersih).
- `DL3015 info: Avoid additional packages by specifying '--no-install-recommends'.`
- `DL3020 error: Use COPY instead of ADD for files and folders.`

Jika `hadolint Dockerfile` tidak mengeluarkan teks apa pun (kosong) dan kembali ke prompt terminal, artinya **Dockerfile kamu telah lulus standar kualitas 100%!**

---

## 7. Hands-On Mandiri Step-by-Step (20 Menit)

Mari kita buktikan kehebatan **Multi-Stage Build** dan **Non-Root User** secara nyata. Kita akan membuat satu aplikasi web Go sederhana, lalu membandingkan hasil rakitan antara **Dockerfile Buruk** vs **Dockerfile Best Practice**.

### Langkah 1: Siapkan Kode Aplikasi & `.dockerignore`
Jalankan perintah berikut di terminal:

```bash
cd ~
mkdir lab-dockerfile-sre && cd lab-dockerfile-sre

# 1. Buat file .dockerignore
cat << 'EOF' > .dockerignore
.git
*.md
Dockerfile*
EOF

# 2. Buat file inisialisasi modul Go
cat << 'EOF' > go.mod
module sre-demo-app

go 1.22
EOF

# 3. Buat kode aplikasi web HTTP sederhana (main.go)
cat << 'EOF' > main.go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		hostname, _ := os.Hostname()
		fmt.Fprintf(w, "Halo SRE! Aplikasi berjalan aman di container: %s\n", hostname)
	})
	fmt.Printf("Server mendengarkan pada port %s...\n", port)
	if err := http.ListenAndServe(":"+port, nil); err != nil {
		panic(err)
	}
}
EOF
```

---

### Langkah 2: Buat dan Bangun "Dockerfile Buruk" (Single-Stage & Root)
```bash
cat << 'EOF' > Dockerfile.buruk
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]
EOF

# Build image versi buruk
docker build -f Dockerfile.buruk -t app-sre:buruk .
```

---

### Langkah 3: Buat dan Bangun "Dockerfile Best Practice" (Multi-Stage & Non-Root)
```bash
cat << 'EOF' > Dockerfile
# --- STAGE 1: Builder ---
FROM golang:1.22-alpine AS builder
WORKDIR /build
COPY go.mod ./
RUN go mod download
COPY main.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /build/server .

# --- STAGE 2: Production Runtime ---
FROM alpine:3.20
RUN adduser -D -u 10001 sreuser
WORKDIR /app
ENV PORT=8080
COPY --from=builder /build/server /app/server
USER 10001
EXPOSE 8080
ENTRYPOINT ["/app/server"]
EOF

# Audit kualitas Dockerfile dengan hadolint (jika sudah terinstal)
hadolint Dockerfile || true

# Build image versi terbaik
docker build -t app-sre:baik .
```

---

### Langkah 4: Bandingkan Ukuran dan Keamanan Keduanya!
Jalankan perintah ini untuk melihat perbedaan ukuran yang mencengangkan:

```bash
docker images app-sre
```

*Contoh tampilan hasil perbandingan:*
```text
REPOSITORY   TAG     IMAGE ID       CREATED          SIZE
app-sre      baik    8f91b2c3d4e5   10 seconds ago   14.8MB   ◄── Hemat ~98% ruang disk!
app-sre      buruk   1a2b3c4d5e6f   45 seconds ago   842MB    ◄── Bengkak & penuh alat kompilasi!
```

Sekarang mari kita uji jalankan image `app-sre:baik` dan verifikasi keamanannya:

```bash
# 1. Jalankan container dan hubungkan port 8080
docker run -d --name server-aman -p 8080:8080 app-sre:baik

# 2. Uji akses HTTP menggunakan curl
curl http://localhost:8080/

# 3. Verifikasi bahwa proses di dalam container berjalan sebagai UID 10001 (Bukan Root!)
docker exec server-aman id
# Output: uid=10001(sreuser) gid=10001(sreuser) groups=10001(sreuser)

# 4. Bersihkan container dan direktori latihan
docker rm -f server-aman
docker rmi app-sre:baik app-sre:buruk
cd ~ && rm -rf lab-dockerfile-sre
```

---

## 8. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 4 jebakan teknis yang sering membuat container gagal berjalan setelah di-build beserta cara mengatasinya:

### Kasus 1: Pesan Aneh `exec ./app: no such file or directory` Padahal Filenya Ada!
- **Gejala:** Kamu melakukan Multi-Stage Build. Di Stage 1 kamu memakai `FROM golang:1.22` (berbasis Debian/`glibc`), lalu di Stage 2 kamu menyalin binernya ke `FROM alpine:3.20`. Saat dijalankan, container langsung mati dengan error `exec /app/server: no such file or directory`. Ketika kamu cek isi foldernya, file `/app/server` jelas-jelas ada di sana!
- **Penyebab:** Pesan error Linux tersebut sebenarnya bukan mengatakan file `/app/server` tidak ada, melainkan **pustaka dinamis (`ld-linux.so` milik `glibc`) yang dibutuhkan oleh biner tersebut tidak ditemukan di dalam Alpine (yang menggunakan `musl libc`)!**
- **Solusi:**
  1. Saat mengompilasi aplikasi Go untuk dimasukkan ke Alpine atau Distroless, selalu matikan ketergantungan pustaka C dengan menambahkan variabel **`CGO_ENABLED=0`** pada perintah `go build`.
  2. Atau pastikan jenis keluarga OS di Stage 1 dan Stage 2 seragam (misalnya sama-sama berbasis `-alpine` atau sama-sama berbasis `debian-slim`).

---

### Kasus 2: `docker stop` Sangat Lambat (Selalu Menunggu 10 Detik Baru Mati)
- **Gejala:** Saat kamu mengetik `docker stop <nama-container>`, terminal membeku selama tepat 10 detik sebelum container akhirnya berhenti dengan kode `Exit Code 137`.
- **Penyebab:** Kamu menulis `CMD ./server` atau `ENTRYPOINT python app.py` menggunakan **Shell Form** (tanpa kurung siku JSON `["..."]`). Akibatnya `/bin/sh` menjadi PID 1 dan mengabaikan sinyal berhenti `SIGTERM`. Docker menunggu masa tenggang 10 detik, lalu terpaksa menembak mati container dengan `SIGKILL`.
- **Solusi:** Ubah baris `ENTRYPOINT` dan `CMD` di Dockerfile menjadi **Exec Form**:
  ```dockerfile
  ENTRYPOINT ["/app/server"]
  ```

---

### Kasus 3: Container Non-Root Gagal Menulis File (`Permission Denied`)
- **Gejala:** Setelah kamu menambahkan `USER 10001`, aplikasi tiba-tiba gagal menyala karena tidak bisa membuat file log atau folder cache di `/app/logs`.
- **Penyebab:** Folder `/app` dibuat oleh instruksi sebelumnya saat masih berstatus `root`, sehingga pemilik folder tersebut adalah `root:root` dan user `10001` tidak punya izin tulis.
- **Solusi:** Gunakan flag **`--chown`** saat menyalin file, atau ubah kepemilikan folder sebelum berpindah ke `USER 10001`:
  ```dockerfile
  COPY --from=builder --chown=10001:10001 /build/server /app/server
  ```

---

## 9. Ringkasan & Langkah Selanjutnya

### Checklist Standar Audit Dockerfile SRE
Setiap kali kamu menulis atau me-review sebuah Dockerfile di Merge Request, pastikan 6 daftar periksa ini terpenuhi:
- [ ] **Ada `.dockerignore`:** Mencegah `.git`, `.env`, dan file sampah masuk ke *build context*.
- [ ] **Versi Base Image Terkunci:** Tidak ada penggunaan tag `:latest`.
- [ ] **Multi-Stage Build:** Alat kompilasi dan kode mentah tidak ikut terbawa ke stage final.
- [ ] **Layer Hemat & Urut Cache:** Instalasi dependensi ditaruh sebelum `COPY . .`, dan perintah `apt`/`apk` dibersihkan dalam satu baris `RUN` yang sama.
- [ ] **Non-Root User:** Menggunakan `USER` dengan nomor UID eksplisit (misal `10001`).
- [ ] **Exec Form pada ENTRYPOINT/CMD:** Menggunakan kurung siku `["program", "arg"]` agar menerima sinyal `SIGTERM`.

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk menguji ketajaman analisis Dockerfile milikmu:

1. **Apa fungsi sebenarnya dari tanda titik (`.`) pada perintah `docker build -t app:v1 .`? Apa bahayanya jika kamu menjalankan perintah tersebut di folder yang belum memiliki file `.dockerignore`?**
2. **Jelaskan mengapa meletakkan baris `COPY . .` sebelum baris `RUN pip install -r requirements.txt` membuat proses build menjadi sangat lambat setiap kali ada perubahan kecil pada kode aplikasi!**
3. **Apa perbedaan mendasar antara menulis `ENTRYPOINT ./app` (Shell Form) dengan `ENTRYPOINT ["./app"]` (Exec Form) ketika Kubernetes mencoba menghentikan Pod secara baik-baik (*Graceful Shutdown*)?**
4. **Mengapa kita sangat disarankan menulis `USER 10001` (menggunakan angka UID) dibandingkan menulis `USER appuser` (menggunakan nama teks) untuk container yang akan dijalankan di atas cluster Kubernetes?**
5. **Bagaimana teknik Multi-Stage Build meningkatkan keamanan server production selain sekadar mengecilkan ukuran file image?**

---
*Lanjutkan ke materi berikutnya: [03-registry-gitlab.md](03-registry-gitlab.md) dan [04-multi-arch-arm64.md](04-multi-arch-arm64.md) untuk mempelajari cara mengunggah image ke GitLab Container Registry serta membangun image lintas arsitektur (`linux/arm64` & `linux/amd64`) menggunakan `docker buildx`!*