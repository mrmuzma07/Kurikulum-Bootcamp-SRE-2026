# 02 — Manajemen Resource Limit & Perbandingan OrbStack vs Docker Desktop

> **Ringkasan:** Memahami cara kerja alokasi sumber daya dinamis (*Dynamic Memory & CPU Allocation*) pada **OrbStack**, menerapkan strategi pembatasan berlapis (*Global Limit* vs *Per-Container Limit*) agar laptop MacBook tetap dingin dan responsif saat menjalankan infrastruktur berat, mengelola kebersihan ruang penyimpanan SSD, serta membedah alasan teknis mengapa **OrbStack** jauh lebih unggul dibandingkan **Docker Desktop** di ekosistem Apple Silicon.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal MacBook, kamu akan mampu:
- Menjelaskan mengapa pembatasan sumber daya (*Resource Limiting*) wajib diterapkan sebelum menjalankan laboratorium Kubernetes dan Observability.
- Memahami konsep **Dynamic Memory Allocation** pada OrbStack dan perbedaan antara pemakaian memori aktual vs **Batas Plafon Maksimal (*Hard Ceiling*)**.
- Mengonfigurasi batas memori dan CPU global OrbStack baik melalui antarmuka grafis maupun baris perintah (`orb config`).
- Menerapkan hierarki pembatasan 3 lapis (Host Mac ➔ Global OrbStack ➔ Individual Container/Pod).
- Menjelaskan 5 keunggulan arsitektural **OrbStack** dibandingkan **Docker Desktop** (RAM idle, kecepatan startup, VirtioFS, jaringan IP langsung, dan fitur Linux Machine).
- Memantau pemakaian CPU/RAM secara *real-time* serta melakukan pembersihan ruang penyimpanan SSD (`docker system df` & `prune`) dengan aman.

---

## 1. Konsep & Arsitektur: Mengapa Laptop SRE Wajib Diberi Batas Kuota?

### Gambaran Beban Kerja Bootcamp di Fase Mendatang
Saat ini di Fase 1, kamu mungkin baru menjalankan 1 atau 2 container kecil yang hanya memakan memori puluhan Megabyte.
Namun, mulai **Fase 2 hingga Fase 9** nanti, laptop MacBook milikmu akan disulap menjadi miniatur data center yang menjalankan puluhan komponen secara bersamaan:
- 3 buah mesin virtual Linux untuk cluster **Kubernetes (k3s Multi-Node)**.
- Controller GitOps **ArgoCD**.
- Tumpukan penuh mesin pemantau **Observability** (Prometheus, Grafana, Alloy, Mimir, Loki, dan Tempo).

Jika salah satu komponen di atas mengalami kebocoran memori (*memory leak*) atau kesalahan konfigurasi, dan kamu tidak memasang "sekring pembatas", maka proses tersebut akan menyedot habis seluruh RAM laptopmu hingga **macOS membeku total (*hang/freeze*)** dan memaksamu menekan tombol power secara paksa!

> ⚡ **Analogi Sekring Listrik Rumah Tangga:**
> Bayangkan rumahmu memiliki daya listrik total **2.200 Watt** (ini adalah total RAM fisik Mac milikmu, misalnya **16 GB**).
> Di dalam rumah induk, kamu harus menyalakan kulkas, lampu ruang tamu, dan AC kamar tidur (**Sistem Operasi macOS, Browser Safari/Chrome, dan VS Code**).
> Lalu kamu membangun sebuah bengkel kerja di garasi samping (**OrbStack**). Agar mesin las di garasi tidak membuat listrik satu rumah induk mati total (jepret) saat terjadi korsleting, kamu wajib memasang **MCB / Sekring Pembatas Khusus Garasi** maksimal **1.000 Watt** (misalnya **8 GB – 10 GB RAM** untuk OrbStack)!

---

## 2. Keajaiban Alokasi Memori Dinamis (Dynamic Memory Allocation)

Sebelum mengatur batas memori, penting bagi pemula untuk memahami bagaimana cara OrbStack mengelola RAM dibandingkan aplikasi virtualisasi generasi lama (seperti VirtualBox atau Docker Desktop lawas).

