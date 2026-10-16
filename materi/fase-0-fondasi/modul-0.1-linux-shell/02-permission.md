# 02 — Permission, User, dan Group di Linux

> **Ringkasan:** Memahami sistem keamanan hak akses di Linux (siapa yang boleh membaca, mengubah, dan menjalankan sebuah file atau folder), cara membaca output `ls -l`, menggunakan `chmod` dan `chown` dengan tepat, mengenal mekanisme `umask` dan `sudo`, serta mengatasi masalah *"Permission denied"* di server production.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Membaca 10 karakter izin pada output `ls -l` (seperti `-rwxr-xr--`) dengan lancar tanpa menebak-nebak.
- Memahami perbedaan mendasar antara izin `r`, `w`, `x` pada **File** dibandingkan pada **Direktori (Folder)**.
- Mengubah hak akses menggunakan perintah `chmod` baik dengan **Mode Angka (Octal)** maupun **Mode Huruf (Symbolic)**.
- Mengatur pemilik dan grup file/folder menggunakan `chown` dan `chgrp`.
- Memahami cara kerja `umask`, penggunaan `sudo` yang aman di production, serta izin khusus (*SUID, SGID, Sticky Bit*).
- Mendiagnosa dan memperbaiki error *"Permission denied"* (pada SSH key, script, maupun aplikasi server).

---

## 1. Mengapa Linux Sangat Ketat Soal Izin (Permission)?

Jika kamu menggunakan laptop pribadi sendirian, mungkin kamu merasa semua file di dalam laptop adalah milikmu dan bebas kamu buka.
Namun, **Linux sejak awal dirancang sebagai sistem operasi *Multi-User* (banyak pengguna)**. Di satu server production, bisa terdapat:
- Puluhan engineer (SRE, Developer, Database Admin) yang login via SSH.
- Belasan program/aplikasi yang berjalan bersamaan di latar belakang (Web Server Nginx, Database PostgreSQL, Agen Monitoring Prometheus).

> **Analogi Gedung Perkantoran & Kartu Akses:**
> Bayangkan server Linux seperti gedung kantor yang memiliki banyak ruangan.
> - Staf Keuangan boleh masuk ke Ruang Brankas, tapi Staf Tamu tidak boleh.
> - Semua karyawan boleh **membaca** Papan Pengumuman di lobi, tapi hanya bagian HRD yang boleh **mengubah/menulis** pengumuman tersebut.
>
> Tanpa aturan izin (*permission*) yang ketat, jika aplikasi web kamu tidak sengaja memiliki celah keamanan (*bug*), peretas yang masuk lewat aplikasi web tersebut bisa langsung membaca file password seluruh server atau menghapus database!

---

## 2. Siapa Kamu di Mata Linux? (User, Group, dan Others)

Setiap file dan folder di Linux selalu mengenali pengunjungnya ke dalam **3 kategori (Lapisan Pengguna)**:

```text
┌───────────────────────────────────────────────────────────────────┐
│  1. OWNER / USER (u)   : Pemilik sah file tersebut (1 orang)      │
│  2. GROUP (g)          : Rekan satu tim/divisi pemilik file       │
│  3. OTHERS (o)         : Orang asing (semua user lain di server)  │
└───────────────────────────────────────────────────────────────────┘
  * Tambahan: ALL (a) = Gabungan ketiganya (User + Group + Others)
```

### Mengenal Jenis-Jenis User di Server Linux
Di dalam server Linux, ada 3 jenis penghuni:
1. **`root` (Superuser / Raja Server):** Memiliki nomor identitas **`UID = 0`**. User `root` punya kekuasaan mutlak: ia bisa membaca, mengubah, dan menghapus file apa saja di seluruh server tanpa peduli izinnya apa.
2. **User Manusia (Regular User):** Akun milik engineer (misalnya `ubuntu`, `budi`, `alice`). Biasanya memiliki nomor `UID` mulai dari `1000` ke atas.
3. **User Aplikasi (Service Account / System User):** Akun khusus yang dibuat bukan untuk manusia login, melainkan untuk menjalankan program di latar belakang (misalnya user `www-data` untuk Nginx, `postgres` untuk database, `redis` untuk Redis). Jika aplikasi Redis disusupi peretas, peretas tersebut hanya terkurung di dalam izin milik user `redis` dan tidak bisa merusak sistem lain.

