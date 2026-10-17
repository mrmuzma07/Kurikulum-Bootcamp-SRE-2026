# 03 — Container Registry: Push, Pull & Manajemen Image di GitLab

> **Ringkasan:** Memahami cara mendistribusikan Container Image dari laptop lokal ke server production melalui **Container Registry**: membedah anatomi alamat dan penamaan image (*tagging*), menerapkan standar keamanan otentikasi menggunakan **Personal Access Token (PAT)** dan **CI Job Token**, menghindari jebakan tag `:latest`, serta mengelola siklus penyimpanan image di **GitLab Container Registry**.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Menjelaskan fungsi **Container Registry** dan perbedaannya dengan repositori kode Git biasa.
- Membedah 4 komponen alamat lengkap sebuah Container Image (`host/namespace/repository:tag`).
- Menjelaskan bahaya penggunaan tag `:latest` di lingkungan production dan menerapkan strategi **Immutable Tagging** (SemVer & Git SHA).
- Melakukan otentikasi yang aman ke **GitLab Container Registry** menggunakan *Personal Access Token (PAT)* melalui `--password-stdin`.
- Mengunggah (*push*), mengunduh (*pull*), dan memverifikasi lapisan image di antarmuka GitLab.
- Memahami standar **OCI (Open Container Initiative)** dan pentingnya kebijakan pembersihan (*Cleanup Policy*) di server registry.

---

## 1. Konsep & Arsitektur: Mengapa Kita Butuh Container Registry?

### Masalah Distribusi: Image Lokal Tidak Bisa Dilihat Server Lain
Di materi sebelumnya, kamu sudah berhasil merakit sebuah image aplikasi yang ramping dan aman menggunakan perintah `docker build -t app-sre:baik .`.
Namun, di mana image tersebut berada saat ini? Image tersebut **hanya tersimpan di dalam harddisk laptop Mac milikmu sendiri**!

Bayangkan kamu memiliki 10 server bare-metal di data center Jakarta dan Surabaya yang tergabung dalam cluster Kubernetes. Bagaimana caranya agar ke-10 server tersebut bisa mengambil dan menjalankan image `app-sre:baik` yang baru saja kamu buat?
Kita tentu tidak mungkin menyalin file image menggunakan flashdisk dari satu server ke server lain. Kita membutuhkan sebuah **Gudang Distribusi Pusat** yang dapat diakses melalui jaringan. Gudang pusat inilah yang disebut **Container Registry**.

```text
 ┌──────────────────┐                      ┌─────────────────────────┐                      ┌──────────────────┐
 │   Laptop Lokal   │                      │   CONTAINER REGISTRY    │                      │   Server / Node  │
 │  ( atau Server   │ ──── docker push ──► │  (registry.gitlab.com)  │ ─── docker pull ───► │    Production    │
 │   GitLab CI )    │      (Unggah)        │                         │      (Unduh)         │  (Kubernetes/k3s)│
 └──────────────────┘                      │  📦 app:v1.0.0          │                      └──────────────────┘
   Merakit Image                           │  📦 app:v1.0.1          │                        Menjalankan Pod
   dari Dockerfile                         └─────────────────────────┘
```

> 🏪 **Analogi Resep (Git) vs Gudang Makanan Beku Siap Saji (Container Registry):**
> Pemula sering bertanya: *"Bukankah kita sudah punya GitLab untuk menyimpan kode? Kenapa harus ada Container Registry lagi?"*
> - **Git Repository (GitLab Repo):** Tempat menyimpan **Resep Masakan dan Bahan Mentah** (kode sumber `.go`, `.py`, dan file `Dockerfile`).
> - **Container Registry (GitLab Registry):** Gudang pendingin tempat menyimpan **Makanan Beku Matang Siap Saji** (Container Image hasil `docker build`). Server production tidak mau repot memasak dari bahan mentah; server production hanya ingin mengambil makanan matang dari Container Registry lalu langsung menyajikannya!

---

### Lanskap Layanan Container Registry di Dunia Kerja

