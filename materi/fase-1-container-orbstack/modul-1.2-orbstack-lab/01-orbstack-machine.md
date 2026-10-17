# 01 — OrbStack Machine: VM Linux Ringan, Filesystem Sharing & Jaringan Stabil

> **Ringkasan:** Mengenal arsitektur di balik layar **OrbStack** pada Apple Silicon: memahami perbedaan peran antara **Docker Container** dan **OrbStack Linux Machine**, mengelola siklus hidup mesin virtual Linux yang memiliki `systemd` penuh, mengatur akses **SSH standar** untuk persiapan otomasi Ansible, memanfaatkan **Filesystem Sharing** dua arah, serta memahami keunggulan **IP Statis & DNS `.orb.local`** untuk simulasi jaringan bare-metal on-premises.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal MacBook, kamu akan mampu:
- Menjelaskan arsitektur unik **OrbStack** dan mengapa ia jauh lebih hemat sumber daya dibandingkan mesin virtual tradisional.
- Membedakan secara tepat kapan harus menggunakan **Docker Container** dan kapan harus menggunakan **OrbStack Linux Machine**.
- Membuat, menyalakan, menghentikan, dan menghapus Linux Machine menggunakan perintah CLI `orb`.
- Mengakses Linux Machine baik melalui jalur pintas (`orb ssh`) maupun melalui **protokol SSH standar berbasis kunci kriptografi (`~/.ssh/config`)**.
- Memanfaatkan fitur **Filesystem Sharing** antara macOS dan Linux serta memahami batasan performa I/O-nya.
- Menjelaskan keunggulan **IP Statis** dan resolusi DNS otomatis (`.orb.local`) sebagai pondasi laboratorium Kubernetes dan MetalLB di fase berikutnya.

---

## 1. Konsep & Arsitektur: Apa Sebenarnya OrbStack Itu?

### Mengapa Mac Membutuhkan Jembatan ke Dunia Linux?
Di Modul 1.1, kita telah belajar bahwa container bukanlah mesin virtual, melainkan proses yang berbagi **Kernel Linux** milik sistem operasi utama (*Host*).
Namun, laptop MacBook yang kita gunakan sehari-hari menjalankan sistem operasi **macOS** yang berintikan **Kernel XNU/Darwin** (keluarga BSD Unix), **bukan Kernel Linux**!

Oleh karena itu, agar laptop Mac bisa menjalankan container berbasis Linux maupun mensimulasikan server data center Linux, kita membutuhkan sebuah lapisan virtualisasi yang menyediakan Kernel Linux. Di sinilah **OrbStack** hadir sebagai solusi paling modern dan efisien khusus untuk prosesor Apple Silicon (M1/M2/M3/M4/M5).

### Dua Kemampuan Utama di Dalam Satu Aplikasi
Berbeda dengan alat lama yang memisahkan aplikasi pemutar container (seperti Docker Desktop) dan aplikasi pembuat VM Linux (seperti VirtualBox, UTM, atau Multipass), OrbStack menyatukan keduanya di atas **satu fondasi Kernel Linux bersama yang super ringan**:

```text
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                         macOS Host (Apple Silicon)                      │
 │                                                                         │
 │  ┌───────────────────────────────────────────────────────────────────┐  │
 │  │              OrbStack Engine (Native ARM64 Virtualization)        │  │
 │  │                                                                   │  │
 │  │   ┌─────────────────────────────┐  ┌───────────────────────────┐  │  │
 │  │   │  1. Docker Container Engine │  │  2. Linux Machines (VMs)  │  │  │
 │  │   │  (Kompatibel 100% Docker CLI│  │  (Ubuntu, Debian, Rocky)  │  │  │
 │  │   │   docker run, compose, k3d) │  │  - Punya systemd penuh    │  │  │
 │  │   │                             │  │  - Punya SSH server & IP  │  │  │
 │  │   └─────────────────────────────┘  └───────────────────────────┘  │  │
 │  │                                                                   │  │
 │  │          Satu Kernel Linux Bersama + VirtioFS + Jaringan          │  │
 │  └───────────────────────────────────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Container vs Linux Machine: Kapan Pakai yang Mana?

Bagi pemula, pertanyaan yang paling sering muncul adalah: *"Kalau keduanya sama-sama berjalan di dalam OrbStack, apa bedanya menjalankan `docker run ubuntu` (Container) dengan membuat `orb create ubuntu` (Linux Machine)?"*

> 🏨 **Analogi Hotel Kapsul (Container) vs Unit Rumah Dinas Lengkap (Linux Machine):**
> - **Docker Container (Hotel Kapsul):** Dirancang hanya untuk menampung **satu aktivitas/aplikasi tunggal** (misal: hanya menjalankan program Nginx atau Redis). Di dalamnya tidak ada ruang tamu, tidak ada dapur, tidak ada sistem manajemen layanan (`systemd`), dan tidak ada pintu masuk SSH. Begitu aplikasi utamanya selesai bekerja, kapsul langsung ditutup dan dibuang (*ephemeral*).
> - **OrbStack Linux Machine (Unit Rumah Dinas Lengkap):** Adalah sebuah **komputer server Linux utuh** yang terus menyala. Ia memiliki proses *init* **`systemd`** (PID 1 adalah `/sbin/init`), layanan SSH daemon, pengaturan firewall `ufw`, penjadwal `cron`, dan alamat IP rumah yang permanen. Kamu bisa masuk ke dalamnya, menginstal puluhan aplikasi berbeda menggunakan `apt`, me-reboot mesinnya, dan mengelolanya dari jarak jauh persis seperti server fisik di data center!

### Tabel Perbandingan & Penggunaan di Bootcamp SRE

| Karakteristik | Docker Container (`docker ...`) | OrbStack Linux Machine (`orb ...`) |
|---|---|---|
| **Proses Utama (PID 1)** | Program aplikasi itu sendiri (misal: `nginx`, `python`). | Sistem operasi penuh (**`/sbin/init` / `systemd`**). |
| **Sifat Penyimpanan** | Sementara (*Ephemeral*), hilang saat container dihapus. | **Permanen (*Persistent*)**, bertahan meskipun mesin di-restart. |
| **Layanan Systemd** | ❌ Tidak bisa menjalankan `systemctl start ...` | ✅ **Mendukung `systemctl` 100%.** |
| **Alamat IP Jaringan** | Berubah-ubah setiap kali container dibuat ulang. | **IP Statis (Tetap)**, tidak berubah walaupun di-reboot. |
| **Kapan Dipakai di Bootcamp?** | • Membungkus aplikasi (Modul 1.1)<br>• Menjalankan Docker Compose<br>• Menjalankan cluster latihan cepat **k3d** (Topik 03) | • Latihan administrasi Linux & SSH (Fase 0)<br>• Membangun cluster **k3s Multi-Node + MetalLB** (Fase 2)<br>• Target otomasi server **Ansible** (Fase 4) |

---

## 3. Membuat & Mengelola OrbStack Linux Machine

OrbStack menyediakan antarmuka grafis (GUI) yang manis, namun sebagai calon SRE sejati, kita wajib terbiasa mengendalikannya melalui baris perintah terminal menggunakan CLI **`orb`**.

### Perintah Dasar Manajemen Machine

```bash
# 1. Memeriksa versi dan status kesehatan OrbStack
orb version
orb status

# 2. Membuat Linux Machine baru berbasis Ubuntu 24.04 LTS bernama 'devbox'
orb create ubuntu:24.04 devbox

# 3. Melihat daftar seluruh Linux Machine beserta status dan ukurannya
orb list
# (atau cukup ketik: orb ls)