### Perintah untuk Mengecek Identitas Diri
Coba ketik perintah ini di terminalmu:
```bash
whoami              # Menampilkan nama user kamu saat ini (misal: ubuntu)
groups              # Menampilkan daftar grup/tim tempat kamu bergabung
id                  # Menampilkan KTP lengkapmu: UID (Nomor User), GID (Nomor Group), dan daftar grup
```
*Contoh Output `id`:*
```text
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),999(docker)
```
*(Artinya: Kamu adalah user `ubuntu` dengan nomor KTP `1000`, grup utamamu adalah `ubuntu`, dan kamu juga anggota grup `sudo` serta `docker`).*

---

## 3. Tiga Jenis Izin Dasar (`r`, `w`, `x`): File vs Direktori

Ada 3 hak akses dasar di Linux: **Read (`r`)**, **Write (`w`)**, dan **Execute (`x`)**.
**PERHATIKAN BAIK-BAIK:** Arti `r`, `w`, dan `x` pada **File Biasa** sedikit berbeda dengan artinya pada **Direktori (Folder)**! Ini adalah konsep yang paling sering menjebak pemula.

| Izin | Simbol | Arti pada **FILE BIASA** | Arti pada **DIREKTORI (FOLDER)** |
|---|---|---|---|
| **Read** (Baca) | **`r`** | Boleh membuka dan membaca isi file (misal dengan `cat` atau `less`). | Boleh melihat daftar nama file di dalam folder tersebut (dengan `ls`). |
| **Write** (Tulis) | **`w`** | Boleh mengubah/mengedit isi di dalam file (atau mengosongkannya). | **Boleh membuat file baru, menghapus file, atau mengganti nama file** di dalam folder tersebut! |
| **Execute** (Jalankan) | **`x`** | Boleh menjalankan file tersebut sebagai program atau script (misal `./deploy.sh`). | **Izin melintas/masuk ke dalam folder (`cd`).** Tanpa `x`, kamu tidak bisa masuk ke folder itu atau mengakses file di dalamnya! |
| **Tidak Ada Izin** | **`-`** | Aksi tersebut dilarang. | Aksi tersebut dilarang. |

> 💡 **Dua Fakta Mengejutkan yang Wajib Diketahui SRE:**
> 1. **Untuk Folder, izin `x` (Execute) adalah "Kunci Pintu":** Jika sebuah folder memiliki izin `r--` (bisa dibaca tapi tidak ada `x`), kamu bahkan **tidak bisa `cd` ke dalamnya** dan tidak bisa membaca file di dalamnya! Folder normal hampir selalu membutuhkan pasangan `r` dan `x` (`r-x`).
> 2. **Siapa yang Boleh Menghapus File?** Izin menghapus sebuah file **TIDAK** ditentukan oleh izin `w` pada file itu sendiri, melainkan ditentukan oleh izin **`w` dan `x` pada FOLDER tempat file itu berada!** Sebab bagi Linux, menghapus file berarti "mengubah daftar isi di dalam folder".

---

## 4. Cara Membaca Output `ls -l` Tanpa Pusing

Ketika kamu mengetik `ls -l`, Linux menampilkan baris informasi di sebelah kiri seperti kode rahasia. Mari kita bedah satu per satu!

```bash
ls -l /etc/hosts deploy.sh
```
*Contoh Output:*
```text
-rw-r--r--  1 root   root   1.7K Oct 12 10:00 /etc/hosts
-rwxr-xr-x  1 ubuntu devops  512 Oct 16 14:20 deploy.sh
```

Mari kita pecah **10 karakter pertama** dari `-rwxr-xr--`:

```text
   -       r w x       r - x       r - -
  ───     ───────     ───────     ───────
   │         │           │           │
   │         │           │           └── 4. Izin untuk OTHERS (Orang lain)
   │         │           │                  (r-- = Hanya boleh baca)
   │         │           │
   │         │           └── 3. Izin untuk GROUP (Rekan satu grup 'devops')
   │         │                  (r-x = Boleh baca dan jalankan, TIDAK boleh ubah)
   │         │
   │         └── 2. Izin untuk OWNER / USER (Pemilik 'ubuntu')
   │                (rwx = Boleh baca, ubah, dan jalankan secara penuh)
   │
   └── 1. TIPE FILE (1 karakter pertama):
          • '-' = File biasa (Regular file)
          • 'd' = Direktori (Folder)
          • 'l' = Symbolic Link (Shortcut)
```

