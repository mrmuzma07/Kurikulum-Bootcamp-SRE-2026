# 01 — Konsep Container: Image, Layer, Namespace & Cgroup

> **Ringkasan:** Membongkar cara kerja container dari balik layar tanpa sihir: memahami perbedaan mendasar antara Container dan Virtual Machine, membedakan Image vs Container, mempelajari cara kerja sistem file berlapis (*Layered Filesystem*), persistensi data dengan *Volume*, serta dua pilar utama Kernel Linux pembentuk container (**Namespaces** dan **Control Groups / cgroups**).

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal bersama OrbStack, kamu akan mampu:
- Menjelaskan masalah klasik pengiriman aplikasi dan mengapa teknologi container menjadi standar infrastruktur modern.
- Membedakan secara arsitektural antara **Proses Biasa**, **Container**, dan **Virtual Machine (VM)**.
- Menjelaskan perbedaan **Container Image** dan **Container Instance** menggunakan analogi yang tepat.
- Memahami cara kerja **Layered Filesystem (OverlayFS)** dan menjelaskan mengapa urutan perintah sangat memengaruhi ukuran akhir image.
- Mengelola penyimpanan data permanen di luar siklus hidup container menggunakan **Docker Volume** dan **Bind Mount**.
- Menjelaskan peran **Linux Namespaces** (isolasi pandangan) dan **Cgroups** (pembatasan sumber daya CPU/RAM), serta kaitannya dengan insiden `OOMKilled` di Kubernetes.

---

## 1. Konsep & Arsitektur: Mengapa Dunia Butuh Container?

### Masalah Klasik: *"Tapi di Laptop Saya Jalan, Kok!"*
Sebelum era container, menjalankan aplikasi dari laptop programmer ke server production adalah mimpi buruk bagi tim operasional:
1. Programmer menulis aplikasi Python di macOS menggunakan Python versi 3.12 dan pustaka (*library*) `libssl` versi terbaru.
2. Saat kode dikirim ke server production yang menggunakan Ubuntu 22.04, ternyata server tersebut menggunakan Python 3.10 dan `libssl` versi lama.
3. Akibatnya: **Aplikasi gagal menyala di server production!** Muncul kalimat legendaris: *"It works on my machine!"* (Di laptop saya jalan kok!).

### Analogi Kontainer Kapal Laut (Shipping Container)
Sebelum tahun 1950-an, mengirim barang lintas benua lewat kapal laut sangat lambat dan kacau. Beras dalam karung, anggur dalam tong kayu, dan suku cadang mesin ditumpuk bercampur aduk. Saat dipindahkan dari kapal ke kereta api, kuli pelabuhan harus membongkar-muat barang satu per satu. Banyak barang rusak atau hilang.

Dunia logistik berubah total ketika ditemukan **Kotak Kontainer Besi Standar (Intermodal Shipping Container)**:
- Tidak peduli apakah isinya mobil mewah, buah pisang, atau pakaian—semuanya dimasukkan ke dalam **kotak besi dengan ukuran dimensi dan kunci pengait yang 100% seragam**.
- Derek (*crane*) di pelabuhan Singapura, truk di Jakarta, maupun kapal kargo di Rotterdam dapat mengangkut kotak tersebut tanpa perlu peduli apa isi di dalamnya.

**Container di dunia IT bekerja dengan filosofi yang sama persis:**
Kita membungkus **kode aplikasi + runtime bahasa pemrograman + library sistem + file konfigurasi** ke dalam satu paket standar yang terisolasi. Paket ini dijamin berjalan identik, baik di laptop MacBook Air M5 milikmu (via OrbStack), di server pengujian CI/CD, maupun di ratusan server bare-metal data center production!

---

## 2. Membongkar Mitos: Container BUKAN Virtual Machine!

Kesalahpahaman terbesar bagi pemula adalah menganggap bahwa *"Container itu adalah Virtual Machine (VM) versi ringan"*. Secara arsitektur sistem operasi, pernyataan tersebut **kurang tepat**.

Untuk memahaminya, kita perlu mengingat bahwa sistem operasi Linux terdiri dari dua bagian utama:
1. **Kernel (Otak Inti OS):** Bagian paling bawah yang bertugas berbicara langsung dengan perangkat keras (mengatur CPU, alokasi RAM, baca-tulis disk, dan kartu jaringan).
2. **User Space (Ruang Pengguna & Aplikasi):** File sistem, pustaka (*libraries*), perintah terminal (`ls`, `bash`, `curl`), dan program aplikasi yang berjalan di atas Kernel.