| Nama Layanan | Alamat Host Default | Karakteristik & Penggunaan di Industri |
|---|---|---|
| **GitLab Container Registry** | `registry.gitlab.com` | **Pilihan utama di Bootcamp ini.** Terintegrasi langsung di dalam setiap proyek GitLab, mendukung repo privat gratis, dan terhubung otomatis dengan pipeline GitLab CI/CD. |
| **Docker Hub** | `docker.io` | Gudang image publik terbesar di dunia (tempat beradanya image resmi `alpine`, `ubuntu`, `nginx`, `postgres`). Memiliki **batas kuota unduh (*Pull Rate Limit*)** yang ketat untuk pengguna gratis. |
| **GitHub Container Registry** | `ghcr.io` | Layanan registry bawaan ekosistem GitHub. |
| **Harbor / Red Hat Quay** | *(Domain Internal Perusahaan)*| Registry **On-Premises mandiri (Self-Hosted)** yang diinstal di dalam data center perusahaan/perbankan yang tertutup dari internet luar (*air-gapped*). |

---

## 2. Membedah Anatomi Alamat Image (Naming & Tagging Convention)

Setiap Container Image di dunia memiliki "alamat rumah" yang mengikuti format baku internasional. Jika kamu salah menulis satu bagian saja dari alamat ini, perintah `docker push` akan ditolak.

### Struktur Lengkap Alamat Image

```text
registry.gitlab.com / budisantoso/sre-bootcamp / payment-service : v1.2.0
└────────┬────────┘   └──────────┬───────────┘   └──────┬──────┘   └──┬─┘
   1. Host Registry       2. Namespace Proyek     3. Nama Image     4. Tag
```

Mari kita bedah keempat bagian tersebut:
1. **Host Registry (`registry.gitlab.com`):** Alamat domain server tempat gudang penyimpanan berada.
2. **Namespace / Path Proyek (`budisantoso/sre-bootcamp`):** Nama pengguna (atau nama grup organisasi) diikuti nama repositori proyek di GitLab. **Catatan penting:** Seluruh huruf pada alamat image **wajib menggunakan huruf kecil (*lowercase*)**!
3. **Nama Image / Sub-komponen (`payment-service`):** Opsional tapi sangat berguna jika satu repositori Git memiliki lebih dari satu jenis image (misal `/frontend`, `/backend`, `/worker`).
4. **Tag (`:v1.2.0`):** Penanda versi spesifik dari image tersebut.

---

### Misteri Nama Pendek: Mengapa `docker pull alpine` Bisa Jalan?
Mungkin kamu bertanya: *"Tadi saat kita mengetik `docker pull alpine:3.20`, kenapa kita tidak perlu mengetik alamat host website-nya?"*

Karena jika kamu tidak menuliskan nama host di bagian depan, Docker secara otomatis akan menganggap kamu ingin mengambil image dari **Docker Hub (`docker.io`)** di dalam folder resmi **`library/`**!
Jadi di balik layar, Docker menerjemahkan nama pendek tersebut seperti ini:

| Yang Kamu Ketik di Terminal | Alamat Sebenarnya yang Diakses oleh Docker |
|---|---|
| `alpine` | `docker.io/library/alpine:latest` |
| `nginx:1.27-alpine` | `docker.io/library/nginx:1.27-alpine` |
| `bitnami/redis:7.2` | `docker.io/bitnami/redis:7.2` |
| `registry.gitlab.com/budi/repo/app:v1` | `registry.gitlab.com/budi/repo/app:v1` *(Menuju ke GitLab)* |

---

### Strategi Tagging Production: Mengapa `:latest` Adalah Musuh SRE?

Jika kamu tidak menulis bagian `:tag` di ujung nama image, Docker akan otomatis menempelkan tag bernama **`:latest`**.
Di lingkungan belajar, `:latest` terlihat praktis. Namun di lingkungan **Production SRE, menggunakan `:latest` sangat dilarang** karena tiga alasan fatal:

1. **Bersifat *Mutable* (Isinya Berubah-ubah Tanpa Jejak):**
   Hari Senin, tim developer mengunggah `app:latest` (berisi kode versi 1). Hari Rabu, mereka mengunggah lagi `app:latest` (berisi kode versi 2 yang ternyata mengandung bug). Karena namanya sama-sama `:latest`, versi 1 tertimpa!