Dan arti kolom sisanya:
- `1` : Jumlah *Hard Link* yang menunjuk ke file ini.
- `ubuntu` : Nama **Owner (User)** pemilik file.
- `devops` : Nama **Group** pemilik file.
- `512` : Ukuran file (dalam byte, kecuali ditambah opsi `-h` pada `ls -lh`).
- `Oct 16 14:20` : Waktu terakhir file dimodifikasi.
- `deploy.sh` : Nama file.

> **Tips:** Jika kamu ingin melihat izin dari sebuah **folder itu sendiri** (bukan daftar isi di dalamnya), tambahkan opsi **`-d`** pada `ls`:
> ```bash
> ls -ld /var/log
> # Output: drwxrwxr-x 11 root syslog 4.0K Oct 16 09:00 /var/log
> ```

---

## 5. Mengubah Hak Akses dengan `chmod` (Change Mode)

Untuk mengubah izin file atau folder, kita menggunakan perintah **`chmod`**. Ada 2 cara menuliskannya, dan seorang SRE harus menguasai keduanya karena masing-masing punya kegunaan praktis.

### Cara 1: Mode Angka / Numerik (Octal Mode) — *Cepat untuk Mengatur Semua Izin Sekaligus*

Mengapa orang Linux sering menyebut *"Kasih chmod 755"* atau *"Ubah ke 600"*? Dari mana datangnya angka tersebut?
Sangat mudah! Setiap huruf izin punya **nilai poin tetap**:
- **`r` (Read) = 4 poin**
- **`w` (Write) = 2 poin**
- **`x` (Execute) = 1 poin**
- **`-` (Kosong) = 0 poin**

Kamu cukup **menjumlahkan poinnya** untuk masing-masing kelompok (**Owner**, **Group**, **Others**):

| Total Angka | Penjumlahan Poin | Simbol Huruf | Artinya |
| :---: | :---: | :---: | --- |
| **`7`** | `4 + 2 + 1` | **`rwx`** | **Izin Penuh:** Boleh baca (`4`), tulis (`2`), dan jalankan (`1`). |
| **`6`** | `4 + 2 + 0` | **`rw-`** | **Baca & Tulis:** Boleh baca (`4`) dan tulis (`2`), tidak bisa dijalankan. |
| **`5`** | `4 + 0 + 1` | **`r-x`** | **Baca & Jalankan:** Boleh baca (`4`) dan masuk/jalankan (`1`), tidak boleh diubah. |
| **`4`** | `4 + 0 + 0` | **`r--`** | **Hanya Baca (Read-Only):** Hanya boleh dibaca (`4`). |
| **`3`** | `0 + 2 + 1` | **`-wx`** | Tulis & jalankan (jarang dipakai). |
| **`2`** | `0 + 2 + 0` | **`-w-`** | Hanya tulis (jarang dipakai). |
| **`1`** | `0 + 0 + 1` | **`--x`** | Hanya jalankan/melintas. |
| **`0`** | `0 + 0 + 0` | **`---`** | **Terkunci Rapat:** Tidak boleh diakses sama sekali. |

**Contoh Cara Menghitung `chmod 755` dan `chmod 640`:**
```text
Contoh 1: chmod 755 deploy.sh
  • Digit ke-1 (Owner)  = 7 (4+2+1) -> rwx
  • Digit ke-2 (Group)  = 5 (4+0+1) -> r-x
  • Digit ke-3 (Others) = 5 (4+0+1) -> r-x
  Hasil: -rwxr-xr-x

Contoh 2: chmod 640 config.yaml
  • Digit ke-1 (Owner)  = 6 (4+2+0) -> rw-
  • Digit ke-2 (Group)  = 4 (4+0+0) -> r--
  • Digit ke-3 (Others) = 0 (0+0+0) -> ---
  Hasil: -rw-r-----
```