### Cara Lama (Static / Pre-Allocated Memory) — ❌ Boros!
Pada aplikasi virtualisasi kuno, jika kamu mengatur *Memory Limit* sebesar **8 GB**, maka detik itu juga aplikasi tersebut langsung **merampas dan mengunci 8 GB RAM dari macOS**, meskipun di dalamnya belum ada satu pun container yang berjalan! Laptopmu langsung terasa berat sejak aplikasi dinyalakan.

### Cara Modern OrbStack (Dynamic Memory Allocation) — ✅ Cerdas & Hemat!
OrbStack menggunakan teknologi pengelolaan memori dinamis dua arah yang bekerja sangat harmonis dengan macOS:

```text
 Plafon Maksimal (Memory Limit OrbStack = 8 GB)
 ══════════════════════════════════════════════════════════════════ ◄── Batas Sekring!
                                      ▲
                                      │ (Masih ada ruang aman)
                              ┌───────┴───────┐
                              │  Dipakai 4 GB │
 ┌───────────────────┐        │ (Saat cluster │        ┌───────────────────┐
 │  Idle (~200 MB)   │ ─────► │  k3s menyala) │ ─────► │  Turun (~250 MB)  │
 │  Saat baru buka   │  Naik  │               │  Turun │  Saat cluster     │
 │     OrbStack      │        └───────────────┘        │   dimatikan!      │
 └───────────────────┘                                 └───────────────────┘
```

1. **Saat Menganggur (Idle):** Ketika OrbStack baru menyala tanpa beban berat, ia hanya memakai sekitar **150 MB – 300 MB RAM**. Sisa RAM tetap bebas digunakan oleh macOS.
2. **Saat Beban Naik:** Ketika kamu menyalakan cluster Kubernetes dan database, OrbStack meminjam RAM dari macOS sesuai kebutuhan nyata (misal naik ke 4 GB).
3. **Saat Beban Dimatikan:** Begitu kamu menghentikan container atau cluster tersebut, OrbStack **otomatis mengembalikan RAM yang tidak terpakai kembali ke macOS**!
4. **Fungsi Angka "Memory Limit":** Angka batas yang kita atur (misalnya 8 GB) bertindak sebagai **Plafon Maksimal (*Hard Ceiling*)**. OrbStack diizinkan mengembang sampai menyentuh plafon tersebut, namun tidak akan pernah dizinkan menembusnya demi melindungi kelancaran macOS.

---

## 3. Hierarki Tiga Lapis Pembatasan Sumber Daya

Sebagai seorang SRE, kita menerapkan prinsip **Defense-in-Depth (Pertahanan Berlapis)**. Mengatur batas memori di OrbStack saja belum cukup jika kita tidak mengatur batas pada tiap container di dalamnya.

### Diagram Hierarki Memori di Laptop Kita

```text
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ LAPIS 1: Total RAM Fisik MacBook (Contoh: 16 GB)                        │
 │  ├── Untuk macOS + Browser + VS Code (~6 GB - 8 GB)                     │
 │  │                                                                      │
 │  └── LAPIS 2: Plafon Global OrbStack (Memory Limit: 8 GB - 10 GB)       │
 │       │  (Total seluruh Machine + Container tidak boleh lewat dari ini) │
 │       │                                                                 │
 │       ├── LAPIS 3A: Container Aplikasi Web (--memory="512m")            │
 │       ├── LAPIS 3B: Container Database (--memory="1g")                  │
 │       └── LAPIS 3C: Linux Machine 'devbox'                              │
 └─────────────────────────────────────────────────────────────────────────┘
```

**Mengapa kita butuh Lapis ke-3 (Per-Container Limit)?**
Bayangkan Plafon Global OrbStack (Lapis 2) kamu atur sebesar **8 GB**. Di dalamnya ada 10 container yang sedang berjalan.
Jika kamu tidak memasang batas Lapis 3 (`--memory`) pada masing-masing container, lalu ada **1 container yang mengalami bug dan memakan 7,8 GB RAM sendirian**, maka plafon 8 GB milik OrbStack akan habis! Akibatnya: **9 container lain yang tidak bersalah akan ikut terbunuh (`OOMKilled`)** karena kehabisan jatah memori di dalam rumah OrbStack!

---

### Tabel Rekomendasi Pengaturan Resource OrbStack (Untuk Bootcamp SRE)