### Perbandingan Arsitektur: VM vs Container

```text
       VIRTUAL MACHINE (VM)                           CONTAINER
 ┌──────────────┐ ┌──────────────┐         ┌──────────────┐ ┌──────────────┐
 │  Aplikasi A  │ │  Aplikasi B  │         │  Aplikasi A  │ │  Aplikasi B  │
 ├──────────────┤ ├──────────────┤         ├──────────────┤ ├──────────────┤
 │  Bin / Libs  │ │  Bin / Libs  │         │  Bin / Libs  │ │  Bin / Libs  │
 ├──────────────┤ ├──────────────┤         └──────┬───────┘ └───────┬──────┘
 │ Guest Kernel │ │ Guest Kernel │  ◄── Berat!    │   (Isolasi:     │
 │ (OS Lengkap) │ │ (OS Lengkap) │                │   Namespaces    │
 └──────┬───────┘ └───────┬──────┘                │   & Cgroups)    │
        ▼                 ▼                       ▼                 ▼
 ┌───────────────────────────────┐         ┌───────────────────────────────┐
 │          Hypervisor           │         │       Docker / Containerd     │
 ├───────────────────────────────┤         ├───────────────────────────────┤
 │          Host Kernel          │         │   Satu Host Kernel Bersama    │
 ├───────────────────────────────┤         ├───────────────────────────────┤
 │     Perangkat Keras Fisik     │         │     Perangkat Keras Fisik     │
 └───────────────────────────────┘         └───────────────────────────────┘
```

> 🏢 **Analogi Rumah Tapak (VM) vs Unit Apartemen (Container):**
> - **Virtual Machine (Rumah Tapak Mandiri):** Setiap aplikasi dibangunkan satu rumah utuh dari nol. Punya pondasi tanah sendiri, sumur bor sendiri, dan genset listrik sendiri (**Guest Kernel OS lengkap**). Sangat terisolasi, tetapi butuh waktu lama untuk dibangun (menit) dan memakan lahan serta biaya perawatan yang sangat boros (RAM bergiga-gigabyte hanya untuk menyalakan OS tambahan).
> - **Container (Kamar Apartemen):** Semua penghuni tinggal di satu gedung yang sama dan berbagi satu pondasi, satu saluran air utama, serta satu gardu listrik utama (**Satu Kernel Linux Host Bersama**). Namun, tiap penghuni diberikan **dinding penyekat dan kunci pintu pribadi (Namespaces)** agar tidak bisa mengintip kamar sebelah, serta **meteran pembatas kuota listrik/air masing-masing (Cgroups)** agar satu kamar tidak menguras habis listrik satu gedung. Masuk ke unit apartemen hanya butuh waktu **hitungan milidetik**!

### Hakekat Sejati Container
Jadi, apa sebenarnya container itu?
**Container hanyalah sebuah proses Linux biasa yang berjalan di atas Host**, namun "dibohongi" dan "dibatasi" oleh Kernel menggunakan dua fitur bawaan Linux:
1. **Namespaces:** Membuat proses tersebut merasa seolah-olah ia memiliki komputer, sistem file, dan jaringan sendiri.
2. **Control Groups (cgroups):** Membatasi seberapa banyak CPU dan Memori (RAM) yang boleh dikonsumsi oleh proses tersebut.

### Pembuktian Nyata di Terminal
Mari kita buktikan bahwa proses di dalam container sebenarnya hanyalah proses biasa di mata sistem operasi utama:

```bash
# 1. Jalankan sebuah container Alpine Linux di latar belakang (-d) yang hanya tidur selama 3600 detik
docker run -d --name bukti-proses alpine sleep 3600

# 2. Lihat daftar proses DARI DALAM container:
docker exec bukti-proses ps aux
```

*Output dari dalam container:*
```text
PID   USER     TIME  COMMAND
    1 root      0:00 sleep 3600
    7 root      0:00 ps aux
```
> Perhatikan! Di dalam ruangannya sendiri, perintah `sleep 3600` melihat dirinya sebagai **PID 1** (proses pertama dan satu-satunya penguasa di sistem itu).

