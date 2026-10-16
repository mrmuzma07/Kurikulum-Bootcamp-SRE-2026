# 06 — SSH (Secure Shell), File Transfer, dan Tunneling

> **Ringkasan:** Menguasai cara masuk dan mengendalikan server dari jarak jauh secara aman tanpa password menggunakan **SSH Key (Ed25519)**, mengelola puluhan server lewat **`~/.ssh/config`** dan **Bastion Host (`ProxyJump`)**, menyalin file secara efisien dengan **`scp` / `rsync`**, membuat terowongan jaringan (**SSH Tunneling `-L`**), serta mengamankan server SSH di production.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Memahami cara kerja kriptografi **Public Key (Gembok)** dan **Private Key (Anak Kunci)** serta membuat pasangan kunci modern **`ed25519`**.
- Mengatur hak akses (*permission*) folder `~/.ssh` dengan benar agar tidak ditolak oleh sistem keamanan SSH.
- Memasang kunci ke server (`ssh-copy-id`) sehingga bisa login tanpa mengetik password.
- Membuat buku alamat pintar di **`~/.ssh/config`** termasuk melompat melewati server gerbang (**Bastion / `ProxyJump`**).
- Menyalin file dan menyinkronkan folder antar server menggunakan **`scp`** dan **`rsync`**.
- Membuka akses ke layanan internal server (seperti Database atau Dashboard pribadi) dari laptop menggunakan **SSH Local Port Forwarding (`-L`)**.
- Mengamankan konfigurasi SSH server (`/etc/ssh/sshd_config`) dan mendiagnosa masalah login dengan `ssh -vvv`.

---

## 1. Mindset Dasar: Apa Itu SSH dan Mengapa SRE Hidup di Dalamnya?

Di dunia nyata, server production perusahaan diletakkan di dalam ruang *Data Center* yang dingin, terkunci rapat, dan mungkin berjarak ratusan kilometer dari tempat dudukmu. Kita tidak pernah mencolokkan layar monitor atau keyboard fisik ke server tersebut.

Untuk mengendalikan server dari jarak jauh, kita menggunakan **SSH (*Secure Shell*)** yang secara standar berjalan di **Port 22**.
- **Apa yang dilakukan SSH?** SSH membangun sebuah "terowongan pipa terenkripsi (anti-sadap)" antara terminal di laptop MacBook kamu dengan server Linux tujuan. Semua ketikan perintah dan data yang lewat di dalamnya diacak secara kriptografi sehingga tidak bisa diintip oleh pihak ketiga di jaringan.
- **Bukan Cuma untuk Manusia!** Di **Fase 4 (Ansible)** nanti, ketika kamu mengotomasi instalasi puluhan server dan cluster Kubernetes sekaligus, Ansible bekerja 100% tanpa agen tambahan dengan cara menumpang lewat jalur SSH yang kamu pelajari di bab ini!

---

## 2. Login Tanpa Password: Analogi Gembok (Public Key) & Anak Kunci (Private Key)

Saat baru mengenal server, pemula biasanya login menggunakan **Password**. Namun di lingkungan production SRE, **login ke server menggunakan password sangat dilarang!** Mengapa?
1. Di internet, ribuan robot peretas (*botnet*) terus-menerus mengetuk Port 22 di seluruh dunia untuk menebak password (*Brute-Force Attack*) 24 jam sehari.
2. Alat otomasi seperti Ansible tidak mungkin mengetik password secara manual ke 100 server satu per satu.

Sebagai gantinya, kita menggunakan **SSH Key-Based Authentication** (Otentikasi Berbasis Pasangan Kunci).

### Analogi Gembok dan Anak Kunci
Saat kamu membuat SSH Key di laptopmu, komputer menghasilkan **sepasang file kembar yang saling berpasangan secara matematis**:

```text
  [ Di Laptop MacBook Kamu ]                        [ Di Server Linux Tujuan ]
  ~/.ssh/id_ed25519                                 ~/.ssh/authorized_keys
  (PRIVATE KEY = ANAK KUNCI EMAS)                   (Daftar PUBLIC KEY = GEMBOK)
          │                                                       ▲
          │      Saat kamu mengetik 'ssh ubuntu@server':          │
          └────► Anak Kunci di laptopmu mencocokkan diri ─────────┘
                 secara matematis dengan Gembok di server!
```