Sesuaikan pengaturan plafon global OrbStack dengan spesifikasi RAM MacBook yang kamu miliki:

| Kapasitas RAM Fisik Mac | Rekomendasi Memory Limit OrbStack | Rekomendasi CPU Limit OrbStack | Sisa Aman untuk macOS + IDE |
|---|---|---|---|
| **8 GB** *(Minimum)* | **4 GB – 5 GB** | 4 Cores | ~3 GB – 4 GB *(Tutup tab browser berlebih saat lab berat)* |
| **16 GB** *(Standar Ideal)* | **8 GB – 10 GB** | 6 Cores (atau Auto) | ~6 GB – 8 GB *(Sangat nyaman untuk multi-node)* |
| **24 GB / 32 GB+** | **14 GB – 16 GB** | 8 Cores (atau Auto) | > 10 GB *(Leluasa untuk seluruh stack Capstone)* |

---

### Cara Mengatur Limit OrbStack (Via CLI & GUI)

Kamu bisa mengatur batas ini melalui dua cara yang saling tersinkronisasi:

#### Cara 1: Melalui Baris Perintah Terminal (`orb config`) — *Cepat & Presisi!*
```bash
# 1. Lihat konfigurasi batas memori dan CPU saat ini:
orb config get memory_mib
orb config get cpu

# 2. Atur batas maksimal RAM menjadi 8 GB (8192 MiB):
orb config set memory_mib 8192

# 3. Atur batas CPU menjadi 6 core:
orb config set cpu 6
```

#### Cara 2: Melalui Tampilan Grafis (GUI OrbStack)
1. Buka aplikasi **OrbStack** ➔ tekan `Cmd + ,` (Settings).
2. Pilih tab **System**.
3. Geser bilah **Memory limit** ke angka yang direkomendasikan (misal `8 GiB`) dan atur **CPU limit**.

---

## 4. Perbandingan Arsitektural: OrbStack vs Docker Desktop

Mungkin kamu bertanya: *"Di banyak tutorial internet orang memakai Docker Desktop. Mengapa di Bootcamp SRE 2026 ini kita memilih menggunakan OrbStack?"*

Mari kita bedah perbandingannya secara objektif dari kacamata rekayasa sistem (*Systems Engineering*):

| Aspek Teknis | **OrbStack** (Pilihan Bootcamp) | **Docker Desktop** |
|---|---|---|
| **Konsumsi RAM Saat Idle** | **Sangat Ringan (~150 – 300 MB)** | Berat (~2.5 GB – 4 GB hanya untuk diam) |
| **Waktu Menyala (*Startup*)**| **Instan (~2 detik)** | Lambat (~15 – 40 detik) |
| **Performa Baca-Tulis File** | **Sangat Cepat** (Implementasi VirtioFS/FUSE yang dioptimalkan khusus macOS) | Lebih lambat pada proyek dengan ribuan file kecil |
| **Akses Jaringan Container & VM** | **Transparan & Langsung:** Punya IP statis yang bisa di-ping langsung dari Mac + DNS `.orb.local` otomatis. | **Terisolasi NAT:** IP container tidak bisa di-ping langsung dari terminal Mac tanpa alat tambahan. |
| **Kemampuan Linux VM Utuh** | **Bawaan (OrbStack Machines):** Bisa bikin banyak VM Ubuntu dengan `systemd` untuk lab Ansible & k3s. | Tidak punya fitur pembuatan VM Linux mandiri (harus instal aplikasi lain seperti Multipass/UTM). |
| **Efisiensi Baterai & Suhu** | Sangat hemat daya di prosesor Apple M-Series; jarang memicu panas berlebih. | Lebih sering memicu proses background yang menguras baterai. |
| **Kesesuaian untuk Lab MetalLB** | **100% Mendukung:** Karena jaringan bridge VM terlihat langsung oleh Mac. | Sulit / membutuhkan peretasan routing jaringan tambahan. |

### Kapan Docker Desktop Masih Digunakan di Industri?
Meskipun OrbStack unggul telak di macOS, kamu mungkin tetap akan menemui Docker Desktop di beberapa perusahaan karena:
1. **Standarisasi Lintas OS (Windows & Linux):** OrbStack dibuat khusus secara eksklusif hanya untuk **macOS**. Jika perusahaan mewajibkan 1 aplikasi yang sama persis untuk staf pengguna laptop Windows dan Mac, mereka biasanya memakai Docker Desktop atau Rancher Desktop.
2. **Kontrak Lisensi Korporat:** Perusahaan raksasa yang sudah terlanjur membeli ribuan kursi lisensi *Docker Business* tahunan.

