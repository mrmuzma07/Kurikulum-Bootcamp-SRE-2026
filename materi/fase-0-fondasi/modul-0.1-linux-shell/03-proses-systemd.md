# 03 — Manajemen Proses & Systemd di Linux

> **Ringkasan:** Memahami apa itu proses di Linux, cara memantau aplikasi yang sedang berjalan dan konsumsi resource-nya, cara mengirim sinyal penghentian dengan aman, mengelola background job, serta membangun dan merawat layanan otomatis (*service*) menggunakan `systemd` dan `journalctl`.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Menjelaskan perbedaan antara program statis dan **proses aktif**, serta silsilah hierarki proses (PID, PPID, dan pohon proses).
- Membaca dan menganalisis metrik penggunaan CPU dan RAM menggunakan perintah `ps`, `top`, dan `htop`.
- Mengirim sinyal komunikasi proses yang tepat (`SIGTERM`, `SIGINT`, `SIGHUP`, `SIGKILL`) sesuai etika operasional SRE.
- Mengelola pekerjaan di latar belakang (*background jobs*) dan memahami batas ketahanan sesi SSH.
- Mengontrol layanan sistem menggunakan `systemctl` (`start`, `stop`, `restart`, `reload`, `enable`, `disable`).
- Menyelidiki akar masalah (*root cause*) kegagalan layanan menggunakan pembaca log terpusat `journalctl`.
- Menulis file unit `systemd` (`.service` dan `.timer`) buatan sendiri yang *production-ready* dilengkapi mekanisme *auto-restart*.

---

## 1. Konsep Dasar: Apa Itu "Proses" di Linux?

Sebelum menyentuh perintah terminal, mari kita samakan pemahaman tentang apa sebenarnya yang dimaksud dengan **Proses**.

> 🍳 **Analogi Buku Resep Masakan vs Koki yang Memasak:**
> - **Program / Aplikasi (Statis):** Seperti buku resep masakan yang tersimpan diam di rak lemari (harddisk). File biner seperti `/usr/bin/nginx` atau script `/opt/app.py` hanyalah teks instruksi mati.
> - **Proses (Dinamis):** Saat kamu menjalankan program tersebut, itu seperti seorang koki yang mulai menyalakan kompor dan memasak mengikuti buku resep. Proses adalah **program yang sedang hidup dan dieksekusi di memori (RAM)** serta menggunakan daya komputasi (CPU).

### Mengapa SRE Sangat Peduli dengan Proses?
Di server production, tidak ada tombol restart fisik yang ditekan sembarangan. Seluruh aplikasi web, database, perute jaringan, dan sistem monitoring berjalan sebagai proses. Jika server lemot, aplikasi *down*, atau memori server habis, tugas pertama seorang SRE adalah **mengidentifikasi proses mana yang bermasalah dan mengambil tindakan presisi**.

### Identitas Proses: PID dan PPID
Setiap kali sebuah proses lahir di Linux, sistem operasi memberikannya nomor identitas unik:
- **`PID` (Process ID):** Nomor KTP unik milik proses tersebut (misal: PID `1420`).
- **`PPID` (Parent Process ID):** Nomor PID dari proses induk (*orang tua*) yang melahirkannya.

Di Linux, **setiap proses (kecuali satu) selalu memiliki orang tua**. Ketika kamu membuka terminal dan mengetik perintah `ls`, maka proses shell bash milikmu bertindak sebagai orang tua (`PPID`), dan perintah `ls` lahir sebagai proses anak (`PID`).

```text
                  [ systemd ] (PID 1 - Nenek Moyang Seluruh Proses)
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
      [ sshd ] (PID 850)      [ nginx ] (PID 1100)
         │
         ▼
    [ bash shell ] (PID 2400)
         │
         ▼
     [ python3 ] (PID 3510)
```

> 👑 **Siapa PID 1?**
> Saat komputer/server Linux pertama kali menyala, Kernel Linux akan melahirkan satu proses pertama yang memiliki **`PID = 1`**, yaitu **`systemd`** (pada distro modern seperti Ubuntu, Debian, RHEL). `systemd` adalah nenek moyang dari seluruh proses yang ada di server. Jika PID 1 mati, seluruh server akan mengalami *kernel panic* dan mati (*crash*).

---

## 2. Status Kehidupan Proses (Process States)

Saat kamu memeriksa proses, Linux menampilkan kode status satu huruf (**`STAT`**). Berikut arti kode yang wajib diketahui:

| Kode STAT | Nama Status | Penjelasan Ramah Pemula |
|:---:|---|---|
| **`R`** | **Running / Runnable** | Proses sedang aktif bekerja menggunakan CPU, atau sedang mengantre di CPU siap dieksekusi. |
| **`S`** | **Interruptible Sleep** | Status paling umum. Proses sedang tidur santai menunggu kejadian (misal Nginx sedang tidur menunggu ada pengunjung web datang). Proses ini bisa dibangunkan kapan saja. |
| **`D`** | **Uninterruptible Sleep** | Proses sedang menunggu perangkat keras (*hardware*), biasanya sedang membaca/menulis ke harddisk (*Disk I/O*). **PENTING:** Proses berstatus `D` tidak bisa diganggu, bahkan perintah `kill -9` tidak akan mempan sampai harddisk selesai merespons! |
| **`T`** | **Stopped / Traced** | Proses sedang dihentikan sementara (misal kamu menekan tombol `Ctrl + Z` di terminal). |
| **`Z`** | **Zombie** | Proses yang **sudah mati**, tapi nomor PID-nya masih menggantung di daftar proses karena proses induknya (*parent*) belum sempat membaca laporan status kematian anaknya. |

> 🧟 **Apakah Proses Zombie Berbahaya?**
> Proses zombie **tidak memakan memori (RAM) dan tidak memakan CPU**, karena programnya sebenarnya sudah selesai berjalan. Namun, zombie tetap memakan 1 jatah nomor PID. Jika ada ribuan zombie yang dibiarkan menumpuk, server bisa kehabisan jatah nomor PID sehingga tidak bisa menjalankan aplikasi baru. Zombie akan lenyap dengan sendirinya begitu proses induknya dimatikan atau mengambil alih laporan kematian.

---

## 3. Mengintip & Memantau Proses (`ps`, `top`, `htop`, `pstree`)

### A. Memotret Proses dengan `ps` (Process Snapshot)
Perintah `ps` memberikan foto kilat (*snapshot*) dari proses yang sedang aktif pada detik kamu menekan Enter.

```bash
ps                  # Hanya menampilkan proses yang berjalan di jendela terminal kamu saat ini
ps aux              # Format paling populer (gaya BSD): Tampilkan SEMUA proses di seluruh sistem
ps -ef              # Format alternatif (gaya System V): Tampilkan semua proses beserta pohon PPID-nya
```

#### Cara Membaca Output `ps aux`:
```text
USER       PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1   0.0  0.2 168432 11200 ?        Ss   Oct10   0:04 /sbin/init
ubuntu    2450   1.5  3.2 540200 65400 pts/0    S+   14:20   0:01 python3 app.py
```

Mari bedah kolom-kolom kritisnya:
- **`USER`:** Siapa pemilik yang menjalankan proses tersebut.
- **`PID`:** Nomor identitas proses.
- **`%CPU` & `%MEM`:** Persentase daya prosesor dan memori RAM yang sedang disedot.
- **`VSZ` vs `RSS` (Konsep Memori Penting!):**
  - **`VSZ` (Virtual Memory Size):** Total memori yang *dipesan/dialokasikan* oleh program (seperti memesan plafon kredit).
  - **`RSS` (Resident Set Size):** Jumlah memori fisik RAM yang **benar-benar sedang dipakai secara nyata** di chip RAM (seperti uang tunai yang sudah ditarik). Saat menyelidiki konsumsi RAM, **selalu perhatikan kolom RSS**!
- **`STAT`:** Status proses (misal `S+`: sedang tidur santai di foreground).
- **`COMMAND`:** Perintah lengkap yang melahirkan proses tersebut.

#### Trik Filter Cepat `ps` untuk SRE:
```bash
# Mencari proses tertentu (misal mencari proses 'nginx')
ps aux | grep nginx

# Menghindari tulisan 'grep' ikut muncul di hasil pencarian:
ps aux | grep [n]ginx

# Menampilkan 5 proses yang paling rakus memakan CPU:
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%cpu | head -n 6

# Menampilkan 5 proses yang paling rakus memakan RAM:
ps -eo pid,ppid,user,%mem,cmd --sort=-%mem | head -n 6
```

---

### B. Memantau Proses Secara Live dengan `top`
Jika `ps` seperti foto diam, maka **`top`** adalah rekaman CCTV video langsung yang memperbarui data setiap 3 detik.

```bash
top
```

#### Cara Membaca Bagian Kepala (*Header*) `top`:
```text
top - 14:35:10 up 6 days,  2:10,  1 user,  load average: 0.15, 0.08, 0.05
Tasks: 110 total,   1 running, 109 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.5 us,  1.0 sy,  0.0 ni, 96.0 id,  0.5 wa,  0.0 hi,  0.0 si
MiB Mem :   1984.2 total,    450.1 free,    850.5 used,    683.6 buff/cache
```