Sekarang, jika kita mengintip dari mesin Linux utama (atau melalui `orb shell` di OrbStack), proses `sleep 3600` tersebut sebenarnya sedang berjalan berdampingan dengan proses-proses lain dengan nomor PID asli (misalnya PID `14892`)!

```bash
# Bersihkan container percobaan
docker rm -f bukti-proses
```

> 💻 **Catatan Khusus Pengguna Mac (Apple Silicon & OrbStack):**
> Container berbasis Linux membutuhkan **Kernel Linux** untuk berjalan, sedangkan laptop Mac milikmu menggunakan **Kernel XNU/Darwin (macOS)**. Itulah sebabnya aplikasi seperti **OrbStack** secara otomatis menjalankan satu mesin virtual Linux yang super ringan dan transparan di latar belakang. Seluruh container Docker yang kamu jalankan di Mac sebenarnya berbagi satu Kernel Linux milik OrbStack tersebut!

---

## 3. Membedakan Image vs Container

Dalam percakapan sehari-hari, pemula sering mencampuradukkan istilah **Image** dan **Container**. Padahal keduanya adalah dua wujud yang berbeda.

```text
 ┌─────────────────────────┐          docker run          ┌─────────────────────────┐
 │                         │ ───────────────────────────► │   Container 1 (Hidup)   │
 │     CONTAINER IMAGE     │                              └─────────────────────────┘
 │                         │          docker run          ┌─────────────────────────┐
 │   (Paket Cetak Biru /   │ ───────────────────────────► │   Container 2 (Hidup)   │
 │    Template Statis /    │                              └─────────────────────────┘
 │       Read-Only)        │          docker run          ┌─────────────────────────┐
 │                         │ ───────────────────────────► │   Container 3 (Hidup)   │
 └─────────────────────────┘                              └─────────────────────────┘
```

### Tabel Perbandingan Image vs Container

| Aspek | Container Image | Container (Instance) |
|---|---|---|
| **Sifat Dasar** | Berkas cetakan statis, **Read-Only** (tidak bisa diubah). | Proses aktif yang sedang hidup/berjalan (**Read-Write**). |
| **Analogi Dunia Nyata** | Resep masakan / Cetakan kue / Stempel karet. | Hidangan siap santap / Kue hasil cetakan / Cap di kertas. |
| **Analogi Pemrograman**| *Class* (Kelas dalam OOP). | *Object* (Objek hasil instansiasi kelas). |
| **Apakah Memakai RAM?**| Tidak (hanya memakan ruang simpan di harddisk). | Ya (memakai CPU dan RAM selama berjalan). |
| **Cara Membuatnya** | `docker build` (dari Dockerfile) atau `docker pull`. | `docker run` (mencetak dari sebuah Image). |

Satu buah **Image** (misalnya `nginx:1.27-alpine`) dapat digunakan untuk menyalakan 1, 10, hingga 1.000 **Container** yang berjalan secara paralel tanpa mengubah atau merusak file Image aslinya sedikit pun.

```bash
# Melihat daftar Image (cetakan) yang tersimpan di disk lokalmu:
docker images

# Melihat daftar Container yang SEDANG berjalan saat ini:
docker ps

# Melihat SEMUA Container (termasuk yang sudah berhenti/mati):
docker ps -a
```

---

## 4. Arsitektur Layer Filesystem: Mengapa Urutan Sangat Penting?

Bagaimana sebuah Container Image disimpan di dalam harddisk?
Image tidak disimpan sebagai satu file raksasa yang utuh, melainkan disusun dari **tumpukan lapisan-lapisan tipis (Layers)** yang bersifat *Read-Only* menggunakan teknologi **Union File System (OverlayFS)**.

### Visualisasi Kue Lapis Image
Setiap baris instruksi di dalam `Dockerfile` (`FROM`, `RUN`, `COPY`) akan menciptakan **satu lapisan baru** di atas lapisan sebelumnya:

```text
Image "aplikasi-web:v1"
┌────────────────────────────────────────────────────────┐
│ Layer 4: CMD ["./server"]          (0 B - Metadata)    │ ◄── Lapisan Teratas
├────────────────────────────────────────────────────────┤
│ Layer 3: COPY ./server /app/server (15 MB)             │
├────────────────────────────────────────────────────────┤
│ Layer 2: RUN apk add --no-cache tzdata (2 MB)          │
├────────────────────────────────────────────────────────┤
│ Layer 1: FROM alpine:3.20          (7.8 MB)            │ ◄── Base Image (Pondasi)
└────────────────────────────────────────────────────────┘
```

### Keuntungan Arsitektur Berlapis: *Layer Sharing* (Hemat Disk & Kuota!)
Bayangkan kamu memiliki 20 aplikasi mikro (*microservices*) yang semuanya dibangun di atas pondasi `ubuntu:24.04` (ukuran ~78 MB).
Apakah harddisk laptopmu akan tersedot `20 x 78 MB = 1.560 MB`? **TIDAK!**
Docker cukup cerdas untuk menyimpan lapisan `ubuntu:24.04` **hanya satu kali saja** di dalam disk. Ke-20 image aplikasimu cukup menunjuk (berbagi referensi) ke lapisan dasar yang sama tersebut.

---

### Jebakan Pemula: Mengapa Menghapus File di Lapisan Baru Tidak Mengecilkan Image?

Karena setiap lapisan Image bersifat **Read-Only (Tidak Bisa Diubah setelah jadi)**, perhatikan apa yang terjadi jika kamu menulis instruksi seperti ini:

```dockerfile
# ❌ CONTOH BURUK (Ukuran akhir Image membengkak!)
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y curl git build-essential   # Layer 2 (+350 MB)
RUN rm -rf /var/lib/apt/lists/*                                     # Layer 3 (0 MB)
```

> 🎨 **Analogi Kertas Transparan & Tipe-X:**
> Bayangkan setiap layer adalah lembaran plastik transparan yang ditumpuk:
> - Di **Lembar ke-2**, kamu menempelkan pemberat timah seberat **350 gram** (`apt-get install`). Begitu selesai, Lembar ke-2 langsung dilaminating permanen (*Read-Only*).
> - Di **Lembar ke-3**, kamu menjalankan perintah hapus (`rm -rf`). Karena Lembar ke-2 sudah dilaminating, kamu tidak bisa mengambil timah di bawah sana! Yang dilakukan Docker di Lembar ke-3 hanyalah menaruh stiker penutup warna putih (*whiteout file*) agar file tersebut **tidak terlihat dari atas**.
> - Namun saat ditimbang, pemberat 350 gram di Lembar ke-2 **masih tetap ada di dalam tumpukan image tersebut**!

**Solusi SRE Best Practice:**
Lakukan pengunduhan, instalasi, dan pembersihan sampah cache **di dalam SATU instruksi `RUN` yang sama** (digabungkan dengan operator `&&`) sebelum lapisan tersebut dikunci (*committed*):

```dockerfile
# ✅ CONTOH BAIK (Bersih dalam 1 Layer tunggal)
FROM ubuntu:24.04
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

Untuk memeriksa lapisan dan ukuran tiap langkah pada sebuah image, gunakan:
```bash
docker history alpine:latest
```

---

## 5. Writable Layer & Persistensi Data (Volumes vs Bind Mounts)

### Sifat Ephemeral (Sementara) pada Container
Jika semua lapisan di dalam Image bersifat *Read-Only*, lalu ke mana perginya file log atau data baru yang ditulis oleh aplikasi saat container sedang berjalan?

```text
┌────────────────────────────────────────────────────────┐
│   WRITABLE LAYER (Container Layer - Read/Write Tipis)  │ ◄── Hilang saat 'docker rm'!
├════════════════════════════════════════════════════════┤
│   Layer 3: COPY app /app                  (Read-Only)  │
│   Layer 2: RUN apk add ...                (Read-Only)  │
│   Layer 1: FROM alpine:3.20               (Read-Only)  │
└────────────────────────────────────────────────────────┘
```

Saat kamu menjalankan `docker run`, Docker meletakkan satu lapisan tipis bernama **Writable Layer (Container Layer)** di puncak paling atas.
- Semua perubahan file, pembuatan file baru di `/tmp`, atau modifikasi konfigurasi terjadi di lapisan paling atas ini.
- **BAHAYA:** Lapisan Writable ini terikat mati pada nyawa container tersebut. Begitu container dihapus dengan `docker rm`, **seluruh data di lapisan Writable tersebut ikut musnah selamanya!**

Mari buktikan sifat *ephemeral* ini di terminal:
```bash
# 1. Buat container pertama dan tulis sebuah file rahasia di dalamnya
docker run --name uji-hilang alpine sh -c 'echo "Data Penting Pelanggan" > /data-pelanggan.txt'