> 💡 **Kabar Baik bagi SRE:**
> Karena OrbStack 100% kompatibel dengan standar **Docker Engine API**, seluruh perintah `docker build`, `docker run`, `docker compose`, `docker network`, dan `buildx` yang kamu pelajari di atas OrbStack **berjalan 100% identik di server Linux maupun di Docker Desktop**! Kamu tidak dikunci pada perintah khusus, melainkan mendapatkan mesin eksekusi terbaik untuk Mac-mu.

---

## 5. Manajemen Ruang Penyimpanan SSD (Disk Hygiene)

SSD internal pada laptop MacBook sangat cepat, namun kapasitasnya terbatas dan tidak bisa di-upgrade. Seiring berjalannya minggu demi minggu di bootcamp, kamu akan mengunduh puluhan image Kubernetes, database, dan alat monitoring.

### Cara Kerja *Sparse Virtual Disk* di OrbStack
OrbStack menyimpan seluruh image container, volume, dan mesin Linux di dalam sebuah disk virtual pintar (*Sparse Disk*).
- Saat kamu menarik banyak image, ukuran pemakaian disk akan membesar.
- Namun, jika ada puluhan image bekas eksperimen minggu lalu yang tidak pernah kamu hapus, ruang SSD Mac milikmu akan terus tergerus!

### Prosedur Audit & Pembersihan Disk Rutin

#### 1. Periksa Peta Pemakaian Disk Docker
```bash
docker system df
```

*Contoh tampilan output:*
```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          18        3         4.82GB    3.91GB (81%)
Containers      5         2         120MB     85MB (70%)
Local Volumes   6         1         850MB     700MB (82%)
Build Cache     42        0         2.15GB    2.15GB
```
Perhatikan kolom **`RECLAIMABLE`** (Bisa diselamatkan/dibersihkan)! Pada contoh di atas, ada hampir **6 GB** ruang SSD yang terbuang percuma oleh image lama dan cache kompilasi.

#### 2. Tiga Tingkatan Perintah Pembersihan (`prune`)

```bash
# TINGKAT 1 (Paling Aman - Lakukan Setiap Minggu):
# Menghapus container yang sudah mati, jaringan menganggur, dan image tanpa nama (dangling):
docker system prune -f

# TINGKAT 2 (Pembersihan Cache Buildx):
# Menghapus cache proses pembuatan Dockerfile yang menumpuk:
docker builder prune -f

# TINGKAT 3 (Agresif - Lakukan Saat Pindah Fase Bootcamp):
# Menghapus SEMUA image yang saat ini sedang tidak dipakai oleh container aktif:
docker system prune -a -f
```

> ⚠️ **Peringatan Mengenai Volume:**
> Perintah `docker system prune` secara default **tidak menghapus Docker Volume** demi melindungi data database. Jika kamu benar-benar ingin menghapus seluruh volume yang sedang tidak terpakai, perintahnya adalah `docker volume prune -f` (pastikan tidak ada data lab penting di dalamnya!).

---

## 6. Hands-On Mandiri Step-by-Step (15 Menit)

Mari kita praktikkan pengaturan batas resource OrbStack, memantau beban secara langsung, dan mengaudit kebersihan ruang penyimpanan:

```bash
# --- LANGKAH 1: Periksa & Atur Konfigurasi Limit OrbStack ---
echo "Konfigurasi Memori (MiB) dan CPU saat ini:"
orb config get memory_mib
orb config get cpu

# Atur Memory Limit ke 8192 MiB (8 GB) — sesuaikan ke 4096 jika Mac kamu 8GB:
orb config set memory_mib 8192

# --- LANGKAH 2: Simulasi Beban & Pemantauan Real-Time ---
# Jalankan sebuah container yang dibatasi ketat hanya boleh memakai 256MB RAM dan 0.5 core CPU:
docker run -d --name uji-monitor --memory="256m" --cpus="0.5" alpine:3.20 \
  sh -c 'while true; do dd if=/dev/zero of=/dev/shm/uji bs=1M count=50 2>/dev/null; sleep 1; done'

# Intip konsumsi CPU dan Memori container tersebut secara langsung (1 kali cuplikan):
docker stats --no-stream uji-monitor

# Hentikan dan hapus container simulasi:
docker rm -f uji-monitor

# --- LANGKAH 3: Audit Pemakaian Disk & Pembersihan Aman ---
echo "Sebelum dibersihkan:"
docker system df

# Bersihkan sampah container mati & build cache:
docker system prune -f
docker builder prune -f

echo "Setelah dibersihkan:"
docker system df
```