1. **Public Key (`id_ed25519.pub` — Akhiran `.pub`) = GEMBOK:**
   - Sifatnya **Publik (Boleh dilihat siapa saja)**.
   - Tugasmu adalah menyalin "Gembok" ini dan memasangnya di pintu server mana saja yang ingin kamu masuki (disimpan di dalam file `~/.ssh/authorized_keys` milik server tujuan).
2. **Private Key (`id_ed25519` — TANPA `.pub`) = ANAK KUNCI PRIBADI:**
   - Sifatnya **SANGAT RAHASIA (Seperti PIN ATM / Kunci Rumah Asli)**.
   - File ini **TIDAK BOLEH pernah keluar dari laptopmu**, tidak boleh dikirim lewat chat, dan tidak boleh di-upload ke Git! Siapa pun yang mencuri file Private Key ini bisa langsung masuk ke seluruh server yang dipasangi gembokmu.

---

## 3. Membuat & Memasang SSH Key ke Server

### Langkah 1: Membuat Pasangan Kunci (`ssh-keygen`)
Kita menggunakan algoritma modern **`ed25519`** (jauh lebih pendek, lebih cepat, dan lebih kuat dibandingkan algoritma `rsa` generasi lama):

```bash
ssh-keygen -t ed25519 -C "macbook-sre-2026"
```
Saat perintah di atas dijalankan, terminal akan bertanya:
1. `Enter file in which to save the key`: Tekan **`Enter`** saja (agar disimpan di lokasi standar `~/.ssh/id_ed25519`).
2. `Enter passphrase`: Kamu boleh mengisi kata sandi pengaman anak kunci, atau tekan **`Enter`** (kosongkan) untuk keperluan lab agar bisa login otomatis tanpa ketik apa-apa.

Mari kita cek dua file yang baru saja lahir di dalam folder `~/.ssh/`:
```bash
ls -la ~/.ssh/
# id_ed25519      <-- JANGAN PERNAH DIBUKA KE ORANG LAIN (Private Key / Anak Kunci)
# id_ed25519.pub  <-- Boleh dilihat dan disalin ke server (Public Key / Gembok)

cat ~/.ssh/id_ed25519.pub
# Output contoh:
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGx... macbook-sre-2026
```

---

### Langkah 2: Aturan Ketat Hak Akses (`chmod`) Folder `~/.ssh`
SSH adalah program yang sangat disiplin soal keamanan. **Jika izin (*permission*) folder `~/.ssh` atau file kuncimu terlalu longgar (bisa dibaca/diedit oleh user lain), SSH akan langsung menolak bekerja!**

Pastikan izinnya selalu diatur seperti tabel standar ini:

| File / Folder | Izin Wajib (`chmod`) | Bentuk `ls -l` | Mengapa Wajib Seperti Ini? |
|---|:---:|:---:|---|
| **`~/.ssh`** | **`700`** | `drwx------` | Hanya kamu sendiri yang boleh masuk ke folder penyimpanan kunci. |
| **`~/.ssh/id_ed25519`** (Private Key) | **`600`** | `-rw-------` | Anak kunci rahasia hanya boleh dibaca oleh pemiliknya sendiri! |
| **`~/.ssh/id_ed25519.pub`** (Public Key) | **`644`** | `-rw-r--r--` | Gembok publik boleh dibaca umum. |
| **`~/.ssh/config`** (Di Laptop) | **`600`** | `-rw-------` | Buku konfigurasi SSH hanya boleh diakses pemiliknya. |
| **`~/.ssh/authorized_keys`** (Di Server) | **`600`** | `-rw-------` | Agar orang lain di server tidak bisa menyisipkan gembok palsu ke akunmu! |