# 4. Menghentikan dan menyalakan kembali machine
orb stop devbox
orb start devbox
orb restart devbox
```

> 💡 **Catatan Arsitektur Mesin (`ARM64` Native):**
> Secara default, perintah `orb create ubuntu:24.04 devbox` akan membuat mesin Linux berarsitektur **`arm64` (`aarch64`)** yang cocok 100% dengan prosesor Apple Silicon milikmu. Hasilnya: mesin menyala dari posisi mati ke posisi siap pakai hanya dalam waktu **kurang dari 2 detik**!

---

### Membuktikan Keberadaan `systemd` di Dalam Machine

Mari kita masuk ke dalam mesin `devbox` menggunakan perintah pintas `orb ssh` (atau `orb shell`) dan buktikan bahwa ini adalah server Linux utuh:

```bash
# Masuk ke dalam terminal machine devbox
orb ssh devbox
```

Begitu berada di dalam prompt `ubuntu@devbox:~$`, jalankan perintah pemeriksaan berikut:
```bash
# 1. Cek proses nomor wahid (PID 1):
ps -p 1 -o pid,comm,args
# Output: 1 systemd /sbin/init  <-- Ini bukti mesin Linux utuh!

# 2. Cek status sistem systemd:
systemctl is-system-running
# Output: running (atau degraded jika ada unit opsional)

# 3. Keluar kembali ke terminal Mac:
exit
```

---

## 4. Mengonfigurasi Akses SSH Standar (Persiapan Ansible & Production)

Perintah `orb ssh devbox` memang sangat praktis karena menggunakan jalur komunikasi internal OrbStack tanpa membutuhkan kunci SSH.
**TETAPI**, di dunia kerja nyata (dan saat kita menggunakan **Ansible** di Fase 4 nanti), alat otomasi tidak mengenal perintah `orb ssh`. Alat otomasi wajib menggunakan **perintah `ssh` standar** melalui jaringan TCP port 22!

Mari kita siapkan akses SSH standar berbasis kunci publik (*Key-Based Authentication*) ke mesin `devbox`:

### Langkah 1: Pastikan Layanan `openssh-server` Aktif di Dalam Machine
Image Ubuntu bawaan OrbStack terkadang belum menginstal atau menyalakan daemon `sshd` demi menghemat memori. Mari kita aktifkan dengan satu baris perintah dari Mac:

```bash
orb ssh devbox sudo apt-get update && orb ssh devbox sudo apt-get install -y openssh-server
orb ssh devbox sudo systemctl enable --now ssh
```

### Langkah 2: Salin Kunci Publik SSH Mac Milikmu ke Dalam Machine
Kita bisa memanfaatkan jalur `orb ssh` untuk menyuntikkan kunci publik SSH yang sudah kamu buat di Fase 0 (`~/.ssh/id_ed25519.pub`) ke dalam buku tamu `~/.ssh/authorized_keys` milik `devbox`:

```bash
# Pastikan kamu sudah punya SSH key (jika belum, buat dengan: ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519)
orb ssh devbox "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
cat ~/.ssh/id_ed25519.pub | orb ssh devbox "cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Langkah 3: Uji Koneksi SSH Standar Menggunakan Domain `.orb.local`
Sekarang, tanpa memakai perintah `orb` sama sekali, panggil mesin tersebut menggunakan klien `ssh` standar dari terminal Mac:

```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@devbox.orb.local "hostname && whoami"
```
*Output:*
```text
devbox
ubuntu
```
Selamat! Mesin `devbox` milikmu kini sudah persis seperti server fisik bare-metal di data center yang siap dikendalikan dari jarak jauh oleh Ansible.

---

## 5. Filesystem Sharing: Jembatan File Dua Arah (Mac ↔ Linux)

Salah satu fitur paling produktif dari OrbStack adalah kemampuannya menjembatani sistem file antara macOS dan Linux secara otomatis tanpa konfigurasi rumit.

```text
  macOS Finder / VS Code                        Dalam Linux Machine (devbox)
 ┌──────────────────────────────┐              ┌──────────────────────────────┐
 │ /Users/budi/Developer/proyek │ ◄──────────► │ /Users/budi/Developer/proyek │
 │                              │   Otomatis   │ (atau lewat /mnt/mac/Users/) │
 │ ~/OrbStack/devbox/home/ubuntu│ ◄──────────► │ /home/ubuntu                 │
 └──────────────────────────────┘   Tersambung └──────────────────────────────┘
```