1. **Load Average (`0.15, 0.08, 0.05`):** Rata-rata beban kerja server dalam rentang **1 menit, 5 menit, dan 15 menit terakhir**.
   > 🌉 **Analogi Beban Jembatan (Load Average):**
   > Jika servermu memiliki **1 Core CPU**, maka angka beban `1.00` berarti jembatan sedang terisi pas 100%. Jika angka beban mencapai `2.50`, berarti ada mobil yang harus mengantre panjang karena kapasitas CPU kelebihan beban 150%!
2. **Detail `%Cpu(s)`:**
   - **`us` (User):** Persentase CPU yang dipakai oleh kode aplikasi buatanmu.
   - **`sy` (System):** Persentase CPU yang dipakai oleh Kernel Linux.
   - **`id` (Idle):** Persentase CPU yang sedang santai / menganggur (semakin tinggi semakin bagus!).
   - **`wa` (I/O Wait — KUNCI SRE!):** Persentase waktu CPU menganggur hanya gara-gara **menunggu harddisk yang lambat**. Jika angka `wa` tinggi (>20%), artinya aplikasi lambat bukan karena CPU kurang kencang, tapi karena harddisk sedang tersedak membaca/menulis data!

#### Tombol Pintasan Ajaib di dalam `top`:
- Tekan **`P`** (Shift + p) : Urutkan proses berdasarkan pemakaian **CPU tertinggi**.
- Tekan **`M`** (Shift + m) : Urutkan proses berdasarkan pemakaian **RAM/Memori tertinggi**.
- Tekan **`k`** : Ketik PID untuk membunuh proses langsung dari layar `top`.
- Tekan **`1`** : Tampilkan rincian performa CPU per masing-masing Core (berguna jika CPU multi-core).
- Tekan **`q`** : Keluar dari `top`.

---

### C. Alternatif Lebih Nyaman: `htop` & `pstree`
Program `top` bawaan Linux agak kaku. SRE modern biasanya menginstal **`htop`** yang memiliki tampilan berwarna, grafik batang visual, dan navigasi mouse/panah keyboard yang ramah:

```bash
sudo apt install -y htop
htop
```

Dan untuk melihat silsilah keluarga proses secara visual:
```bash
pstree -p               # Menampilkan pohon silsilah proses lengkap dengan nomor PID-nya
```

---

## 4. Sinyal Proses (Signals) — Cara Kita Berbicara dengan Proses

Bagaimana cara kita meminta sebuah proses untuk berhenti, memuat ulang konfigurasi, atau mematikannya secara paksa? Di Linux, kita melakukannya dengan mengirimkan **Sinyal (Signals)**.

> 📢 **Analogi Peluit Wasit:**
> Mengirim sinyal seperti wasit sepak bola yang meniup peluit dengan pola nada berbeda: ada peluit jeda minum (*pause*), peluit pelanggaran ringan (*warning*), hingga kartu merah langsung keluar lapangan (*kill*).

Berikut adalah 5 sinyal terpenting yang wajib dipahami luar kepala:

| Nama Sinyal | Nomor Sinyal | Perintah Singkat | Perilaku & Etika SRE |
|---|:---:|---|---|
| **`SIGTERM`** | **`15`** | `kill 1234` | **Cara Sopan (Default).** Memberitahu proses: *"Tolong bersiap-siap untuk berhenti."* Aplikasi yang baik akan menyelesaikan transaksi database yang sedang jalan, menutup koneksi, membersihkan file sementara, baru kemudian menutup diri dengan damai (*Graceful Shutdown*). |
| **`SIGINT`** | **`2`** | `Ctrl + C` | **Interupsi Keyboard.** Sinyal yang dikirimkan saat kamu menekan tombol `Ctrl + C` di terminal untuk membatalkan perintah yang sedang berjalan. |
| **`SIGHUP`** | **`1`** | `kill -1 1234` | **Reload Konfigurasi (Hangup).** Memberitahu aplikasi server (seperti Nginx atau SSH): *"Tolong baca ulang file konfigurasimu di `/etc/` tanpa perlu mematikan aplikasi!"* Layanan tetap aktif melayani pengguna tanpa downtime. |
| **`SIGKILL`** | **`9`** | `kill -9 1234` | **Eksekusi Mati Paksa (Seketika).** Sinyal ini **TIDAK dikirimkan ke aplikasi, melainkan langsung ditangani oleh Kernel Linux**. Kernel langsung mencabut aliran memori dan menghapus proses seketika tanpa peringatan! |
| **`SIGSTOP`** & **`SIGCONT`** | `19` & `18` | `kill -STOP` / `kill -CONT` | **Pause dan Resume.** Membekukan proses sementara di memori, lalu melanjutkannya kembali nanti. |