# 2. Pastikan filenya memang ada di container tersebut
docker start -a uji-hilang
# (Container sudah berhenti, sekarang kita hapus container-nya)
docker rm uji-hilang

# 3. Nyalakan container baru dari image alpine yang sama dan cari file tadi:
docker run --rm alpine cat /data-pelanggan.txt
# Output: cat: can't open '/data-pelanggan.txt': No such file or directory
```

---

### Solusi Penyimpanan Abadi: Volume & Bind Mount
Agar data penting (seperti isi database PostgreSQL, MySQL, atau file unggahan pengguna) tetap selamat meskipun container dihapus dan diganti versi baru, kita harus menyimpannya **di luar lapisan container** menggunakan mekanisme *Mounting*:

```text
  Host / Mesin Utama (OrbStack)                 Dalam Container
 ┌─────────────────────────────┐               ┌─────────────────────┐
 │  /var/lib/docker/volumes/.. │ ◄───────────► │  /var/lib/postgresql│
 │      (Docker Volume)        │   Tersambung  │     (Folder Data)   │
 └─────────────────────────────┘               └─────────────────────┘
```

Ada 2 jenis penyimpanan persisten utama:

#### 1. Docker Volume (Direkomendasikan untuk Production & Database)
Dikelola sepenuhnya oleh Docker di area penyimpanan khusus yang terisolasi dan aman.
```bash
# 1. Buat brankas volume baru
docker volume create brankas-db

# 2. Jalankan container pertama dengan mencantolkan (-v) volume tersebut ke folder /data
docker run --rm -v brankas-db:/data alpine sh -c 'echo "Transaksi Rp 100.000" > /data/rekening.txt'
# (Perhatikan flag --rm langsung menghapus container begitu selesai!)