2. **Ketidakkonsistenan Antar Node di Kubernetes:**
   Bayangkan kamu punya 5 server Kubernetes. Server 1 dan 2 menarik `app:latest` pada hari Senin. Pada hari Kamis, beban trafik naik dan Server 3 menyala lalu menarik `app:latest` yang baru. Sekarang di dalam satu cluster yang sama, sebagian server menjalankan kode hari Senin dan sebagian lagi menjalankan kode hari Kamis!
3. **Tidak Bisa Rollback Cepat:**
   Saat terjadi insiden jam 2 pagi, kamu tidak tahu harus mundur ke tag nama apa karena semua versi sebelumnya sama-sama dinamai `:latest`.

#### Solusi Standar SRE: Gunakan *Immutable Tag* (Tag yang Tidak Pernah Ditimpa)
Gunakan salah satu (atau gabungan) dari tiga pola penamaan berikut:
- **Semantic Versioning (SemVer):** `v1.0.0`, `v1.0.1`, `v1.1.0`. Setiap ada perubahan sekecil apa pun, nomor versi wajib dinaikkan.
- **Git Commit Short SHA:** `sha-4a8b1c2` (mengambil 7 karakter pertama dari `git rev-parse --short HEAD`). Ini sangat disukai SRE karena dari nama container yang sedang jalan di server, kita langsung tahu persis commit Git mana yang menghasilkannya!
- **Image Digest (`@sha256:...`):** Setiap image yang diunggah ke registry otomatis mendapatkan sidik jari kriptografi unik sepanjang 64 karakter, contoh:
  `alpine@sha256:beefdbd8a1da6d2915566fde36db9db0b524eb737fc57cd1367effd16dc0d06d`. Menarik image menggunakan `@sha256` menjamin 100% keaslian byte demi byte.

---

## 3. Otentikasi & Keamanan Akses ke GitLab Registry

Untuk mengunggah (*push*) image ke dalam repositori GitLab milikmu, Docker di laptopmu harus membuktikan identitasnya terlebih dahulu melalui perintah `docker login`.

> 🔒 **Aturan Keamanan GitLab:**
> Demi keamanan, GitLab **melarang penggunaan password login akun utama** saat melakukan `docker login` di terminal (terutama jika akunmu mengaktifkan *Two-Factor Authentication / 2FA*). Sebagai gantinya, kita wajib menggunakan **Personal Access Token (PAT)**.