#### Daftar Kombinasi Angka Standar di Dunia Kerja SRE:
| Angka | Bentuk Izin | Kapan Digunakan? |
| :---: | :---: | --- |
| **`644`** | `-rw-r--r--` | **Standar File Biasa/Umum** (dokumen teks, konfigurasi publik). Pemilik bisa mengedit, orang lain hanya bisa membaca. |
| **`755`** | `-rwxr-xr-x` | **Standar Folder & File Program/Script.** Pemilik punya kuasa penuh, orang lain bisa menjalankan program atau masuk ke folder tanpa bisa merusaknya. |
| **`600`** | `-rw-------` | **Standar File Rahasia Pribadi** (seperti kunci SSH `~/.ssh/id_ed25519` atau file password `.env`). Hanya pemilik yang boleh baca/tulis; grup dan orang lain diblokir total! |
| **`700`** | `drwx------` | **Standar Folder Pribadi/Rahasia** (seperti folder `~/.ssh`). Hanya pemilik yang boleh masuk ke dalam folder ini. |
| **`640`** | `-rw-r-----` | **Standar File Konfigurasi Aplikasi Tim.** Pemilik bisa mengedit, grup aplikasi (misal `www-data`) bisa membaca, dan pihak luar (`others`) diblokir. |
| **`400`** | `-r--------` | **File Sertifikat / Kunci Read-Only.** Bahkan pemiliknya sendiri sengaja dicegah agar tidak sengaja menimpa/mengedit isinya. |

```bash
# Contoh mempraktikkan Mode Angka:
chmod 644 index.html            # File web biasa
chmod 755 backup.sh             # Script bash agar bisa dieksekusi
chmod 700 ~/.ssh                # Mengamankan folder SSH
chmod 600 ~/.ssh/id_ed25519     # Mengamankan private key SSH
```

> 🚨 **LARANGAN KERAS DI PRODUCTION: Jangan Pernah Gunakan `chmod 777`!**
> Saat menemui error *"Permission denied"*, banyak pemula mencari jalan pintas di internet lalu mengetik `chmod 777` (`rwxrwxrwx`).
> Memberikan `777` berarti **mengizinkan siapa saja di server (termasuk proses asing atau peretas)** untuk mengedit, menghapus, atau menyisipkan virus ke dalam file/program tersebut. Bahkan program seperti **SSH akan menolak bekerja** jika melihat file kuncinya diberi izin yang terlalu terbuka! Selalu cari tahu *siapa* user yang membutuhkan akses, lalu berikan izin minimal yang diperlukan.

---

### Cara 2: Mode Simbolik / Huruf (Symbolic Mode) — *Praktis untuk Menambah/Mengurangi Satu Izin*

Bagaimana jika kamu hanya ingin menambahkan izin jalan (`x`) pada sebuah script tanpa mau repot menghitung angka izin yang sudah ada? Gunakan **Mode Simbolik**!

**Rumus Mode Simbolik:**
```text
chmod   [Siapa: u/g/o/a]   [Aksi: + / - / =]   [Izin: r/w/x]   nama_file
```
- **Siapa:** **`u`** (User/Owner), **`g`** (Group), **`o`** (Others), **`a`** (All / Semuanya).
- **Aksi:** **`+`** (Tambahkan izin), **`-`** (Cabut/hapus izin), **`=`** (Atur persis menjadi seperti ini).
- **Izin:** **`r`** (Read), **`w`** (Write), **`x`** (Execute).

```bash
chmod u+x deploy.sh             # Tambahkan (+) izin execute (x) untuk Owner (u) saja
chmod +x deploy.sh              # Tambahkan izin execute (x) untuk SEMUA (sama dengan a+x)
chmod g-w laporan.txt           # Cabut (-) izin tulis (w) dari Group (g)
chmod o=r laporan.txt           # Atur (=) agar Others (o) hanya punya izin baca (r) saja
chmod go-rwx rahasia.txt        # Cabut seluruh izin r, w, x dari Group dan Others sekaligus!
chmod -R g+w /opt/shared/       # Tambahkan izin tulis untuk Group secara Rekursif (-R)
                                # ke seluruh folder dan file di dalam /opt/shared/
```

---

## 6. Mengatur Pemilik dan Grup (`chown` & `chgrp`)

Terkadang izin `640` sudah benar, tetapi aplikasimu tetap gagal membaca file karena file tersebut masih dimiliki oleh user `root`, padahal aplikasi berjalan sebagai user `www-data`. Untuk mengubah pemilik atau grup sebuah file, gunakan **`chown` (Change Owner)**.

> **Catatan:** Mengubah kepemilikan file ke user lain hanya bisa dilakukan oleh administrator menggunakan **`sudo`**.