# 3. Jalankan container KEDUA yang sama sekali baru, pasang volume yang sama, dan baca isinya:
docker run --rm -v brankas-db:/data alpine cat /data/rekening.txt
# Output: Transaksi Rp 100.000 (Data tetap hidup dan utuh!)
```

#### 2. Bind Mount (Cocok untuk Development Lokal / Konfigurasi)
Menghubungkan folder fisik tertentu di laptop Mac milikmu langsung ke dalam folder container secara *real-time*.
```bash
# Memetakan folder kerja saat ini ($PWD) ke dalam folder /workspace di dalam container
docker run --rm -v "$PWD:/workspace" alpine ls -la /workspace
```

---

## 6. Dua Pilar Inti Kernel Linux: Namespaces & Cgroups

Mengapa proses di dalam container tidak bisa mengganggu proses lain di host? Mari kita bedah dua komponen kernel yang bekerja di balik layar.

### Pilar 1: Namespaces — *"Apa yang Boleh Kamu Lihat"* (Isolasi Pandangan)
*Namespace* membungkus sumber daya sistem global sehingga proses di dalam namespace tersebut merasa memiliki sistem operasi eksklusif miliknya sendiri.

Linux menyediakan beberapa jenis Namespace utama untuk setiap container:

| Jenis Namespace | Apa yang Diisolasi? | Efek Nyata di Dalam Container |
|---|---|---|
| **`pid`** (Process ID) | Penomoran urut proses. | Container punya proses nomor **PID 1** sendiri dan tidak bisa melihat proses milik host atau container tetangga. |
| **`net`** (Network) | Kartu jaringan, alamat IP, tabel routing, dan nomor port. | Dua container berbeda bisa sama-sama memakai **Port 80** di waktu bersamaan tanpa bentrok karena punya IP internal sendiri-sendiri! |
| **`mnt`** (Mount) | Titik kait sistem direktori (*root filesystem*). | Container melihat folder `/` miliknya sendiri (misal Alpine), terpisah dari folder `/` milik host utama. |
| **`uts`** (UNIX Time-Sharing)| Nama komputer (*hostname*) dan domain. | Setiap container memiliki `hostname` unik sendiri (biasanya berupa ID pendek container). |
| **`ipc`** (Inter-Process Comm)| Memori bersama (*shared memory*). | Komunikasi memori kecepatan tinggi hanya bisa diakses oleh proses dalam grup yang sama. |
| **`user`** (User ID) | Pemetaan nomor identitas pengguna (UID/GID). | Memungkinkan pengguna di dalam container dipetakan ke pengguna tanpa hak akses tinggi di sisi host. |

> ☸️ **Hubungan Sangat Penting dengan Kubernetes (Fase 2):**
> Kelak saat kamu belajar **Kubernetes**, unit terkecil bukanlah Container, melainkan **Pod**.
> Apa itu Pod sebenarnya? Pod adalah **sekumpulan container yang sengaja diatur untuk berbagi Namespace `net` dan `ipc` yang sama!** Itulah sebabnya dua container di dalam satu Pod yang sama memiliki **alamat IP yang sama persis** dan dapat berkomunikasi satu sama lain cukup melalui `localhost`.

---

### Pilar 2: Control Groups (cgroups) — *"Seberapa Banyak yang Boleh Kamu Makan"* (Jatah Kuota)
Isolasi pandangan (*Namespace*) saja tidak cukup!
Bayangkan penghuni kamar apartemen sebelah tidak bisa masuk ke kamarmu, tetapi ia menyalakan 50 mesin las listrik sekaligus sehingga sekring satu gedung meledak dan kamarmu ikut gelap gulita. Di dunia server, fenomena ini disebut **Noisy Neighbor (Tetangga yang Rakus)**.

Untuk mencegahnya, Kernel Linux menggunakan **cgroups** guna memasang batas plafon pemakaian perangkat keras pada setiap container:

| Sumber Daya | Flag Docker | Apa yang Terjadi Jika Melewati Batas? |
|---|---|---|
| **Memori (RAM)** | `--memory="256m"` | Jika aplikasi memaksa meminta RAM melebihi 256 MB, Kernel Linux akan langsung membunuh container tersebut (**`OOMKilled` — Exit Code 137**). |
| **Prosesor (CPU)** | `--cpus="1.5"` | Aplikasi **tidak akan dibunuh**, melainkan kecepatannya direm/dicekik (*CPU Throttling*) agar tidak melebihi jatah 1,5 core CPU. |
| **Jumlah Proses** | `--pids-limit=100` | Mencegah serangan atau bug *fork bomb* (aplikasi beranak-pinak membuat ribuan proses hingga server lumpuh). |

#### Simulasi Nyata Insiden `OOMKilled` di Terminal
Sebagai seorang SRE, kamu akan sangat sering bertemu dengan status `OOMKilled` di lingkungan production. Mari kita simulasikan bagaimana Kernel mengeksekusi hukuman mati pada container yang melanggar batas memori:

```bash
# 1. Jalankan container dengan batas RAM super ketat (hanya 20 Megabyte),
#    namun perintahkan ia membuat beban di memori sebesar 50 Megabyte!
docker run --name simulasi-oom --memory="20m" --memory-swap="20m" alpine \
  sh -c 'dd if=/dev/zero of=/dev/shm/bongkahan bs=1M count=50'