```bash
# Perintah penyelamat jika suatu saat SSH mengeluh soal permission:
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

---

### Langkah 3: Memasang Gembok (Public Key) ke Server Tujuan (`ssh-copy-id`)
Bagaimana cara menaruh isi `id_ed25519.pub` kita ke dalam file `~/.ssh/authorized_keys` di server tujuan?

**Cara 1 — Otomatis & Paling Aman (Menggunakan `ssh-copy-id`):**
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.64.5
```
*(Masukkan password user `ubuntu` untuk terakhir kalinya. `ssh-copy-id` akan otomatis membuatkan folder `~/.ssh`, mengatur izin `700`/`600`, dan menempelkan gembokmu di sana!).*

**Cara 2 — Manual lewat Pipa (Jika di sistemmu tidak ada `ssh-copy-id`):**
```bash
cat ~/.ssh/id_ed25519.pub | ssh ubuntu@192.168.64.5 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

**Sekarang, uji login kembali:**
```bash
ssh ubuntu@192.168.64.5
```
Kamu akan langsung masuk ke dalam server tanpa ditanya password lagi!

> 💡 **Apa arti pertanyaan *"The authenticity of host '...' can't be established"* saat pertama kali SSH?**
> Saat kamu menghubungi sebuah server untuk pertama kalinya, SSH akan menampilkan sidik jari (*fingerprint*) server tersebut dan bertanya: `Are you sure you want to continue connecting (yes/no/[fingerprint])?`
> Ketika kamu mengetik **`yes`**, SSH mencatat sidik jari server tersebut ke dalam file **`~/.ssh/known_hosts`** di laptopmu. Tujuannya: jika besok-besok ada peretas di tengah jalan yang menyamar menggunakan IP yang sama (*Man-in-the-Middle Attack*), SSH akan mendeteksi bahwa sidik jari servernya berubah dan langsung memperingatkanmu!

---

## 4. Buku Kontak Pintar (`~/.ssh/config`) & Bastion Host (`ProxyJump`)

Bayangkan seorang SRE yang harus mengelola 30 server dengan IP, port, dan kunci yang berbeda-beda. Menghafal dan mengetik `ssh -i ~/.ssh/kunci_prod -p 2222 admin@10.20.30.45` setiap kali mau login tentu sangat menyiksa!

Solusinya: Buatlah file **`~/.ssh/config`** di laptopmu sebagai buku kontak alias pintar.

### Contoh Isi File `~/.ssh/config`
```ssh-config
# 1. Alias untuk VM Lab Harian di OrbStack
Host lab01
    HostName 192.168.64.5
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

# 2. Gerbang Depan Production (Bastion / Jump Host)
Host bastion
    HostName 203.0.113.10
    User sre_admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

# 3. Server Internal di Dalam Data Center (Otomatis lompat lewat Bastion!)
Host prod-db-01
    HostName 10.0.10.55
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion

# 4. Pengaturan Global untuk SEMUA koneksi (*) agar tidak mudah putus/freeze:
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```
*(Jangan lupa jalankan `chmod 600 ~/.ssh/config` setelah membuatnya).*

Dengan konfigurasi di atas:
- Kamu cukup mengetik **`ssh lab01`** untuk langsung masuk ke `ubuntu@192.168.64.5`!
- Apa fungsi **`ServerAliveInterval 60`**? Setiap 60 detik, laptopmu akan mengirim sinyal "detak jantung" kecil ke server agar koneksi SSH tidak diputus otomatis oleh router/Wi-Fi ketika kamu sedang diam membaca layar.

---

### Konsep Arsitektur On-Premises: Bastion Host & `ProxyJump`
Di perusahaan yang menerapkan standar keamanan tinggi, server penting (seperti Database atau Worker Kubernetes) **dilarang memiliki IP Publik**. Mereka disembunyikan di jaringan privat internal (`10.0.x.x`).
Satu-satunya pintu masuk dari luar hanyalah melewati satu server penjaga khusus yang diperkuat keamanannya, yang disebut **Bastion Host (Jump Host)**.

```text
[ Laptop Mac Kamu ] ──(SSH)──► [ Bastion Host ] ──(SSH Internal)──► [ Server Database ]
                                (203.0.113.10)                         (10.0.10.55)