```bash
# 1. Mengubah Owner (pemilik) saja menjadi user 'alice'
sudo chown alice laporan.txt

# 2. Mengubah Owner sekaligus Group (dipisahkan dengan tanda titik dua ':')
sudo chown alice:developers laporan.txt

# 3. Mengubah Group saja (perhatikan tanda titik dua di depan nama group)
sudo chown :developers laporan.txt
# Atau menggunakan perintah khusus chgrp:
sudo chgrp developers laporan.txt

# 4. Mengubah pemilik seluruh folder beserta semua anak file di dalamnya (-R = Recursive)
sudo chown -R www-data:www-data /var/www/html/
```

### Sekilas: Menambahkan User ke dalam Group
Di lingkungan kerja SRE, kamu sering perlu memasukkan user ke dalam grup tertentu (misalnya memasukkan user `ubuntu` ke dalam grup `docker` agar bisa menjalankan Docker tanpa `sudo`):
```bash
# Buat grup baru bernama 'devops'
sudo groupadd devops

# Masukkan user 'ubuntu' ke dalam grup 'devops' (-a = append/tambahkan, -G = nama grup)
sudo usermod -aG devops ubuntu
```
*(PENTING: Jangan pernah lupa huruf **`-a`** pada `-aG`! Tanpa `-a`, user tersebut akan dikeluarkan dari semua grup lamanya!).*

---

## 7. `umask` — Mengapa File Baru Selalu Berizin `644` dan Folder `755`?

Pernahkah kamu bertanya: *"Saat saya membuat file baru dengan `touch coba.txt`, kok izinnya otomatis menjadi `-rw-r--r--` (`644`), sedangkan saat membuat folder dengan `mkdir folderbaru`, kok otomatis menjadi `drwxr-xr-x` (`755`)? Siapa yang mengaturnya?"*

Jawabannya adalah **`umask` (User File-Creation Mode Mask)**.
`umask` adalah "angka penyaring/pengurang" bawaan saat sebuah file atau folder baru dilahirkan:
1. Untuk **Direktori (Folder) baru**, nilai awal maksimalnya adalah **`777`** (`rwxrwxrwx`).
2. Untuk **File baru**, nilai awal maksimalnya adalah **`666`** (`rw-rw-rw-`). *(Mengapa bukan 777? Demi keamanan, Linux tidak pernah mengizinkan file teks yang baru dibuat langsung memiliki izin `x` / execute secara otomatis!).*

Mari kita cek nilai `umask` standar di Ubuntu:
```bash
umask
```
*Output:* `0022` (kita cukup lihat 3 angka terakhirnya, yaitu **`022`**).

Sekarang kurangkan nilai dasar dengan angka `umask` (`022`):
```text
• Saat membuat FILE baru      : 666 - 022 = 644 (-rw-r--r--)
• Saat membuat DIREKTORI baru : 777 - 022 = 755 (drwxr-xr-x)
```
Jika di server yang sangat rahasia kamu mengubah `umask` menjadi **`027`**, maka setiap file baru yang kamu buat akan otomatis berizin `640` (`666 - 027`), sehingga user asing (`others`) sama sekali tidak bisa mengintip file barumu!

---

## 8. `sudo` dan Prinsip *Least Privilege* di Production

Sebagai seorang SRE, kamu memiliki akses administrator di server. Namun, **bekerja sehari-hari dengan login langsung sebagai user `root` adalah kebiasaan yang sangat berbahaya.**
Jika kamu login sebagai `root`, satu kesalahan ketik perintah bisa langsung menghancurkan server tanpa peringatan.

Sebagai gantinya, kita selalu login sebagai **user biasa** (misal `ubuntu`), dan hanya meminjam kekuatan `root` untuk satu perintah tertentu dengan menambahkan kata **`sudo` (Superuser Do)** di depan perintah:

```bash
apt update                      # GAGAL (Permission denied, karena user biasa tidak boleh mengubah sistem)
sudo apt update                 # BERHASIL (Dijalankan dengan hak akses root)

sudo -l                         # Cek perintah sudo apa saja yang diizinkan untuk akunmu
sudo -u postgres psql           # Jalankan perintah 'psql' dengan meminjam identitas user 'postgres'
```