# 2. Periksa status kematian container tersebut menggunakan docker inspect:
docker inspect simulasi-oom --format 'Status={{.State.Status}} | ExitCode={{.State.ExitCode}} | OOMKilled={{.State.OOMKilled}}'
```

*Output yang muncul:*
```text
Status=exited | ExitCode=137 | OOMKilled=true
```

> 🔍 **Rumus Matematika SRE: Mengapa Exit Code 137?**
> Di Linux, ketika sebuah proses dihentikan paksa oleh sinyal sistem, kode keluarnya dihitung dengan rumus: **`128 + Nomor Sinyal`**.
> Sinyal pembunuhan paksa oleh kernel (`SIGKILL`) memiliki nomor **9**.
> Maka: **`128 + 9 = 137`**. Setiap kali kamu melihat **Exit Code 137** di Docker maupun Kubernetes, tuduhan pertamamu adalah: **Aplikasi kehabisan memori (OOMKilled) atau dimatikan paksa oleh sistem!**

```bash
# Bersihkan container simulasi
docker rm simulasi-oom
```

---

## 7. Ringkasan Perbandingan: Proses Biasa vs Container vs VM

| Aspek Penilaian | Proses Linux Biasa | Container | Virtual Machine (VM) |
|---|---|---|---|
| **Kernel yang Dipakai** | Berbagi Kernel Host | Berbagi Kernel Host | **Punya Kernel OS Sendiri** |
| **Batas Isolasi** | Sangat minim (tergantung permission user) | Sedang–Tinggi (Namespaces & Cgroups) | **Sangat Tinggi** (Isolasi perangkat keras/Hypervisor) |
| **Waktu Menyala (Boot)** | Instan (milidetik) | **Instan (milidetik)** | Lambat ( puluhan detik – menit) |
| **Beban Memori (Overhead)**| Nol | **Sangat ringan (beberapa MB)** | Berat (Ratusan MB hingga GB per VM) |
| **Portabilitas (Kemudahan Pindah)** | Buruk (sering gagal jika pindah server karena beda library) | **Sangat Tinggi (Image berjalan identik di mana saja)** | Tinggi, namun ukuran file image sangat besar ( puluhan GB) |

---

## 8. Hands-On Mandiri Step-by-Step (15 Menit)

Jalankan rangkaian perintah berikut di terminal Mac milikmu (pastikan aplikasi **OrbStack** sudah berjalan) untuk memvalidasi seluruh konsep yang telah dipelajari:

```bash
# --- LATIHAN 1: Siklus Hidup Image & Container ---
# Unduh image resmi alpine dari registry
docker pull alpine:3.20

# Jalankan container bernama 'kotak-pertama' di background
docker run -d --name kotak-pertama alpine:3.20 sleep 3600

# Bandingkan hostname komputer Mac milikmu dengan hostname di dalam container (Isolasi UTS Namespace)
hostname
docker exec kotak-pertama hostname

# --- LATIHAN 2: Membuktikan Isolasi PID Namespace ---
# Lihat nomor PID proses sleep dari dalam container (pasti bernomor PID 1)
docker exec kotak-pertama ps aux

# Lihat nomor PID asli proses tersebut di mesin Linux OrbStack
docker inspect -f 'PID Asli di Kernel Host: {{.State.Pid}}' kotak-pertama

# Hapus container pertama
docker rm -f kotak-pertama

# --- LATIHAN 3: Menginspeksi Lapisan (Layers) Image ---
docker pull nginx:1.27-alpine
docker history nginx:1.27-alpine

# --- LATIHAN 4: Membuktikan Persistensi Docker Volume ---
docker volume create data-latihan
docker run --rm -v data-latihan:/penyimpanan alpine:3.20 sh -c 'date > /penyimpanan/waktu.txt'
docker run --rm -v data-latihan:/penyimpanan alpine:3.20 cat /penyimpanan/waktu.txt

# Bersihkan volume latihan
docker volume rm data-latihan