### Perintah Praktis Mengirim Sinyal:
```bash
kill 3420                   # Mengirim SIGTERM (15) ke PID 3420 (cara sopan & standar)
kill -TERM 3420             # Sama dengan perintah di atas secara eksplisit

kill -HUP $(pidof nginx)    # Meminta Nginx membaca ulang konfigurasinya tanpa mati
killall nginx               # Mengirim SIGTERM ke SEMUA proses yang bernama 'nginx'
pkill -f "python3 app.py"   # Mencari dan menghentikan proses berdasarkan pola teks perintahnya

kill -9 3420                # Mengirim SIGKILL (9) — HANYA JIKA PROSES MACET/MEMBANDEL!
```

> ⚠️ **ETIKA EMAS SRE: Mengapa `kill -9` adalah Langkah Terakhir?**
> Jangan pernah langsung mengetik `kill -9` begitu melihat aplikasi bermasalah!
> Mengapa? Karena saat dihantam `kill -9`, aplikasi tidak diberi kesempatan 1 milidetik pun untuk merapikan datanya. Akibatnya:
> 1. File database bisa **korup / rusak permanen**.
> 2. File kunci (*lock file*) tertinggal di harddisk, sehingga saat aplikasi dinyalakan lagi, ia menolak jalan karena mengira masih ada instance lama yang aktif.
> 
> **Urutan Etika Berhenti:**
> Kirim **`SIGTERM`** (atau `kill <PID>`) $\longrightarrow$ Tunggu 5–10 detik $\longrightarrow$ Jika proses benar-benar membeku (*hung*) dan tidak merespons, barulah kirim **`kill -9`**.

---

## 5. Mengelola Tugas: Foreground vs Background & Sesi Terminal

Saat kamu menjalankan script panjang (misalnya backup data 1 jam) di terminal:
- **Foreground (Garis Depan):** Terminalmu terkunci dan tidak bisa mengetik perintah lain sampai script tersebut selesai. Jika kamu menutup jendela terminal atau koneksi SSH putus, script tersebut **ikut mati seketika**!
- **Background (Garis Belakang):** Script berjalan di latar belakang, terminal langsung bebas digunakan kembali.

```bash
# Menjalankan script di latar belakang (tambahkan simbol '&' di ujung perintah):
python3 backup.py &
# Output: [1] 4521  <-- Artinya Job Nomor 1, dengan PID 4521
```

### Navigasi Pekerjaan Shell (`jobs`, `fg`, `bg`)
```bash
jobs                # Melihat daftar perintah yang sedang berjalan di background terminal ini
fg %1               # Membawa Job Nomor 1 kembali ke Foreground (garis depan)
```
*(Trik: Jika sebuah perintah sedang jalan di foreground dan kamu ingin melemparnya ke background, tekan tombol **`Ctrl + Z`** untuk mem-pause proses, lalu ketik perintah **`bg`** untuk menyuruhnya lanjut berjalan di background).*

### Masalah Terbesar Background Tradisional: Sinyal `SIGHUP`
Ketika kamu login ke server lewat SSH lalu menjalankan `python3 server.py &`, proses tersebut memang berjalan di background. Namun, begitu laptopmu kamu tutup atau koneksi Wi-Fi terputus, shell SSH akan mengirimkan sinyal **`SIGHUP` (Hangup)** yang **mematikan semua proses anak di dalamnya**.

Dulu, orang menggunakan trik lama:
```bash
nohup python3 server.py > app.log 2>&1 &
```
*(Perintah `nohup` bertugas menangkis sinyal SIGHUP agar proses tidak mati saat terminal ditutup).*

Namun di dunia modern, **menjalankan aplikasi production menggunakan `nohup ... &` adalah praktik kuno yang buruk!** Mengapa?
- Jika server reboot, aplikasimu tidak akan hidup lagi otomatis.
- Jika aplikasi crash di tengah malam, tidak ada yang menyalakannya kembali.
- Tidak ada standarisasi pemantauan kesehatan aplikasi.

Untuk itulah kita membutuhkan manajer layanan modern: **`systemd`**!

---

## 6. `systemd` — Sang Manajer & Penjaga Layanan Server

Di era modern, hampir seluruh distro Linux menggunakan **`systemd`** sebagai sistem inisialisasi (*Init System*) dan pengelola layanan (*Service Manager*).

Layanan di Linux sering disebut **Daemon** (aplikasi yang berjalan terus-menerus di latar belakang untuk melayani kebutuhan sistem atau pengguna, seperti web server atau database).

### Mengendalikan Layanan dengan `systemctl`
Untuk mengendalikan service, kita menggunakan perintah sakti **`systemctl`**:

```bash
sudo systemctl status nginx         # Cek status kesehatan layanan secara detail
sudo systemctl start nginx          # Menyalakan layanan yang sedang mati
sudo systemctl stop nginx           # Mematikan layanan
sudo systemctl restart nginx        # Mematikan lalu menyalakan ulang layanan
sudo systemctl reload nginx         # Meminta layanan membaca ulang file konfigurasi (graceful)
```

### Konsep yang Sering Membingungkan Pemula: `start/stop` vs `enable/disable`
Perhatikan perbedaan besar kedua konsep ini:

```text
┌───────────────────────────────────────┬────────────────────────────────────────┐
│        START / STOP                   │          ENABLE / DISABLE              │
├───────────────────────────────────────┼────────────────────────────────────────┤
│ • Berlaku untuk DETIK INI JUGA.       │ • Berlaku untuk MASA DEPAN (Saat Boot).│
│ • 'start' = Nyalakan sekarang.        │ • 'enable' = Pasang saklar auto-start. │
│ • Jika server di-restart (reboot),    │ • Jika server reboot, service otomatis │
│   status ini TIDAK diingat!           │   langsung menyala kembali.            │
└───────────────────────────────────────┴────────────────────────────────────────┘
```

> 💡 **Analogi Saklar Lampu:**
> - `systemctl start`: Kamu menyalakan saklar lampu kamar saat ini.
> - `systemctl enable`: Kamu memasang sensor pintar agar setiap kali listrik rumah baru menyala setelah padam, lampu tersebut otomatis langsung hidup sendiri.

```bash
sudo systemctl enable nginx         # Pasang agar Nginx otomatis jalan saat server baru boot
sudo systemctl disable nginx        # Matikan fitur auto-start

# Jurus Singkat (Nyalakan sekarang DAN pasang auto-start sekaligus):
sudo systemctl enable --now nginx
```

### Cara Membaca Output `systemctl status`:
```text
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2026-10-14 10:00:00 UTC; 2 days ago
   Main PID: 1120 (nginx)
      Tasks: 3 (limit: 2315)
     Memory: 24.5M
        CPU: 1.250s
     CGroup: /system.slice/nginx.service
             ├─1120 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             ├─1121 nginx: worker process
             └─1122 nginx: worker process

Oct 14 10:00:00 server01 systemd[1]: Starting A high performance web server...
Oct 14 10:00:00 server01 systemd[1]: Started A high performance web server.
```
- **`Loaded`:** Lokasi file konfigurasinya, dan apakah statusnya `enabled` (auto-start aktif).
- **`Active`:** Statusnya apakah `active (running)`, `inactive (dead)`, atau `failed`.
- **`Main PID`:** Nomor PID dari proses utama aplikasi tersebut.
- **`Memory` & `CPU`:** Konsumsi resource nyata dari layanan ini beserta seluruh anak prosesnya.

---

## 7. Membaca Log Layanan dengan `journalctl`

Dulu, setiap aplikasi menaruh log di tempat yang berbeda-beda di dalam `/var/log/`.
Dengan `systemd`, ada komponen pencatat log terpadu bernama **`systemd-journald`**. Apa pun yang dikeluarkan oleh aplikasi ke layar (*standard output* dan *standard error*), semuanya otomatis ditangkap rapi oleh `journald`!

Untuk membaca log tersebut, kita menggunakan alat detektif bernama **`journalctl`**.

```bash
# 1. Menampilkan log dari satu service tertentu saja:
journalctl -u nginx

# 2. Jurus SRE: Pantau log secara LIVE (Real-Time) saat aplikasi sedang dipakai
#    (sama seperti 'tail -f'):
journalctl -u nginx -f

# 3. Menampilkan 50 baris log TERAKHIR tanpa terputus pagination:
journalctl -u nginx -n 50 --no-pager

# 4. Melakukan investigasi berdasarkan rentang waktu:
journalctl -u nginx --since "15 minutes ago"
journalctl -u nginx --since "2026-10-16 08:00:00" --until "2026-10-16 09:30:00"

# 5. Hanya cari log yang tingkat keparahannya ERROR ke atas (abaikan info biasa):
journalctl -u nginx -p err

# 6. Melihat catatan log sejak server terakhir kali di-boot (dinyalakan):
journalctl -b
```

---

## 8. Hands-On: Membuat Systemd Service Sendiri yang *Production-Ready*

Mari kita buktikan betapa mudahnya membuat aplikasi buatan kita sendiri menjadi sebuah layanan resmi yang dijaga oleh Linux!