```

Dulu, engineer harus login ke Bastion dulu, baru dari dalam Bastion mengetik SSH lagi ke server dalam.
Sekarang, berkat baris **`ProxyJump bastion`** di dalam `~/.ssh/config` (atau opsi `ssh -J bastion ubuntu@10.0.10.55`), kamu cukup mengetik **`ssh prod-db-01`** dari laptop Mac-mu, dan SSH akan **otomatis melompat melewati Bastion secara transparan dan aman**!

---

## 5. Eksekusi Jarak Jauh & Transfer File (`scp` vs `rsync`)

### A. Menjalankan Perintah Tanpa Masuk ke Dalam Server
Kamu tidak perlu masuk ke dalam server hanya untuk mengecek sisa disk atau me-restart service. Cukup tambahkan perintahnya dalam tanda kutip di ujung perintah `ssh`:
```bash
ssh lab01 "hostname && uptime && df -h /"
```
Perintah tersebut akan dijalankan di dalam `lab01`, hasilnya ditampilkan di layar laptopmu, lalu koneksi langsung ditutup kembali!

---

### B. Menyalin File dengan `scp` (Secure Copy)
Untuk menyalin satu atau dua file secara cepat lewat jalur SSH, gunakan `scp`:
```bash
# 1. Upload (Dari Laptop ke Server lab01 di folder /tmp/):
scp laporan.txt lab01:/tmp/

# 2. Download (Dari Server lab01 ke folder saat ini '.' di Laptop):
scp lab01:/var/log/syslog ./syslog-backup.log

# 3. Menyalin satu folder penuh beserta isinya (-r = recursive):
scp -r folder_config/ lab01:/tmp/folder_config/
```

---

### C. Menyinkronkan Folder Besar dengan `rsync` (Andalan Utama SRE!)
Jika kamu ingin memindahkan folder berukuran besar (misal ratusan file atau data bergiga-gigabyte), **jangan gunakan `scp` — gunakanlah `rsync`!**

**Mengapa `rsync` jauh lebih hebat daripada `scp`?**
1. **Hanya Mengirim Selisih Perubahan (*Delta Transfer*):** Jika di dalam folder ada 1.000 file dan hanya 2 file yang baru berubah, `rsync` hanya akan mengirim 2 file yang berubah tersebut! (`scp` akan menyalin ulang ke-1.000 file dari awal).
2. **Bisa Melanjutkan Transfer yang Putus (*Resume*):** Jika koneksi internet putus saat menyalin file besar, menjalankan ulang `rsync -avzP` akan melanjutkan dari titik terakhir.

```bash
# Rumus Standar SRE: rsync -avzP [sumber] [tujuan]
# -a (archive: jaga permission & waktu), -v (verbose), -z (kompresi), -P (progress bar & resume)
rsync -avzP ./aplikasi/ lab01:/opt/aplikasi/

# Uji coba dulu tanpa benar-benar menyalin file (--dry-run atau -n):
rsync -avzn ./aplikasi/ lab01:/opt/aplikasi/
```

> ⚠️ **Perhatikan Garis Miring di Ujung Nama Folder Sumber (`/` — Trailing Slash) pada `rsync`:**
> - `rsync -avz ./aplikasi/ lab01:/opt/target/` *(Pakai `/` di ujung `./aplikasi/`)* $\longrightarrow$ Menyalin **ISI di dalam** folder `aplikasi` ke dalam `/opt/target/`.
> - `rsync -avz ./aplikasi lab01:/opt/target/` *(Tanpa `/`)* $\longrightarrow$ Membuat folder baru bernama `/opt/target/aplikasi/`.

---

## 6. Terowongan Ajaib: SSH Tunneling / Port Forwarding (`-L`, `-R`, `-D`)

Fitur paling keren dari SSH adalah kemampuannya membungkus lalu lintas port jaringan lain ke dalam terowongan SSH yang aman.

### 1. Local Port Forwarding (`ssh -L`) — *Menarik Layanan Internal Server ke Laptop Kita*
**Kasus Nyata SRE:**
Di dalam server `lab01`, ada layanan Database PostgreSQL (Port `5432`) atau dashboard web internal (Port `8080`) yang demi keamanan **hanya mendengarkan di `127.0.0.1` (Localhost server)** dan diblokir oleh firewall dari luar.
Bagaimana caranya agar kamu bisa membuka layanan tersebut langsung dari browser atau aplikasi di laptop MacBook kamu tanpa membuka firewall server?

Gunakan **`ssh -L` (Local Forwarding)**!

```text
Rumus:   ssh -L [Port_Lokal_di_Mac]:[IP_Tujuan_Dari_Sisi_Server]:[Port_Tujuan] [Nama_Server_SSH]
Contoh:  ssh -L 9000:127.0.0.1:8080 lab01
```

```text
[ Browser di Laptop Mac ]                         [ Di Dalam Server lab01 ]
Buka: http://localhost:9000                       Aplikasi Web Internal
           │                                      (127.0.0.1:8080)
           ▼                                             ▲
    (Masuk ke Pintu 9000)                                │
           │         ═══ Terowongan Aman SSH ═══         │
           └─────────────────────────────────────────────┘