# --- LATIHAN 5: Menguji Batas Cgroup ---
docker run -d --name kotak-hemat --memory="64m" --cpus="0.5" alpine:3.20 sleep 60
docker stats --no-stream kotak-hemat
docker rm -f kotak-hemat
```

---

## 9. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 4 masalah operasional yang paling sering dialami pemula saat bekerja dengan container beserta panduan diagnosanya:

### Kasus 1: Container Langsung Mati Seketika (*Exited (0)*) Saat Dijalankan
- **Gejala:** Kamu mengetik `docker run -d --name serverku ubuntu:24.04`, namun saat dicek dengan `docker ps`, container tersebut tidak ada! Saat dicek dengan `docker ps -a`, statusnya `Exited (0) 2 seconds ago`.
- **Penyebab:** Ingat konsep dasar: **Container bukanlah mesin virtual yang terus menyala jika menganggur!** Container hanya hidup selama **proses utama (PID 1)** di dalamnya masih berjalan. Image `ubuntu:24.04` memiliki perintah default `/bin/bash`. Jika dijalankan di background (`-d`) tanpa terminal interaktif (`-it`), program `bash` tidak menerima input apa pun, sehingga ia langsung menutup dirinya sendiri dengan sukses (`Exit Code 0`).
- **Solusi:**
  - Jika ingin masuk secara interaktif ke dalam shell container: gunakan flag `-it` (`docker run -it ubuntu:24.04 bash`).
  - Jika ingin container berjalan di background, pastikan perintah utamanya adalah layanan yang berjalan terus-menerus di *foreground* (seperti web server Nginx, aplikasi Go, atau `sleep infinity`).

---

### Kasus 2: Kehilangan Seluruh Data Database Saat Upgrade Container
- **Gejala:** Kamu menjalankan container PostgreSQL tanpa flag `-v`. Setelah 1 minggu berisi data, kamu menghapus container lama dan menjalankan image PostgreSQL versi baru. Seluruh tabel dan data hilang bersih.
- **Penyebab:** Data ditulis di *Writable Layer* milik container lama yang ikut terhapus saat `docker rm`.
- **Pencegahan SRE:** Untuk aplikasi *stateful* (database, message queue, penyimpanan objek), **wajib selalu memasang Volume eksternal** sebelum digunakan dan memvalidasinya menggunakan `docker inspect <nama-container> --format '{{json .Mounts}}'`.

---

### Kasus 3: Kapasitas Disk Laptop/Server Penuh Akibat Sampah Docker
- **Gejala:** Harddisk tiba-tiba penuh padahal hanya sedikit container yang aktif berjalan.
- **Penyebab:** Tumpukan container yang sudah mati (`stopped`), image versi lama yang tidak terpakai (*dangling images*), build cache, dan volume yatim piatu.
- **Diagnosa & Solusi:**
  ```bash
  # 1. Periksa rincian penggunaan ruang penyimpanan oleh Docker:
  docker system df

  # 2. Bersihkan seluruh container mati, jaringan tak terpakai, dan cache build:
  docker system prune -f
  ```
  > ⚠️ **Hati-hati:** Jangan menjalankan `docker volume prune` di server bersama kecuali kamu yakin tidak ada volume database penting yang sedang dalam posisi container mati sementara!

---

## 10. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Perintah Harian (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Unduh image dari registry** | `docker pull <nama-image>:<tag>` |
| **Daftar image di komputer lokal** | `docker images` |
| **Jalankan container di background** | `docker run -d --name <nama> <image>` |
| **Jalankan container interaktif (masuk shell)** | `docker run -it --rm <image> sh` |
| **Daftar container yang sedang aktif** | `docker ps` |
| **Daftar seluruh container (aktif + mati)** | `docker ps -a` |
| **Masuk ke container yang sedang jalan** | `docker exec -it <nama-container> sh` |
| **Lihat penggunaan CPU & RAM real-time** | `docker stats` |
| **Inspeksi detail konfigurasi & OOM status** | `docker inspect <nama-container>` |
| **Lihat rincian lapisan (layers) image** | `docker history <nama-image>` |
| **Cek pemakaian disk oleh Docker** | `docker system df` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk memastikan kamu telah menguasai fondasi container sebelum menulis Dockerfile:

1. **Mengapa menyalakan sebuah Container hanya membutuhkan waktu beberapa milidetik, sedangkan menyalakan sebuah Virtual Machine membutuhkan waktu puluhan detik hingga menit? Komponen besar apa yang tidak perlu dinyalakan ulang oleh Container?**
2. **Di dalam sebuah Dockerfile, seorang developer menulis `RUN wget file-besar.zip` pada baris ke-3, lalu menulis `RUN rm file-besar.zip` pada baris ke-5. Mengapa ukuran akhir Container Image tetap besar seolah-olah file tersebut belum dihapus?**
3. **Apa yang terjadi pada file yang disimpan di dalam folder biasa pada container saat perintah `docker rm` dijalankan? Bagaimana cara menyimpan data agar kebal terhadap penghapusan container?**
4. **Jelaskan perbedaan fungsi antara Linux *Namespaces* dan *Control Groups (cgroups)* dengan bahasamu sendiri!**
5. **Saat melakukan investigasi gangguan aplikasi, kamu menemukan sebuah container berhenti dengan status `Exit Code 137`. Apa arti angka 137 tersebut dan komponen kernel mana yang baru saja bertindak?**

---
*Lanjutkan ke materi berikutnya: [02-dockerfile-best-practice.md](02-dockerfile-best-practice.md) untuk mempelajari cara meracik Container Image sendiri yang berukuran super ramping, aman (non-root), dan cepat menggunakan teknik Multi-Stage Build!*