### Skenario:
Kita memiliki sebuah script aplikasi web sederhana di `/opt/myapp/server.py`. Kita ingin script ini:
1. Berjalan menggunakan user khusus `ubuntu` (bukan user root!).
2. Menyimpan log otomatis ke `journalctl`.
3. **Jika aplikasinya crash / mati tiba-tiba, systemd harus otomatis menyalakannya kembali dalam 5 detik!**

### Langkah 1: Buat Direktori dan Script Aplikasinya
```bash
sudo mkdir -p /opt/myapp
sudo chown ubuntu:ubuntu /opt/myapp

# Buat script server sederhana:
cat << 'EOF' > /opt/myapp/server.py
#!/usr/bin/env python3
import time, sys

print("=== Aplikasi SRE Demo Berhasil Dinyalakan ===", flush=True)
counter = 0

while True:
    time.sleep(5)
    counter += 1
    print(f"Detak jantung aplikasi ke-{counter} (Berjalan normal)", flush=True)
    
    # Simulasi crash jika mencapai hitungan ke-10:
    if counter == 10:
        print("CRITICAL: Terjadi error fatal! Aplikasi sengaja crash...", file=sys.stderr, flush=True)
        sys.exit(1)
EOF

chmod +x /opt/myapp/server.py
```

### Langkah 2: Buat File Unit Service di `/etc/systemd/system/`
Semua file service buatan sendiri ditaruh di folder **`/etc/systemd/system/`** dengan akhiran **`.service`**.

```bash
sudo nano /etc/systemd/system/myapp.service
```
*(Atau gunakan perintah cat di bawah ini)*:

```ini
[Unit]
Description=Aplikasi Web SRE Demo
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/server.py
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

#### Bedah Bagian File Service:
- **`[Unit]`**:
  - `Description`: Nama penjelas yang manusiawi.
  - `After=network.target`: Jangan nyalakan aplikasi ini sebelum sistem jaringan Linux siap!
- **`[Service]`**:
  - `Type=simple`: Tipe paling standar di mana perintah di `ExecStart` menjadi proses utama.
  - `User=ubuntu`: Jalankan aplikasi dengan aman sebagai user biasa.
  - `ExecStart`: Alamat lengkap biner dan script yang ingin dijalankan (wajib menggunakan *Absolute Path*!).
  - `Restart=on-failure`: **Fitur Self-Healing!** Jika aplikasi mati karena error (exit code bukan 0), systemd otomatis menghidupkannya lagi.
  - `RestartSec=5s`: Tunggu jeda 5 detik sebelum menyalakan ulang agar tidak membanjiri CPU.
- **`[Install]`**:
  - `WantedBy=multi-user.target`: Tandai agar service ini diikutsertakan saat server masuk ke mode operasional multi-pengguna biasa.

### Langkah 3: Muat Ulang Systemd & Nyalakan Layanan!
Setiap kali kamu membuat atau mengedit file di `/etc/systemd/system/`, kamu **WAJIB** memberitahu systemd agar membaca ulang konfigurasinya dari disk:

```bash
# 1. Muat ulang konfigurasi systemd (WAJIB!):
sudo systemctl daemon-reload

# 2. Nyalakan dan aktifkan auto-start:
sudo systemctl enable --now myapp.service

# 3. Periksa status kesehatannya:
sudo systemctl status myapp.service

# 4. Intip log-nya secara live dan saksikan bagaimana systemd otomatis 
#    menghidupkannya kembali saat aplikasi sengaja crash di hitungan ke-10!
journalctl -u myapp.service -f
```
*(Tekan `Ctrl + C` untuk berhenti melihat log).*

---

## 9. Penjadwalan Tugas: `systemd timer` vs `cron`

Seringkali kita perlu menjalankan tugas secara berkala (misal: jalankan script backup database setiap jam 2 pagi, atau cek kesehatan server tiap 5 menit). Ada dua cara melakukannya di Linux:

### Cara 1: `cron` (Pola Warisan Klasik)
Cron adalah alat penjadwal legendaris yang sudah ada sejak era Unix.
Untuk mengatur jadwal cron milikmu, ketik:
```bash
crontab -e
```

Format tabel waktu cron terdiri dari **5 Bintang**:
```text
 ┌───────────── Menit (0 - 59)
 │ ┌────────────── Jam (0 - 23)
 │ │ ┌─────────────── Hari dalam Bulan (1 - 31)
 │ │ │ ┌──────────────── Bulan (1 - 12)
 │ │ │ │ ┌───────────────── Hari dalam Minggu (0 - 6, 0 = Minggu)
 │ │ │ │ │
 * * * * *  /perintah/yang/dijalankan