---

## 7. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 gejala masalah sumber daya yang sering terjadi selama bootcamp beserta langkah penanganannya:

### Kasus 1: Kipas MacBook Berputar Kencang / Suhu Panas & Baterai Boros
- **Gejala:** Laptop tiba-tiba terasa hangat padahal kamu sedang tidak melakukan kompilasi apa pun.
- **Penyebab:** Ada satu container atau proses di dalam Linux Machine yang terjebak dalam perulangan tak berujung (*infinite loop* / *runaway process*) dan memakan 100% CPU tanpa henti di latar belakang.
- **Diagnosa & Solusi:**
  1. Cek apakah pelakunya adalah container Docker:
     ```bash
     docker stats --no-stream
     ```
     Lihat kolom `CPU %`. Jika ada container yang menyentuh `100%` atau lebih padahal sedang tidak digunakan, hentikan dengan `docker stop <nama>`.
  2. Jika bukan container, cek apakah pelakunya ada di dalam Linux Machine:
     ```bash
     orb ssh devbox top -b -n 1 | head -n 15
     ```
  3. Matikan Linux Machine yang sedang tidak dipakai dengan `orb stop <nama-machine>`.

---

### Kasus 2: Seluruh Cluster / Container Tiba-tiba Restart Bersamaan
- **Gejala:** Saat sedang menjalankan banyak layanan sekaligus, tiba-tiba semua container di OrbStack terputus sejenak lalu menyala ulang.
- **Penyebab:** Total pemakaian RAM dari seluruh container + Linux Machine telah menabrak **Plafon Global (`memory_mib`)** OrbStack, sehingga Kernel Linux di dalam OrbStack mengeksekusi pembersihan memori darurat.
- **Solusi:**
  1. Matikan mesin atau container dari modul lama yang sudah tidak dipakai (`orb list` dan `docker ps`).
  2. Jika RAM fisik Mac milikmu mencukupi (misal 16 GB atau 32 GB), naikkan sedikit plafon `orb config set memory_mib` (misalnya dari 6 GB menjadi 10 GB).

---

## 8. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Manajemen Resource (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Cek batas RAM global OrbStack** | `orb config get memory_mib` |
| **Ubah batas RAM global ke 8 GB** | `orb config set memory_mib 8192` |
| **Pantau CPU/RAM container secara live** | `docker stats` |
| **Cek rincian pemakaian disk SSD** | `docker system df` |
| **Bersihkan container mati & jaringan sisa**| `docker system prune -f` |
| **Bersihkan cache build Dockerfile** | `docker builder prune -f` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk menguji pemahamanmu tentang manajemen sumber daya lokal:

1. **Apa yang dimaksud dengan *Dynamic Memory Allocation* pada OrbStack? Jika kamu mengatur batas memori OrbStack sebesar 8 GB, apakah RAM macOS milikmu langsung berkurang 8 GB saat itu juga? Jelaskan!**
2. **Mengapa kita tetap wajib memasang batas memori per-container (`--memory`) meskipun kita sudah memasang batas memori global pada aplikasi OrbStack?**
3. **Sebutkan minimal 3 keunggulan teknis OrbStack dibandingkan Docker Desktop yang membuatnya sangat cocok digunakan untuk laboratorium simulasi server On-Premises di MacBook Apple Silicon!**
4. **Apa perbedaan dampak antara menjalankan perintah `docker system prune` biasa dengan `docker system prune -a`?**

---
*Lanjutkan ke materi berikutnya: [03-k3d-on-orbstack.md](03-k3d-on-orbstack.md) untuk mulai menyalakan cluster Kubernetes pertamamu dalam hitungan detik menggunakan **k3d** di atas OrbStack!*