```
Selama koneksi `ssh -L 9000:127.0.0.1:8080 lab01` itu menyala:
Setiap kali kamu membuka **`http://localhost:9000` di laptop Mac-mu**, paketnya akan diselundupkan lewat terowongan SSH langsung menuju **`127.0.0.1:8080` di dalam server `lab01`**!

*(Tips: Tambahkan opsi **`-fN`** jika kamu hanya ingin membuka terowongan di background tanpa perlu masuk ke layar terminal server: `ssh -fN -L 9000:127.0.0.1:8080 lab01`).*

### 2. Sekilas tentang Remote Forwarding (`-R`) dan Dynamic Proxy (`-D`)
- **`ssh -R 9000:127.0.0.1:3000 lab01` (*Remote Forwarding*):** Kebalikan dari `-L`. Membuka port `9000` di sisi server `lab01` yang jika diakses orang, trafiknya diarahkan masuk ke aplikasi yang sedang berjalan di port `3000` laptopmu.
- **`ssh -D 1080 lab01` (*Dynamic SOCKS5 Proxy*):** Mengubah server `lab01` menjadi server Proxy SOCKS5 di port `1080` laptopmu. Jika browser laptopmu diarahkan ke SOCKS5 proxy `localhost:1080`, maka seluruh aktivitas browsing internetmu akan keluar menggunakan alamat IP milik server `lab01`.

---

## 7. Mengamankan Server SSH di Production (*SSH Hardening*)

Sebagai SRE, begitu kamu selesai memasang SSH Key di server baru, kamu **wajib mengunci pengaturan server SSH** yang berada di file **`/etc/ssh/sshd_config`** agar tahan dari serangan peretas:

```ini
# Pengaturan Rekomendasi Production di /etc/ssh/sshd_config:
PubkeyAuthentication yes          # Izinkan masuk dengan SSH Key (Gembok)
PasswordAuthentication no         # MATIKAN login menggunakan password biasa!
PermitRootLogin no                # LARANG orang login langsung sebagai user 'root'!
```

> 🛡️ **Prosedur Aman SRE Saat Mengubah `/etc/ssh/sshd_config` (Agar Tidak Terkunci di Luar!):**
> 1. Pastikan kamu **sudah berhasil login menggunakan SSH Key** tanpa password.
> 2. Setelah mengedit `/etc/ssh/sshd_config`, **cek dulu apakah ada salah ketik sintaks** dengan perintah:
>    ```bash
>    sudo sshd -t
>    ```
>    *(Jika tidak muncul pesan error apa-apa, berarti sintaks aman!).*
> 3. Muat ulang layanan SSH:
>    ```bash
>    sudo systemctl reload ssh
>    ```
> 4. **JANGAN TUTUP JENDELA TERMINAL SSH LAMA KAMU!** Buka tab terminal baru di laptopmu, lalu coba login lagi. Jika gagal, kamu masih punya tab terminal lama yang masih terbuka untuk memperbaiki kesalahannya!

---

## 8. Pola Kegagalan (Failure Modes) & Diagnosa dengan `ssh -vvv`