### Langkah 1: Membuat Personal Access Token (PAT) di GitLab Web UI
1. Buka [gitlab.com](https://gitlab.com) dan login ke akunmu.
2. Klik foto profilmu di pojok kiri atas ➔ pilih **Edit profile**.
3. Di menu bilah kiri, klik **Access Tokens** ➔ klik tombol **Add new token**.
4. Isi formulir dengan prinsip *Least Privilege* (Hak akses seperlunya saja):
   - **Token name:** `orbstack-mac-registry`
   - **Expiration date:** Atur misal 30 atau 90 hari ke depan.
   - **Select scopes (Centang 2 kotak ini saja):**
     - [x] `read_registry` (Izin mengunduh image)
     - [x] `write_registry` (Izin mengunggah image)
5. Klik **Create personal access token**.
6. **PENTING:** Salin deretan kode token yang muncul (biasanya diawali `glpat-...`) dan simpan di tempat aman (seperti Password Manager). Begitu kamu menutup halaman tersebut, token itu tidak akan pernah ditampilkan lagi!

---

### Langkah 2: Melakukan `docker login` dengan Aman (`--password-stdin`)

Banyak pemula melakukan login dengan mengetik `docker login -u username -p glpat-rahasia123`.
**JANGAN LAKUKAN ITU!** Mengetik password/token langsung di baris perintah akan membuat token rahasiamu tersimpan secara *plaintext* di dalam file riwayat shell (`~/.zsh_history`)!

Cara yang benar dan aman:
```bash
# Cara 1: Interaktif (Ketik perintah ini, lalu tempel token saat diminta Password)
docker login registry.gitlab.com -u <GITLAB_USERNAME_ANDA>

# Cara 2: Menggunakan variabel atau pipe --password-stdin (Standar Skrip & Otomasi)
echo "<GITLAB_PAT_ANDA>" | docker login registry.gitlab.com -u <GITLAB_USERNAME_ANDA> --password-stdin
```

*Output jika berhasil:*
```text
Login Succeeded
```

---

### Di Mana Docker Menyimpan Sesi Login Kita?
Setelah `Login Succeeded`, Docker mencatat informasi otentikasi di dalam file **`~/.docker/config.json`**.
Mari kita periksa isinya:
```bash
cat ~/.docker/config.json
```
Di macOS dengan OrbStack, kamu akan melihat konfigurasi `"credsStore": "osxkeychain"`. Ini adalah fitur keamanan yang sangat bagus: token aslimu tidak ditulis telanjang di dalam file teks, melainkan disimpan terenkripsi di dalam **macOS Keychain**!

> 🚨 **Peringatan Higienitas Rahasia:**
> Di server Linux biasa yang belum memasang *credential helper*, file `~/.docker/config.json` menyimpan password/token hanya dengan penyandian **Base64** (yang bukan enkripsi, melainkan bisa dibaca balik oleh siapa saja dalam 1 detik dengan perintah `base64 -d`). Oleh karena itu, **jangan pernah menyalin atau meng-commit file `~/.docker/config.json` ke dalam repositori Git!**

---

### Bagaimana Pipeline CI/CD Login Tanpa Token Manual? (`CI_JOB_TOKEN`)
Kelak di **Fase 6 (GitOps & CI/CD)**, mesin otomatis GitLab Runner akan merakit dan mengunggah image setiap kali kamu melakukan Merge Request. Apakah kita harus membuat PAT manual untuk mesin tersebut?
**Tidak perlu!** GitLab menyediakan token sementara sekali pakai bernama **`$CI_JOB_TOKEN`** yang otomatis dibuat saat pipeline berjalan dan otomatis hangus begitu pipeline selesai:

```bash
# Perintah yang kelak berjalan di dalam .gitlab-ci.yml:
echo "$CI_JOB_TOKEN" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
```

---

## 4. Alur Kerja Praktis: Tag, Push, dan Pull

Ada dua cara untuk memberikan alamat GitLab Registry pada image yang kita buat:

### Cara A: Langsung Saat Proses `docker build`
```bash
docker build -t registry.gitlab.com/<username>/sre-bootcamp/demo-app:v1.0.0 .
```

### Cara B: Memberi Label Baru pada Image Lokal yang Sudah Ada (`docker tag`)
Perintah `docker tag` tidak menggandakan ukuran file di harddisk, melainkan hanya menempelkan stiker alamat baru pada Image ID yang sama:

```bash
# Format: docker tag <image-asal> <alamat-tujuan-lengkap>
docker tag app-sre:baik registry.gitlab.com/<username>/sre-bootcamp/demo-app:v1.0.0
```

### Mengunggah ke GitLab (`docker push`)
```bash
docker push registry.gitlab.com/<username>/sre-bootcamp/demo-app:v1.0.0
```

*Contoh tampilan proses push:*
```text
The push refers to repository [registry.gitlab.com/budi/sre-bootcamp/demo-app]
5f70bf18a086: Pushed
c9b1b535fdd9: Pushed
e3b0c44298fc: Layer already exists
v1.0.0: digest: sha256:8a9b2c3d4e5f... size: 942
```
> 💡 **Perhatikan tulisan `Layer already exists`!**
> Karena arsitektur layer yang kita pelajari di topik 01, jika nanti kamu merilis versi `v1.0.1` yang hanya mengubah file biner aplikasimu, Docker **tidak akan mengunggah ulang layer OS Alpine dasarnya**. Docker hanya mengunggah layer selisihnya saja, sehingga proses `docker push` selesai dalam sekejap!

---

## 5. Standar OCI & Manajemen Kebersihan Registry

### Mengenal Standar OCI (Open Container Initiative)
Dahulu, timbul kekhawatiran apakah image yang dibuat dengan Docker hanya bisa dijalankan oleh produk Docker saja. Untuk mencegah monopoli, para raksasa teknologi membentuk standar terbuka bernama **OCI (Open Container Initiative)**.
Berkat standar OCI:
- Image yang kamu build dengan Docker dan kamu simpan di GitLab Registry memiliki format yang 100% standar.
- Image tersebut dapat langsung ditarik dan dijalankan oleh runtime container di Kubernetes (**containerd** di k3s, atau **CRI-O**) tanpa membutuhkan Docker Daemon sama sekali di server production!
- Bahkan di Fase 5 nanti, kita juga akan menyimpan paket **Helm Chart** di dalam GitLab Container Registry yang sama menggunakan protokol OCI.

### Kebijakan Pembersihan (Cleanup Policy)
Di proyek aktif yang melakukan puluhan deployment setiap minggu, jumlah tag image di dalam registry akan membengkak hingga ratusan gigabyte.
Sebagai SRE yang baik, kamu dapat mengatur pembersihan otomatis di GitLab:
1. Buka proyekmu di GitLab ➔ **Settings** ➔ **Packages and registries**.
2. Pada bagian **Container Registry cleanup policies**, klik **Set cleanup rules**.
3. Atur agar GitLab selalu **menyimpan 10 tag terakhir** (atau selalu melindungi tag berformat rilis resmi `v*`), dan otomatis menghapus tag pengujian lama yang sudah berumur lebih dari 30 hari.

---

## 6. Hands-On Mandiri Step-by-Step (15 Menit)

Mari kita uji seluruh siklus pengiriman image dari laptop Mac milikmu menuju gudang awan GitLab Container Registry, lalu menariknya kembali!

### Prasyarat
- Kamu sudah memiliki repositori **`sre-bootcamp`** di gitlab.com (dari Modul 0.3).
- Kamu sudah membuat **Personal Access Token (PAT)** dengan centang `read_registry` dan `write_registry`.

### Langkah Pelaksanaan

```bash
# --- LANGKAH 1: Tetapkan Variabel Username GitLab (HURUF KECIL SEMUA!) ---
# Ganti 'username_anda' dengan username GitLab milikmu dalam huruf kecil:
export GITLAB_USER="username_anda"

# --- LANGKAH 2: Login ke GitLab Container Registry ---
docker login registry.gitlab.com -u "$GITLAB_USER"
# (Tempel Personal Access Token glpat-... milikmu saat diminta Password)

# --- LANGKAH 3: Buat Image Aplikasi Sederhana ---
cd ~
mkdir -p lab-registry && cd lab-registry

cat << 'EOF' > Dockerfile
FROM alpine:3.20
RUN adduser -D -u 10001 appuser
USER 10001
CMD ["sh", "-c", "echo 'Halo dari GitLab Container Registry milik SRE!'"]
EOF

# Build dengan alamat lengkap GitLab Registry:
docker build -t "registry.gitlab.com/${GITLAB_USER}/sre-bootcamp/sapa-sre:v0.1.0" .

# --- LANGKAH 4: Unggah (Push) ke GitLab Registry ---
docker push "registry.gitlab.com/${GITLAB_USER}/sre-bootcamp/sapa-sre:v0.1.0"

# --- LANGKAH 5: Buktikan Image Sudah Aman di Awan ---
# Hapus paksa image tersebut dari harddisk laptop lokalmu!
docker rmi "registry.gitlab.com/${GITLAB_USER}/sre-bootcamp/sapa-sre:v0.1.0"

# Pastikan sudah tidak ada di lokal:
docker images | grep sapa-sre || echo "Image lokal sudah bersih!"

# Sekarang jalankan langsung (Docker akan otomatis melakukan pull dari GitLab):
docker run --rm "registry.gitlab.com/${GITLAB_USER}/sre-bootcamp/sapa-sre:v0.1.0"

# --- LANGKAH 6: Bersihkan Folder Latihan ---
cd ~ && rm -rf lab-registry
```

### Verifikasi di Web Browser GitLab
Buka halaman proyek `sre-bootcamp` milikmu di GitLab ➔ pilih menu kiri **Operate** (atau **Deploy**) ➔ **Container Registry**.
Kamu akan melihat paket **`sapa-sre`** lengkap dengan tag **`v0.1.0`**, ukuran file, dan kode *digest* `sha256`-nya!

---

## 7. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 pesan error yang paling sering membuat peserta terjebak saat berinteraksi dengan Container Registry beserta solusinya:

### Kasus 1: Error `denied: requested access to the resource is denied` Saat Push
- **Gejala:** Kamu sudah berhasil `docker login`, namun saat menjalankan `docker push`, muncul pesan:
  `denied: requested access to the resource is denied`.
- **Penyebab Umum (Cek 3 Hal Ini):**
  1. **Salah ketik Path Proyek:** Nama alamat image tidak sama persis dengan alamat URL repositori GitLab milikmu. Ingat formatnya harus menyertakan nama repo: `registry.gitlab.com/<username>/<nama-repo>/<nama-image>:<tag>`.
  2. **Ada Huruf Kapital pada Username:** Username GitLab kamu mungkin `BudiSantoso`, tetapi di alamat Docker tag kamu **wajib menulisnya dalam huruf kecil semua**: `budisantoso`.
  3. **Token Kurang Izin:** Saat membuat PAT, kamu hanya mencentang `read_registry` dan lupa mencentang `write_registry`.

---

### Kasus 2: Error `HTTP Basic: Access denied` Saat `docker login`
- **Gejala:** Login selalu gagal meskipun kamu yakin password yang diketik benar.
- **Penyebab:** Kamu memasukkan password akun GitLab biasa, atau token PAT sudah kedaluwarsa (*expired*).
- **Solusi:** Buat *Personal Access Token* baru di menu **User Settings ➔ Access Tokens**, salin token berawalan `glpat-...` tersebut, dan gunakan sebagai password saat `docker login`.

---

### Kasus 3: Error `toomanyrequests: You have reached your pull rate limit`
- **Gejala:** Saat men-deploy banyak aplikasi atau menjalankan CI/CD yang menarik image dari Docker Hub (seperti `alpine` atau `ubuntu`), proses tiba-tiba gagal dengan pesan *pull rate limit exceeded*.
- **Penyebab:** Docker Hub membatasi pengguna anonim (yang tidak login) hanya boleh mengunduh maksimal **100 image per 6 jam per alamat IP publik**. Jika satu kantor atau data center berbagi satu IP publik yang sama, kuota ini cepat habis!
- **Solusi SRE:**
  1. Lakukan `docker login` ke Docker Hub agar kuota naik menjadi 200 pull/6 jam.
  2. Salin (*mirror*) base image yang sering dipakai ke dalam **GitLab Container Registry** milikmu sendiri, atau aktifkan fitur **GitLab Dependency Proxy** agar server menarik dari cache internal GitLab tanpa terhadang limit Docker Hub.

---

## 8. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Perintah Registry (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Login aman ke GitLab Registry** | `docker login registry.gitlab.com -u <username>` |
| **Beri label alamat registry pada image** | `docker tag <img-lokal> registry.gitlab.com/<user>/<repo>/<app>:<tag>` |
| **Unggah image ke registry** | `docker push registry.gitlab.com/<user>/<repo>/<app>:<tag>` |
| **Unduh image dari registry** | `docker pull registry.gitlab.com/<user>/<repo>/<app>:<tag>` |
| **Logout (hapus kredensial sesi)** | `docker logout registry.gitlab.com` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk memastikan pemahamanmu sebelum melangkah ke topik Multi-Architecture:

1. **Apa perbedaan mendasar antara fungsi Git Repository dan Container Registry di dalam alur pengiriman aplikasi ke server production?**
2. **Ketika seseorang mengetik `docker pull redis:7.2`, ke server registry mana sebenarnya Docker menghubungi, dan bagaimana bentuk alamat lengkapnya di balik layar?**
3. **Sebutkan minimal 2 alasan mengapa penggunaan tag `:latest` sangat dilarang pada manifes deployment di server production, dan pola penamaan tag apa yang sebaiknya digunakan sebagai penggantinya?**
4. **Mengapa mengetik `docker login -u budi -p tokenrahasia123` secara langsung di terminal dianggap sebagai praktik keamanan yang buruk di Linux/macOS?**

---
*Lanjutkan ke materi berikutnya: [04-multi-arch-arm64.md](04-multi-arch-arm64.md) untuk mempelajari mengapa image yang dibuat di MacBook Apple Silicon (ARM64) bisa gagal berjalan di server Intel/AMD (AMD64), serta cara merakit satu image sakti yang berjalan mulus di kedua arsitektur tersebut menggunakan `docker buildx`!*