> 🛡️ **Aturan Disiplin SRE di Production:**
> 1. **Hindari kebiasaan mengetik `sudo su -` atau `sudo bash`** untuk berubah menjadi `root` permanen selama berjam-jam.
> 2. **Mengapa?** Ketika ada 5 engineer di tim kamu dan semuanya langsung mengetik `sudo su -`, maka semua perintah selanjutnya tercatat di log server sebagai `root` — kita kehilangan **Audit Trail** (tidak tahu siapa engineer yang sebenarnya menjalankan perintah mematikan database).
> 3. Sebaliknya, jika setiap perintah diawali `sudo` (misal `sudo systemctl restart nginx`), Linux mencatat dengan rapi di dalam `/var/log/auth.log`: *"Pada jam 14:05, user `budi` menjalankan perintah `systemctl restart nginx` sebagai `root`."*

---

## 9. Special Permission Bits (`SUID`, `SGID`, `Sticky Bit`) & `ACL`

Selain `r`, `w`, dan `x`, Linux memiliki 3 "izin sakti" tambahan untuk menangani kasus khusus:

### 1. SUID (Set User ID — Nilai Angka: `4xxx`, Simbol: `s` di kolom Owner)
- **Fungsi:** Saat file program yang memiliki bit SUID dijalankan oleh user biasa, program tersebut **berjalan menggunakan hak akses pemilik file (`root`)**, bukan hak akses orang yang menjalankannya!
- **Contoh Nyata:** Perintah `passwd` (untuk mengganti password). Password disimpan di file `/etc/shadow` yang hanya boleh diubah oleh `root`. Agar user biasa bisa mengganti passwordnya sendiri, file `/usr/bin/passwd` diberi izin SUID:
  ```bash
  ls -l /usr/bin/passwd
  # Output: -rwsr-xr-x 1 root root 59976 ... /usr/bin/passwd
  #            ^ perhatikan huruf 's' menggantikan 'x' pada kolom owner!
  ```
- **Catatan Keamanan:** Jangan pernah memberikan SUID (`chmod 4755`) pada script atau program sembarangan (apalagi `bash`, `vim`, atau `python`), karena itu sama saja memberikan celah bagi user biasa untuk menjadi `root` secara ilegal!

### 2. SGID (Set Group ID — Nilai Angka: `2xxx`, Simbol: `s` di kolom Group)
- **Fungsi pada Direktori:** Normalnya, jika user `budi` membuat file di dalam folder bersama `/opt/tim-sre`, file tersebut akan dimiliki oleh grup `budi`, sehingga rekan setimnya (`alice`) kesulitan mengeditnya.
  Jika folder `/opt/tim-sre` diberi **SGID**, maka **file atau sub-folder apa pun yang baru dibuat di dalamnya akan otomatis mewarisi Group milik folder tersebut (`tim-sre`)!**
- **Cara pakai:**
  ```bash
  sudo mkdir -p /opt/tim-sre
  sudo chown :devops /opt/tim-sre
  sudo chmod 2775 /opt/tim-sre        # Angka '2' di depan mengaktifkan SGID (atau chmod g+s)
  ls -ld /opt/tim-sre                 # Output: drwxrwsr-x (perhatikan huruf 's' di kolom group)
  ```

### 3. Sticky Bit (Nilai Angka: `1xxx`, Simbol: `t` di ujung paling kanan)
- **Fungsi pada Direktori:** Ingat aturan di Bagian 3 bahwa siapa pun yang punya izin tulis (`w`) di sebuah folder bisa menghapus file apa saja di dalamnya?
  Di folder umum seperti **`/tmp`**, semua user diizinkan menaruh file di sana (`rwxrwxrwt`). Agar user `budi` **tidak bisa menghapus file milik user `alice`** di dalam `/tmp`, Linux memasang **Sticky Bit**! Dengan Sticky Bit, hanya pemilik asli file (dan `root`) yang boleh menghapus filenya sendiri.
- **Bukti di terminal:**
  ```bash
  ls -ld /tmp
  # Output: drwxrwxrwt 14 root root 4096 ... /tmp
  #                  ^ perhatikan huruf 't' di ujung kanan!
  ```

### 4. Bonus: `ACL` (Access Control List) — *Ketika User/Group/Others Tidak Cukup*
Bagaimana jika ada sebuah file milik `budi:keuangan`, orang lain (`others`) tidak boleh baca, tapi kamu ingin memberi izin baca & tulis khusus kepada **satu orang saja** bernama `andi` (dari divisi IT) tanpa memasukkan `andi` ke grup `keuangan`? Gunakan **ACL**:
```bash
sudo apt install -y acl                     # Pastikan paket acl terpasang
setfacl -m u:andi:rw laporan-gaji.csv       # Beri izin rw khusus untuk user 'andi' saja
getfacl laporan-gaji.csv                    # Lihat daftar lengkap izin ACL pada file tersebut
```