Jika kamu gagal login SSH dan pesannya tidak jelas, tambahkan opsi **`-vvv` (*Very Verbose*)**:
```bash
ssh -vvv ubuntu@192.168.64.5
```
SSH akan mencetak langkah demi langkah file kunci mana yang ia coba dan di tahap mana server menolakmu.

Berikut adalah 4 masalah SSH paling umum beserta solusinya:

| Pesan Error | Penyebab Utama | Solusi Cepat SRE |
|---|---|---|
| **`Permission denied (publickey)`** | 1. Public key belum terpasang di `~/.ssh/authorized_keys` server.<br>2. Nama user salah (misal harusnya `ubuntu@...` tapi mengetik `root@...`).<br>3. Izin folder `~/.ssh` (`700`) atau `authorized_keys` (`600`) di server salah! | Cek `ssh -vvv`, pastikan user benar, dan perbaiki `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys` di server. |
| **`WARNING: UNPROTECTED PRIVATE KEY FILE!`** | Izin file Private Key (`id_ed25519`) di laptopmu terlalu terbuka (misal `644`). | Jalankan di laptopmu:<br>`chmod 600 ~/.ssh/id_ed25519` |
| **`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`** | Server baru saja di-install ulang (atau IP yang sama dipakai oleh VM baru), sehingga sidik jari server berbeda dengan catatan lama di `~/.ssh/known_hosts`. | Jika kamu yakin servernya memang habis dibuat ulang, hapus catatan sidik jari lama dengan:<br>`ssh-keygen -R <IP_atau_Hostname_Server>` |
| **`Too many authentication failures`** | Kamu memiliki terlalu banyak file SSH Key di dalam `~/.ssh/`, sehingga SSH mencoba semuanya satu per satu sampai batas percobaan server habis. | Tambahkan baris `IdentitiesOnly yes` dan `IdentityFile ~/.ssh/kunci_yang_benar` di dalam `~/.ssh/config`. |

---

## 9. Latihan Praktik Mandiri (10 Menit di Terminal)

Mari kita uji pembuatan kunci, konfigurasi alias, transfer file, dan tunneling di dalam lingkungan latihanmu:

```bash
# 1. Buat pasangan kunci SSH khusus latihan tanpa passphrase (-N "")
ssh-keygen -t ed25519 -C "latihan-sre" -f ~/.ssh/id_ed25519_latihan -N ""

# 2. Periksa kedua file yang terbentuk dan pastikan izinnya benar (600 dan 644)
ls -l ~/.ssh/id_ed25519_latihan*

# 3. Pasang public key latihan tersebut ke akun lokal kita sendiri (localhost)
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat ~/.ssh/id_ed25519_latihan.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# 4. Uji login SSH ke localhost menggunakan kunci baru tersebut tanpa password:
ssh -i ~/.ssh/id_ed25519_latihan -o StrictHostKeyChecking=accept-new $(whoami)@localhost "echo 'Login SSH tanpa password BERHASIL di host:' \$(hostname)"

# 5. Uji menyalin file menggunakan 'rsync' lewat jalur SSH ke localhost:
mkdir -p ~/latihan-ssh/sumber
echo "Data konfigurasi penting" > ~/latihan-ssh/sumber/config.ini
rsync -avz -e "ssh -i ~/.ssh/id_ed25519_latihan" ~/latihan-ssh/sumber/ $(whoami)@localhost:~/latihan-ssh/tujuan/
ls -la ~/latihan-ssh/tujuan/

# 6. Uji SSH Local Port Forwarding (-L):
#    Nyalakan web server lokal di port 8123:
python3 -m http.server 8123 --bind 127.0.0.1 &
WEB_PID=$!

#    Buat tunnel SSH dari port 9123 menuju port 8123 di background (-fN):
ssh -i ~/.ssh/id_ed25519_latihan -fN -L 9123:127.0.0.1:8123 $(whoami)@localhost

#    Buktikan kita bisa mengakses web server tersebut lewat pintu terowongan 9123:
curl -I http://localhost:9123

# 7. Bersihkan proses dan file latihan
kill -TERM $WEB_PID
pkill -f "9123:127.0.0.1:8123"
rm -rf ~/latihan-ssh ~/.ssh/id_ed25519_latihan*
```