### 1. Mengakses File Mac dari Dalam Linux Machine
Saat kamu berada di dalam terminal `devbox`, seluruh folder pengguna Mac milikmu otomatis terpasang (*mounted*) dengan jalur alamat yang **sama persis**:
```bash
# Menjalankan perintah di dalam Linux devbox untuk melihat folder di Mac:
orb ssh devbox ls -la "$HOME"
# Atau melalui titik mount eksplisit:
orb ssh devbox ls -la /mnt/mac/Users
```
Artinya: kamu bisa menulis kode dengan nyaman menggunakan aplikasi **VS Code atau Cursor di macOS**, lalu langsung menguji dan menjalankannya menggunakan kompilator Linux di dalam `devbox`!

### 2. Mengakses File Linux Machine dari macOS
Sebaliknya, dari terminal Mac atau aplikasi **Finder**, kamu bisa mengintip dan mengambil file yang ada di dalam komputer Linux melalui folder khusus `~/OrbStack`:
```bash
# Dari terminal Mac, lihat isi folder /etc milik mesin Linux devbox:
ls -la ~/OrbStack/devbox/etc/
```

> ⚠️ **Tips Performa SRE (Dimana Harus Menaruh File?):**
> Jembatan file (*Filesystem Sharing* berbasis VirtioFS/FUSE) sangat cepat untuk mengedit skrip, kode aplikasi, atau file konfigurasi.
> **Namun**, jika kamu sedang melakukan operasi baca-tulis disk yang sangat masif di dalam Linux Machine (seperti menyimpan data database PostgreSQL, meng-clone repositori raksasa dengan ratusan ribu file kecil, atau merakit kernel), **simpanlah di dalam direktori asli Linux (`/home/ubuntu/...`)**, bukan di `/Users/...` milik Mac! Menyimpan langsung di filesystem native Linux (`ext4`/`btrfs`) memberikan kecepatan I/O maksimal dan bebas dari *overhead* jembatan antar-OS.

---

## 6. Keunggulan Jaringan OrbStack: IP Statis & DNS `.orb.local`

Di bagian inilah OrbStack mengalahkan hampir semua pesaingnya untuk kebutuhan laboratorium **Site Reliability Engineering (SRE)**.

### 1. Alamat IP Privat yang Stabil (Tidak Berubah Saat Reboot)
Mari kita cek alamat IP milik mesin `devbox`:
```bash
orb list
# atau ambil khusus IPv4-nya dari dalam machine:
orb ssh devbox hostname -I
```
Setiap Linux Machine di OrbStack diberikan satu alamat IPv4 khusus (biasanya di rentang subnet **`198.19.249.x`**).
Keistimewaannya:
- Alamat IP ini **bisa langsung di-`ping` atau diakses dengan `curl` dari terminal Mac** tanpa perlu mengatur *Port Forwarding* manual!
- Alamat IP ini bersifat **Statis (Tetap)** meskipun mesin `devbox` kamu matikan (`orb stop`), kamu nyalakan kembali (`orb start`), atau bahkan saat laptop Mac-mu di-restart.

> 🌐 **Mengapa IP Stabil Ini Sangat Penting untuk Fase 2 & Fase 4?**
> Di lingkungan server On-Premises nyata, server tidak berganti-ganti IP setiap kali mati lampu. Saat kita membangun cluster Kubernetes **k3s Multi-Node** dan memasang **MetalLB LoadBalancer (Modul 2.3)**, kita membutuhkan rentang alamat IP yang konsisten di satu jaringan lokal yang bisa diakses langsung dari browser laptop Mac kita. Jaringan `198.19.249.x` milik OrbStack membuat simulasi data center bare-metal menjadi 100% nyata!

---

### 2. DNS Otomatis `.orb.local` (Untuk Machine & Container!)
Kamu tidak perlu menghafal angka IP atau mengedit file `/etc/hosts` setiap kali membuat mesin atau container baru. OrbStack menyediakan server DNS otomatis yang mendaftarkan setiap nama Machine dan nama Container di bawah domain **`.orb.local`**:

| Target yang Dinyalakan | Cara Mengakses Langsung dari Browser / Terminal Mac |
|---|---|
| Linux Machine bernama **`devbox`** | `ping devbox.orb.local` atau `http://devbox.orb.local` |
| Docker Container (`--name web-toko`) | `ping web-toko.orb.local` atau `http://web-toko.orb.local` |
| Layanan Docker Compose (`proyek` / `api`) | `http://api.proyek.orb.local` |

Mari kita buktikan kecanggihan DNS otomatis ini untuk container Docker:
```bash
# 1. Jalankan container web server nginx TANPA flag -p (tanpa port forwarding sama sekali!):
docker run -d --name web-ajaib nginx:1.27-alpine

# 2. Akses langsung dari terminal Mac menggunakan nama container + .orb.local:
curl -I http://web-ajaib.orb.local

# 3. Bersihkan container:
docker rm -f web-ajaib
```
Perhatikan betapa bersihnya! Kamu bahkan tidak perlu memetakan `-p 8080:80` dan tidak perlu khawatir bentrok nomor port antar-container karena setiap container memiliki alamat IP dan domain `.orb.local` pribadinya sendiri!

---

## 7. Hands-On Mandiri Step-by-Step (15 Menit)

Jalankan skenario latihan berikut di terminal Mac milikmu untuk menguasai pengelolaan OrbStack Machine secara menyeluruh:

```bash
# --- LANGKAH 1: Membuat Linux Machine 'devbox' ---
orb create ubuntu:24.04 devbox 2>/dev/null || orb start devbox
orb list

# --- LANGKAH 2: Verifikasi Systemd & Arsitektur ---
orb ssh devbox "uname -a && systemctl is-system-running"

# --- LANGKAH 3: Membuktikan Kestabilan IP Saat Reboot ---
echo "IP Sebelum Restart:"
orb ssh devbox hostname -I

orb restart devbox

echo "IP Setelah Restart (Tetap Sama!):"
orb ssh devbox hostname -I

# --- LANGKAH 4: Menginstal Layanan Web Server via Systemd di Dalam Machine ---
orb ssh devbox sudo apt-get update
orb ssh devbox sudo apt-get install -y nginx
orb ssh devbox sudo systemctl enable --now nginx

# --- LANGKAH 5: Mengakses Layanan Machine Langsung dari Mac via Domain .orb.local ---
curl -I http://devbox.orb.local

# --- LANGKAH 6: Menguji Jembatan Filesystem Sharing ---
# Buat file di folder /tmp milik Mac, lalu baca file tersebut dari dalam Linux devbox:
echo "Pesan dari macOS untuk Linux" > /tmp/pesan-sre.txt
orb ssh devbox cat /mnt/mac/tmp/pesan-sre.txt
rm -f /tmp/pesan-sre.txt

# Matikan dulu layanan nginx di devbox agar hemat resource (mesin devbox biarkan tetap ada untuk deliverable)
orb ssh devbox sudo systemctl disable --now nginx
```

---

## 8. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 kendala yang sering ditemui peserta saat bekerja dengan OrbStack Machine beserta cara mengatasinya:

### Kasus 1: Bingung Sedang Berada di Terminal Mac atau di Dalam Linux Machine
- **Gejala:** Kamu mengetik perintah `brew install ...` atau `open .`, tetapi terminal mengeluarkan error `command not found`.
- **Penyebab:** Kamu sebelumnya telah mengetik `orb ssh devbox` dan lupa keluar (`exit`), sehingga saat ini kamu sedang berdiri di dalam sistem operasi Linux Ubuntu, bukan di macOS!
- **Diagnosa & Solusi:**
  - Selalu perhatikan tulisan *Prompt* di sebelah kiri kursor atau jalankan perintah `uname -s`:
    - Jika tertulis **`Darwin`** ➔ Kamu sedang di **macOS**.
    - Jika tertulis **`Linux`** ➔ Kamu sedang di dalam **OrbStack Machine / Container**.
  - Ketik `exit` (atau tekan `Ctrl + D`) untuk kembali ke terminal macOS.