---

## 10. Pola Kegagalan (Failure Modes) di Production & Cara Diagnosanya

Berikut adalah 4 masalah nyata sehari-hari yang pasti akan kamu temui sebagai SRE beserta cara menyelesaikannya:

| Gejala / Pesan Error | Penyebab Utama | Cara Diagnosa & Solusi |
|---|---|---|
| **`WARNING: UNPROTECTED PRIVATE KEY FILE! Permissions 0644 for 'id_ed25519' are too open.`** saat ingin SSH | SSH menolak bekerja karena file kunci pribadimu bisa dibaca oleh orang lain (`644`). | Kunci rahasia SSH wajib hanya bisa diakses pemiliknya sendiri:<br>`chmod 700 ~/.ssh`<br>`chmod 600 ~/.ssh/id_ed25519` |
| **`bash: ./deploy.sh: Permission denied`** saat menjalankan script | File script baru dibuat atau baru di-download sehingga belum memiliki izin jalan (**`x` / execute**). | Cek dengan `ls -l deploy.sh`. Jika belum ada huruf `x`, jalankan:<br>`chmod +x deploy.sh` (atau `chmod 755 deploy.sh`). |
| **`bash: /tmp/installer.sh: Permission denied`** padahal sudah diberi `chmod 755`! | Di banyak server production yang sudah di-*hardening*, partisi `/tmp` dipasang (*mount*) dengan aturan keamanan **`noexec`** (melarang program apa pun dijalankan dari `/tmp`). | Cek dengan `mount \| grep /tmp`. Solusinya: pindahkan script tersebut ke home directory (`~/`) atau jalankan dengan memanggil `bash /tmp/installer.sh`. |
| Aplikasi Web / Container **CrashLoop** dengan log: **`open /var/log/myapp/app.log: permission denied`** | Folder `/var/log/myapp` dibuat menggunakan `sudo mkdir` sehingga pemiliknya adalah `root:root`, sedangkan aplikasi berjalan sebagai user non-root (misal `appuser`). | Ubah kepemilikan folder log tersebut agar dimiliki oleh user aplikasi:<br>`sudo chown -R appuser:appuser /var/log/myapp` |

---

## 11. Latihan Praktik Mandiri (10 Menit di Terminal VM OrbStack)

Jalankan skenario berikut di dalam VM `lab01` kamu untuk melatih otot jarimu (*muscle memory*):

```bash
# 1. Cek identitas diri dan nilai umask saat ini
whoami
id
umask

# 2. Buat folder kerja dan masuk ke dalamnya
mkdir -p ~/latihan-izin && cd ~/latihan-izin

# 3. Buat file biasa dan perhatikan izin bawaannya (666 - 022 = 644 -> -rw-r--r--)
echo "RAHASIA_DB=12345" > .env
ls -la .env

# 4. Amankan file .env agar HANYA kamu yang bisa membaca dan menulisnya (600)
chmod 600 .env
ls -la .env                     # Pastikan berubah menjadi: -rw-------

# 5. Buat sebuah script bash sederhana
cat << 'EOF' > halo.sh
#!/usr/bin/env bash
echo "Halo SRE! Script ini berhasil dijalankan oleh: $(whoami)"
EOF

# 6. Coba jalankan langsung (pasti gagal karena belum ada izin 'x')
./halo.sh                       # Perhatikan pesan: Permission denied

# 7. Berikan izin execute (755 atau u+x) lalu jalankan kembali
chmod 755 halo.sh
ls -l halo.sh                   # Pastikan berubah menjadi: -rwxr-xr-x
./halo.sh                       # Sekarang berhasil!

# 8. Eksperimen mengubah pemilik file ke user 'nobody' (menggunakan sudo)
sudo chown nobody:nogroup .env
ls -l .env                      # Lihat pemiliknya berubah menjadi nobody

# 9. Karena pemilik .env sekarang adalah 'nobody' dengan izin 600 (-rw-------),
#    coba baca file tersebut sebagai user biasa:
cat .env                        # Pasti ditolak (Permission denied)!
sudo cat .env                   # Tapi 'root' (sudo) tetap bisa membacanya!

# 10. Buat folder kolaborasi tim dengan SGID (2775)
mkdir -p tim-sre
chmod 2775 tim-sre
ls -ld tim-sre                  # Perhatikan huruf 's' pada drwxrwsr-x

# 11. Bersihkan folder latihan setelah selesai
cd ~
sudo rm -rf ~/latihan-izin
```