---

## 10. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Buat pasangan SSH Key baru (`ed25519`) | `ssh-keygen -t ed25519 -C "komentar"` |
| Pasang Public Key ke server tujuan | `ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host` |
| Masuk ke server via SSH / jalankan 1 perintah | `ssh user@host` / `ssh user@host "uptime"` |
| Melompat melewati Bastion Host | `ssh -J user@bastion user@internal-ip` (atau pakai `ProxyJump` di `~/.ssh/config`) |
| Diagnosa detail saat gagal login SSH | `ssh -vvv user@host` |
| Hapus catatan sidik jari server lama yang berubah | `ssh-keygen -R <IP_atau_Hostname>` |
| Salin 1 file cepat lewat SSH | `scp file.txt user@host:/tmp/` |
| Sinkronisasi folder dengan kompresi & progress bar | `rsync -avzP ./lokal/ user@host:/target/` |
| Buat terowongan Local Port Forwarding | `ssh -L 8080:127.0.0.1:5432 user@host` |
| Cek validitas sintaks `/etc/ssh/sshd_config` | `sudo sshd -t` |

---

## 11. Cek Pemahaman (Self-Check)

Jawab pertanyaan berikut dengan kata-katamu sendiri sebelum membuka kunci jawaban di bawah:

1. **Antara file `id_ed25519` dan `id_ed25519.pub`, manakah yang harus disalin ke dalam server tujuan (`~/.ssh/authorized_keys`), dan manakah yang haram dibagikan ke orang lain?**
2. **Mengapa SSH menolak login jika izin (*permission*) pada folder `~/.ssh` atau file `~/.ssh/authorized_keys` di server diubah menjadi `777`?**
3. **Di dalam server `prod-server`, database PostgreSQL berjalan di `127.0.0.1:5432`. Tuliskan perintah `ssh -L` agar kamu bisa mengakses database tersebut dari laptopmu melalui `localhost:15432`!**
4. **Mengapa saat menyinkronkan folder berukuran 5 GB yang berisi ribuan file ke server, menggunakan `rsync -avzP` jauh lebih baik dibandingkan `scp -r`?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **`id_ed25519` vs `id_ed25519.pub`:**
   - Yang disalin ke dalam `~/.ssh/authorized_keys` di server tujuan adalah **`id_ed25519.pub` (Public Key / Gembok)**.
   - Sedangkan **`id_ed25519` (Private Key / Anak Kunci)** wajib dijaga kerahasiaannya di laptop sendiri dan haram dibagikan ke siapa pun.

2. **Mengapa izin `777` ditolak oleh SSH?**
   - Jika `~/.ssh` atau `authorized_keys` memiliki izin `777` (semua user di server boleh menulis/mengubahnya), maka pengguna lain di server tersebut bisa diam-diam menambahkan Public Key milik mereka ke dalam `authorized_keys` kamu dan mengambil alih akunmu tanpa izin. Demi mencegah celah tersebut, daemon `sshd` wajib memastikan folder `~/.ssh` berizin `700` dan `authorized_keys` berizin `600`.

3. **Perintah `ssh -L` untuk menarik port Database `5432` ke port lokal `15432`:**
   ```bash
   ssh -L 15432:127.0.0.1:5432 user@prod-server
   ```
   *(Urutannya: `-L [Port_Lokal_Laptop]:[Alamat_Tujuan_di_Server]:[Port_Tujuan_di_Server]`).*

4. **Keunggulan `rsync -avzP` dibanding `scp -r`:**
   - `rsync` menggunakan algoritma *delta-transfer* yang hanya mengirimkan file atau bagian blok data yang berubah saja (tidak menyalin ulang seluruh 5 GB jika sebagian sudah ada), mendukung kompresi saat transit (`-z`), menjaga hak akses dan symlink asli (`-a`), serta dapat melanjutkan proses transfer yang terputus di tengah jalan tanpa harus mengulang dari nol (`-P`).

</details>