---

### Kasus 2: Error `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` Saat SSH
- **Gejala:** Kamu menghapus mesin `devbox` lama (`orb delete devbox`), lalu membuat ulang mesin baru dengan nama `devbox` yang sama. Saat kamu melakukan `ssh ubuntu@devbox.orb.local`, SSH menolak masuk dengan peringatan layar penuh tentang kemungkinan serangan penyadapan (*Man-in-the-Middle*).
- **Penyebab:** Mesin `devbox` yang baru memiliki sidik jari kunci server (*Host Key*) yang baru, sedangkan laptop Mac milikmu masih mengingat sidik jari mesin `devbox` yang lama di dalam catatan `~/.ssh/known_hosts`.
- **Solusi:** Hapus catatan sidik jari lama tersebut menggunakan perintah:
  ```bash
  ssh-keygen -R devbox.orb.local
  ```

---

### Kasus 3: Perintah `systemctl` Gagal dengan Pesan `System has not been booted with systemd as init system (PID 1)`
- **Gejala:** Kamu ingin menyalakan layanan dengan `systemctl start nginx`, namun muncul pesan error bahwa sistem tidak di-boot dengan systemd.
- **Penyebab:** Kamu tidak sengaja menjalankan perintah tersebut di dalam **Docker Container biasa** (`docker exec -it ...`), bukan di dalam **OrbStack Linux Machine** (`orb ssh ...`)!
- **Solusi:** Ingat kembali tabel perbedaan di Bagian 2: jika skenario membutuhkan `systemd` penuh layaknya server fisik, gunakan selalu **OrbStack Machine**.

---

## 9. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Perintah OrbStack Machine (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Buat VM Linux Ubuntu 24.04 baru** | `orb create ubuntu:24.04 <nama-vm>` |
| **Lihat daftar seluruh VM Linux** | `orb list` |
| **Masuk cepat ke terminal VM** | `orb ssh <nama-vm>` |
| **Jalankan 1 perintah di VM dari Mac** | `orb ssh <nama-vm> <perintah>` |
| **Matikan / Nyalakan VM** | `orb stop <nama-vm>` / `orb start <nama-vm>` |
| **Hapus VM secara permanen** | `orb delete <nama-vm>` |
| **Akses jaringan VM/Container dari Mac**| `<nama>.orb.local` |
| **Akses folder Mac dari dalam VM** | `/mnt/mac/Users/<username-mac>` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk menguji pemahamanmu tentang fondasi lingkungan laboratorium kita:

1. **Mengapa kita membutuhkan lapisan seperti OrbStack untuk menjalankan container Linux di atas laptop MacBook, padahal macOS sendiri sudah merupakan sistem operasi berbasis UNIX?**
2. **Sebutkan minimal 3 perbedaan teknis antara sebuah *Docker Container* (hasil `docker run`) dengan sebuah *OrbStack Linux Machine* (hasil `orb create`)!**
3. **Jika kamu ditugaskan untuk menguji sebuah Ansible Playbook yang bertugas mengonfigurasi firewall `ufw` dan menginstal layanan `k3s` via `systemd`, apakah kamu akan memilih Docker Container atau OrbStack Machine sebagai targetnya? Jelaskan alasanmu!**
4. **Mengapa sebaiknya kita tidak menyimpan file database berukuran besar yang melakukan ribuan operasi tulis per detik di dalam folder `/mnt/mac/...` (Filesystem Sharing)? Di mana lokasi terbaik untuk menyimpannya?**

---
*Lanjutkan ke materi berikutnya: [02-resource-limit-vs-docker-desktop.md](02-resource-limit-vs-docker-desktop.md) untuk mempelajari cara mengatur batas aman pemakaian RAM & CPU agar MacBook tetap dingin dan responsif, serta memahami mengapa OrbStack jauh lebih unggul dibanding Docker Desktop!*