---

## 12. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Cek identitas user & grup saat ini | `whoami` dan `id` |
| Lihat izin dan pemilik file di dalam folder | `ls -la` |
| Lihat izin dari folder itu sendiri | `ls -ld /path/folder` |
| Ubah izin dengan angka (Octal) | `chmod 644 file.txt` / `chmod 755 script.sh` / `chmod 600 key.pem` |
| Tambah izin jalan (*execute*) dengan cepat | `chmod +x script.sh` |
| Ubah pemilik dan grup sebuah file | `sudo chown user:group file.txt` |
| Ubah pemilik seluruh isi folder secara rekursif | `sudo chown -R user:group /path/folder/` |
| Masukkan user ke dalam grup tambahan | `sudo usermod -aG nama_grup nama_user` |
| Cek nilai pengurang izin bawaan | `umask` |
| Jalankan perintah sebagai administrator (`root`) | `sudo <perintah>` |
| Jalankan perintah sebagai user lain | `sudo -u <nama_user> <perintah>` |

---

## 13. Cek Pemahaman (Self-Check)

Uji pemahamanmu dengan menjawab pertanyaan berikut sebelum membuka kunci jawaban:

1. **Apa arti dari rincian izin `drwxr-x---` jika dilihat pada sebuah folder? Siapa saja yang bisa masuk (`cd`) ke dalam folder tersebut?**
2. **Apakah ada perbedaan hasil akhir antara menjalankan `chmod 754 file.sh` dengan `chmod u=rwx,g=rx,o=r file.sh`?**
3. **Mengapa file baru yang kita buat di Linux secara bawaan memiliki izin `644` (`-rw-r--r--`), bukan `755` atau `600`?**
4. **Mengapa memberikan izin `chmod 777` untuk mengatasi masalah *Permission denied* sangat dilarang di lingkungan server production?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **Arti `drwxr-x---`:**
   - Huruf `d` menunjukkan bahwa ini adalah sebuah **Direktori (Folder)**.
   - **Owner (`rwx`):** Pemilik folder bisa melihat daftar isi (`r`), membuat/menghapus file di dalamnya (`w`), dan masuk ke dalam folder (`x`).
   - **Group (`r-x`):** Anggota grup bisa melihat daftar isi (`r`) dan masuk ke dalam folder (`x`), tetapi **tidak bisa** membuat atau menghapus file di dalamnya (karena tidak ada `w`).
   - **Others (`---`):** User lain di luar grup tidak bisa melihat isinya maupun masuk ke dalam folder tersebut sama sekali.

2. **`chmod 754` vs `chmod u=rwx,g=rx,o=r`:**
   - **Sama persis!** Keduanya menghasilkan izin `-rwxr-x--`:
     - `u=rwx` sama dengan `4 + 2 + 1 = 7`
     - `g=rx` sama dengan `4 + 0 + 1 = 5`
     - `o=r` sama dengan `4 + 0 + 0 = 4`

3. **Mengapa file baru default-nya `644`?**
   - Karena nilai dasar pembuatan file baru di Linux adalah **`666`** (demi keamanan, file baru tidak pernah diberi izin *execute* `x` secara otomatis), kemudian dikurangi oleh nilai default **`umask` `022`** (`666 - 022 = 644`). Jika kamu ingin file rahasia hanya bisa dibaca olehmu sendiri, kamu harus mengubahnya secara eksplisit dengan `chmod 600` atau memperketat `umask` menjadi `077`.

4. **Mengapa `chmod 777` berbahaya di production?**
   - Karena `777` (`rwxrwxrwx`) memberikan hak **baca, tulis, hapus, dan eksekusi kepada SELURUH user dan proses di dalam server** (`others`). Jika salah satu layanan di server disusupi pihak luar, penyerang bisa memodifikasi kode program, menyisipkan *malware*, atau mencuri data sensitif tanpa hambatan.

</details>