```

Contoh umum:
```text
30 2 * * *    /usr/local/bin/backup.sh      # Berjalan setiap hari pada pukul 02:30 malam
*/10 * * * *  /usr/local/bin/cek-health.sh   # Berjalan setiap kelipatan 10 menit
```

> ⚠️ **Jebakan Maut Cron bagi Pemula:**
> Mengapa script yang kamu jalankan manual di terminal sukses, tapi saat dijalankan lewat Cron **selalu gagal/tidak bereaksi**?
> Karena **Cron TIDAK memuat environment shell biasamu**! Variabel `$PATH` di Cron sangat minim, sehingga perintah seperti `python3` atau `aws` tidak ditemukan.
> **Solusi:** Selalu tulis *Absolute Path* secara lengkap di dalam script cron (misal `/usr/bin/python3`, bukan hanya `python3`), atau definisikan `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` di baris paling atas file crontab.

---

### Cara 2: `systemd timer` (Pola Modern Standar SRE)
Di lingkungan produksi modern, SRE lebih menyukai **`systemd timer`** daripada Cron biasa.
`systemd timer` selalu terdiri dari sepasang file:
1. File **`.service`**: Berisi *apa* yang harus dieksekusi (bertipe `Type=oneshot`).
2. File **`.timer`**: Berisi *kapan* jadwal waktu tugas tersebut dieksekusi.

Contoh file `/etc/systemd/system/backup.timer`:
```ini
[Unit]
Description=Jadwal Rutin Backup Server

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

**Mengapa SRE Lebih Suka `systemd timer`?**
1. **Ada Log Terpusat:** Output script otomatis tersimpan dan bisa dilacak via `journalctl -u backup.service`.
2. **Fitur `Persistent=true`:** Jika server mati/mati lampu pada pukul 02:30 malam, begitu server dinyalakan kembali jam 06:00 pagi, systemd tahu bahwa jadwal tadi malam terlewat dan akan **langsung mengeksekusi backup saat itu juga**! Cron biasa tidak bisa melakukan ini.
3. **Mudah Dipantau:** Kamu bisa melihat daftar jadwal yang akan datang cukup dengan satu perintah:
   ```bash
   systemctl list-timers
   ```

---

## 10. Pola Kegagalan (Failure Modes) di Production & Cara Diagnosanya

Berikut adalah 4 masalah insiden nyata terkait proses dan service yang paling sering terjadi di lingkungan kerja SRE:

| Masalah Insiden | Gejala & Error | Akar Masalah & Solusi SRE |
|---|---|---|
| **Service Gagal Start (CrashLoop)** | Status service: `activating (auto-restart)` atau `failed` dengan status `exit-code`. | **Penyebab:** Kesalahan sintaks file konfigurasi, path binary salah, atau folder kerja tidak ada.<br>**Solusi:** Jangan panik! Langsung buka log detail kegagalan dengan:<br>`journalctl -u nama-service -n 50 --no-pager` |
| **Port Tabrakan (*Address Already in Use*)** | Service web (Nginx/App) gagal start dengan log: `bind() to 0.0.0.0:80 failed (98: Address already in use)`. | **Penyebab:** Port `80` sudah terlanjur dikuasai oleh proses lain (atau ada instance lama yang macet).<br>**Solusi:** Lacak siapa pemilik port tersebut dengan:<br>`sudo ss -tulpn \| grep :80`<br>Lalu matikan PID proses yang menyerobot port tersebut. |
| **Server Tidak Responsif (*OOM Killer*)** | Salah satu proses penting tiba-tiba menghilang misterius tanpa pesan error di aplikasi. | **Penyebab:** Server kehabisan memori RAM fisik, sehingga mekanisme pertahanan Linux Kernel bernama **OOM (Out Of Memory) Killer** turun tangan membunuh proses yang paling rakus RAM demi menyelamatkan server dari *freeze*.<br>**Solusi:** Cek bukti eksekusi mati oleh Kernel dengan:<br>`sudo dmesg -T \| grep -i oom` |
| **Service Gagal Setelah Diedit** | Menjalankan `systemctl restart app` memunculkan peringatan merah: `Warning: The unit file has changed on disk`. | **Penyebab:** File unit di `/etc/systemd/system/` baru saja kamu ubah, tapi systemd masih memegang konfigurasi lama di memorinya.<br>**Solusi:** Jalankan:<br>`sudo systemctl daemon-reload`<br>lalu restart kembali layanannya. |

---

## 11. Latihan Praktik Mandiri (10 Menit di Terminal VM OrbStack)

Ayo uji pemahamanmu dengan melakukan drill simulasi di terminal:

```bash
# 1. Buka dua tab terminal. Di Tab 1, jalankan proses tidur buatan selama 500 detik:
sleep 500 &
PID_SAYA=$!
echo "Proses saya lahir dengan PID: $PID_SAYA"

# 2. Periksa keberadaan proses tersebut menggunakan ps:
ps aux | grep [s]leep

# 3. Kirim sinyal penghentian sopan (SIGTERM):
kill -TERM $PID_SAYA

# 4. Periksa kembali apakah proses sudah lenyap dengan damai:
ps aux | grep [s]leep

# 5. Cari 3 service apa saja yang saat ini statusnya gagal (failed) di servermu:
systemctl --failed

# 6. Lihat daftar seluruh jadwal timer otomatis yang aktif di server:
systemctl list-timers

# 7. Intip 30 baris log kernel sistem terbaru:
journalctl -k -n 30 --no-pager
```

---

## 12. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Lihat seluruh proses aktif di server | `ps aux` |
| Filter proses tertentu berdasarkan nama | `ps aux \| grep nama_app` |
| Tampilkan proses paling rakus CPU | `ps -eo pid,user,%cpu,%mem,cmd --sort=-%cpu \| head -n 10` |
| Monitor kinerja CPU/RAM real-time | `top` (atau `htop`) |
| Hentikan proses secara sopan (SIGTERM) | `kill <PID>` atau `pkill -f nama_app` |
| Matikan proses secara paksa (Langkah Terakhir) | `kill -9 <PID>` |
| Minta aplikasi reload konfigurasi tanpa downtime | `kill -HUP <PID>` |
| Cek status kesehatan layanan | `systemctl status nama_service` |
| Nyalakan & pasang auto-start layanan sekaligus | `sudo systemctl enable --now nama_service` |
| Muat ulang konfigurasi systemd setelah edit file unit | `sudo systemctl daemon-reload` |
| Pantau log layanan secara live | `journalctl -u nama_service -f` |
| Cek daftar seluruh timer penjadwal | `systemctl list-timers` |

---

## 13. Cek Pemahaman (Self-Check)

Uji pemahamanmu sebelum membuka kunci jawaban di bawah:

1. **Mengapa seorang SRE tidak boleh langsung menggunakan `kill -9` saat ingin mematikan aplikasi yang lambat atau bermasalah?**
2. **Apa perbedaan antara `systemctl stop app` dan `systemctl disable app`?**
3. **Mengapa setelah kita mengedit file `/etc/systemd/system/myapp.service`, kita wajib mengetik `sudo systemctl daemon-reload` sebelum me-restart layanannya?**
4. **Saat menginvestigasi performa CPU di perintah `top`, apa arti metrik `wa` (I/O Wait), dan apa kesimpulanmu jika angka `wa` mencapai 35% padahal `%Cpu id` (idle) masih tinggi?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **Bahaya `kill -9` (SIGKILL):**
   - Sinyal `SIGKILL` (9) mematikan proses seketika lewat Kernel tanpa memberi kesempatan bagi aplikasi untuk melakukan pembersihan (*graceful cleanup*). Hal ini bisa menyebabkan data transaksi database korup, file yang sedang ditulis rusak setengah jalan, atau meninggalkan *file lock* yang mencegah aplikasi untuk bisa dinyalakan kembali. Selalu gunakan `SIGTERM` (15) terlebih dahulu.

2. **`systemctl stop` vs `systemctl disable`:**
   - **`systemctl stop`** mematikan layanan pada **detik ini juga**, namun jika server di-reboot, layanan tersebut akan tetap hidup kembali otomatis jika sebelumnya berstatus `enabled`.
   - **`systemctl disable`** mencabut konfigurasi otomatis saat booting, sehingga saat server dinyalakan ulang di masa depan, layanan tersebut **tidak akan berjalan otomatis**.

3. **Fungsi `sudo systemctl daemon-reload`:**
   - `systemd` membaca dan menyimpan seluruh konfigurasi file unit ke dalam memori RAM saat pertama kali booting. Ketika kamu mengubah file unit di harddisk, systemd tidak otomatis mengetahuinya. `daemon-reload` memerintahkan systemd untuk membaca ulang seluruh file konfigurasi dari disk ke memorinya.

4. **Arti Metrik `wa` (I/O Wait):**
   - `wa` menunjukkan persentase waktu prosesor (CPU) terpaksa menganggur hanya untuk **menunggu respons baca/tulis dari perangkat penyimpanan (Disk/Storage)**.
   - Jika `wa` mencapai 35%, kesimpulannya server lambat **bukan** karena aplikasi kekurangan daya prosesor, melainkan karena ada *bottleneck* pada harddisk yang sedang kewalahan (misal disk lambat, harddisk rusak, atau aplikasi menulis log/database secara berlebihan).

</details>
