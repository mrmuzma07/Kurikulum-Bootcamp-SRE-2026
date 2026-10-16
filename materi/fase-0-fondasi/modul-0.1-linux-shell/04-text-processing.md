# 04 — Pengolahan Teks (Text Processing) untuk SRE

> **Ringkasan:** Menguasai senjata utama SRE dalam membedah log, konfigurasi, dan data JSON di terminal menggunakan **Pipes (`|`)**, **Redirection (`>`)**, **`grep`**, **`sed`**, **`awk`**, **`jq`**, serta merangkainya menjadi *pipeline* analisis otomatis.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Memahami konsep aliran data di Linux (**`stdin`**, **`stdout`**, **`stderr`**) serta cara membelokkannya menggunakan **Redirect (`>`, `>>`, `2>`)** dan **Pipe (`|`)**.
- Mencari kata kunci atau pola kesalahan (*error*) di dalam jutaan baris log menggunakan **`grep`** dan *Regular Expression* dasar.
- Mengubah dan membersihkan isi teks atau file konfigurasi secara otomatis menggunakan **`sed`**.
- Memotong dan menganalisis data berbentuk kolom/tabel (seperti log web server atau CSV) menggunakan **`awk`**.
- Membongkar dan memfilter data berformat **JSON** (dari API, Docker, atau Kubernetes) menggunakan **`jq`**.
- Merangkai beberapa perintah menggunakan `sort`, `uniq -c`, dan `head` untuk menemukan sumber masalah utama (seperti *Top 5 IP penyerang* atau *Endpoint paling sering error*).

---

## 1. Mengapa SRE Wajib Jago Mengolah Teks?

Di dunia server Linux, **hampir semua informasi disajikan dalam bentuk teks**:
- Rekaman kejadian aplikasi disimpan sebagai file teks di `/var/log/`.
- Pengaturan server disimpan sebagai file teks di `/etc/`.
- Laporan status dari sistem modern (seperti Docker, Kubernetes, dan OpenTofu) keluar dalam bentuk teks berformat **JSON** atau tabel.

Bayangkan server production kamu tiba-tiba diserang atau mengalami error, dan file log web server (`access.log`) ukurannya sudah mencapai **10 Gigabyte (berisi 50 juta baris teks)**.
Jika kamu mencoba membukanya dengan aplikasi seperti Notepad atau Microsoft Word, komputermu pasti langsung macet (*hang/freeze*) karena kehabisan memori!

Dengan alat *Text Processing* di terminal Linux, kamu bisa menyaring 50 juta baris tersebut dan menemukan *"Siapa 5 alamat IP yang paling banyak memicu error 500 dalam 10 menit terakhir"* hanya dalam waktu **kurang dari 2 detik**!

---

## 2. Pondasi Aliran Data: Streams, Redirect (`>`), dan Pipe (`|`)

Sebelum memakai alat bedah teks, kamu harus memahami bagaimana program di Linux menerima masukan dan mengeluarkan hasil.

### A. Tiga Saluran Aliran Data (Standard Streams)
Setiap perintah yang kamu jalankan di terminal sebenarnya memiliki **3 saluran pipa air** yang diberi nomor `0`, `1`, dan `2`:

```text
                        ┌──────────────────────┐
   0: stdin (Input) ───>│                      │───> 1: stdout (Output Normal)
   (Keyboard / File)    │   Perintah Linux     │        (Tampil di Layar)
                        │  (misal: ls, grep)   │
                        │                      │───> 2: stderr (Pesan Error)
                        └──────────────────────┘        (Tampil di Layar)
```

1. **`stdin` (Standard Input — Saluran `0`):** Teks yang masuk ke dalam program (biasanya dari ketikan keyboard kamu).
2. **`stdout` (Standard Output — Saluran `1`):** Hasil **normal/sukses** dari program (default-nya ditampilkan ke layar monitor).
3. **`stderr` (Standard Error — Saluran `2`):** Pesan **keluhan/error** jika program mengalami kegagalan (default-nya juga ditampilkan ke layar monitor).

---

### B. Redirection (`>`, `>>`, `2>`) — Membelokkan Aliran ke File
Daripada hasil perintah hanya lewat di layar terminal lalu hilang, kita bisa membelokkan alirannya agar masuk ke dalam sebuah file:

| Simbol | Nama Aksi | Contoh Penggunaan | Penjelasan untuk Pemula |
|:---:|---|---|---|
| **`>`** | **Overwrite (Timpa)** | `ls -la > daftar.txt` | Simpan hasil normal (`stdout`) ke dalam `daftar.txt`. **AWAS: Jika file sudah ada, seluruh isi lamanya akan dihapus dan ditimpa!** |
| **`>>`** | **Append (Tambah)** | `echo "Baris baru" >> daftar.txt` | Tambahkan hasil di **baris paling bawah** file `daftar.txt` tanpa menghapus isi lamanya (Aman!). |
| **`2>`** | **Error Redirect** | `find / -name "*.conf" 2> error.log` | Tangkap pesan error (`stderr` / saluran 2) saja dan simpan ke dalam file `error.log`. |
| **`2>/dev/null`** | **Buang Error** | `find / -name "*.conf" 2>/dev/null` | Buang semua pesan error ke dalam "lubang hitam" (`/dev/null`) agar layar bersih dari gangguan *Permission denied*. |
| **`> file 2>&1`** | **Gabung Semua** | `./script.sh > semua.log 2>&1` | Simpan hasil sukses (`stdout`) ke `semua.log`, DAN gabungkan juga pesan error (`2>&1`) ke tempat yang sama. |

---

### C. Pipe (`|`) dan `tee` — Menyambung Beberapa Perintah Seperti Lego
Simbol garis tegak **`|` (Pipe / Pipa)** adalah fitur paling jenius di Linux.
Fungsinya: **Menyambungkan saluran keluar (`stdout`) dari perintah di sebelah kiri langsung menjadi saluran masuk (`stdin`) untuk perintah di sebelah kanan!**

```text
┌──────────────┐      Pipe (|)      ┌──────────────────┐      Pipe (|)      ┌──────────────┐
│ cat app.log  │ ─────────────────> │   grep "ERROR"   │ ─────────────────> │    wc -l     │
└──────────────┘   (Kirim semua     └──────────────────┘   (Kirim baris     └──────────────┘
                    baris log)                              ERROR saja)     (Hitung jumlahnya)
```

**Bagaimana jika kita ingin mengalirkan data lewat pipa, TAPI ingin sekaligus menyimpannya ke file di tengah jalan?**
Gunakan perintah **`tee`** (diambil dari nama sambungan pipa berbentuk huruf **T**):

```bash
# Tampilkan hasil di layar (atau teruskan ke perintah berikutnya) SEKALIGUS simpan ke file:
cat app.log | grep "ERROR" | tee hasil-error.log | wc -l
```

---

## 3. `grep` — Mesin Pencari Kata & Pola di Dalam Teks

**`grep`** adalah perintah yang berfungsi seperti fitur **`Ctrl + F` (Find)** super cepat. Ia membaca teks baris demi baris, lalu **hanya menampilkan baris yang mengandung kata kunci yang kamu cari**.

### Struktur Dasar `grep`
```bash
grep [opsi] "kata_yang_dicari" nama_file
```

### Opsi-Opsi `grep` yang Wajib Dikuasai SRE
Mari kita lihat contoh penggunaannya pada sebuah file log bernama `app.log`:

```bash
# 1. Pencarian dasar (peka huruf besar/kecil: "ERROR" beda dengan "error")
grep "ERROR" app.log

# 2. Abaikan huruf besar/kecil (-i = ignore case) -> cocok dengan "Error", "error", "ERROR"
grep -i "error" app.log

# 3. Tampilkan beserta nomor barisnya (-n = line number) -> memudahkan saat ingin buka file
grep -n "ERROR" app.log

# 4. Kebalikan / Pengecualian (-v = invert match)
#    Contoh: Tampilkan semua baris log KECUALI baris sampah "DEBUG"
grep -v "DEBUG" app.log

# 5. Hitung ada berapa jumlah baris yang cocok (-c = count)
grep -c "ERROR" app.log

# 6. Cari di seluruh file di dalam sebuah folder beserta sub-foldernya (-r = recursive)
grep -r "database_password" /etc/

# 7. Hanya tampilkan NAMA FILE-nya saja yang mengandung kata tersebut (-l = files with matches)
grep -rl "ERROR" /var/log/
```

### Jurus Investigasi SRE: Melihat Baris di Sekitar Error (`-A`, `-B`, `-C`)
Di dunia nyata, pesan error di log (seperti *Java/Python Stack Trace*) sering kali terdiri dari beberapa baris ke bawah. Jika kamu hanya memakai `grep "ERROR"`, kamu cuma dapat 1 baris judulnya saja dan kehilangan rincian penyebabnya!

Gunakan opsi konteks:
- **`-A 3` (*After*):** Tampilkan baris yang cocok **beserta 3 baris setelahnya**.
- **`-B 3` (*Before*):** Tampilkan baris yang cocok **beserta 3 baris sebelumnya**.
- **`-C 3` (*Context*):** Tampilkan baris yang cocok **beserta 3 baris sebelum DAN 3 baris sesudahnya**.

```bash
grep -n -C 3 "CRITICAL" app.log
```

### Pengenalan Singkat Regular Expression (`grep -E`)
Jika kamu ingin mencari beberapa kata sekaligus atau mencari pola khusus, tambahkan opsi **`-E` (Extended Regex)**:

| Simbol Regex | Arti | Contoh Perintah `grep -E` | Penjelasan |
|:---:|---|---|---|
| **`\|`** | **ATAU (OR)** | `grep -E "ERROR\|FATAL\|WARN" app.log` | Cari baris yang mengandung kata `ERROR`, **atau** `FATAL`, **atau** `WARN`. |
| **`^`** | **Awal Baris** | `grep -E "^192\.168" access.log` | Hanya cari baris yang **diawali** oleh teks `192.168`. |
| **`$`** | **Akhir Baris** | `grep -E "failed$" app.log` | Hanya cari baris yang **diakhiri** oleh kata `failed`. |
| **`^$`** | **Baris Kosong** | `grep -v -E "^$" config.txt` | Buang (`-v`) semua baris yang kosong melompong (`^$`). |
| **`[0-9]`** | **Angka 0 s.d. 9** | `grep -E " 5[0-9]{2} " access.log` | Cari kode status HTTP 5xx (angka `5` diikuti dua digit angka apa saja, misal `500`, `502`, `504`). |

---

## 4. `sed` (Stream Editor) — Mencari dan Mengganti Teks Otomatis

Jika `grep` adalah fitur *Find* (`Ctrl+F`), maka **`sed`** adalah fitur **Find and Replace (`Ctrl+H`)** otomatis dari baris perintah tanpa perlu membuka teks editor!

### Aturan Emas `sed` untuk Pemula
Secara bawaan (default), **`sed` TIDAK mengubah file aslinya sama sekali!** Ia hanya menampilkan hasil perubahannya ke layar terminal (`stdout`). Ini sangat bagus karena kamu bisa menguji perintahmu dengan aman.
File asli baru akan benar-benar diubah jika kamu menambahkan opsi **`-i` (*in-place*)**.

### 1. Mengganti Teks (`s` = Substitute)
```text
Rumus:   sed 's/teks_lama/teks_baru/g' nama_file
```
- Huruf **`s`** di awal berarti **Substitute** (Ganti).
- Huruf **`g`** di ujung berarti **Global** (Ganti **semua** kata yang cocok di dalam baris tersebut. Tanpa `g`, hanya kata pertama di tiap baris yang diganti!).

```bash
# Uji coba di layar dulu: ganti semua kata 'localhost' menjadi '10.0.0.50'
sed 's/localhost/10.0.0.50/g' config.yaml

# Jika tampilan di layar sudah benar, terapkan langsung ke file asli (-i)
# TIPS SRE: Gunakan -i.bak agar sed otomatis membuat file cadangan 'config.yaml.bak'!
sed -i.bak 's/localhost/10.0.0.50/g' config.yaml
```

> 💡 **Trik Cerdas SRE saat Mengubah Path Direktori:**
> Bagaimana jika teks yang ingin kamu ganti mengandung garis miring `/` (misal mengubah `/var/log` menjadi `/opt/log`)?
> Kamu **boleh mengganti tanda pembatas `/`** di dalam `sed` dengan simbol lain seperti **`|`** atau **`#`** agar tidak bertabrakan!
> ```bash
> sed 's|/var/log|/opt/log|g' config.yaml
> ```

### 2. Menghapus Baris Tertentu (`d` = Delete)
Saat membaca file konfigurasi di `/etc/`, sering kali ada ratusan baris komentar (diawali `#`) dan baris kosong yang mengganggu pandangan. Kita bisa membersihkannya dengan `sed`:
```bash
sed '/DEBUG/d' app.log                  # Hapus semua baris yang mengandung kata 'DEBUG'
sed '/^#/d' /etc/ssh/sshd_config        # Hapus semua baris komentar yang diawali tanda '#'
sed '/^$/d' /etc/ssh/sshd_config        # Hapus semua baris kosong
```

### 3. Menampilkan Baris Nomor Tertentu Saja (`-n` dan `p`)
```bash
# Tampilkan hanya baris ke-10 sampai baris ke-20 dari sebuah file besar:
sed -n '10,20p' app.log
```

---

## 5. `awk` — Pisau Bedah Data Berbentuk Kolom / Tabel

Jika `grep` bekerja memilih **baris**, maka **`awk`** adalah alat ajaib yang melihat setiap baris teks sebagai **kolom-kolom terpisah (seperti tabel Excel)**!

Secara bawaan, `awk` memisahkan kolom berdasarkan **spasi atau tab**.

### Memahami Nomor Kolom di `awk`
Perhatikan satu baris log di bawah ini:
```text
192.168.1.10   GET   /api/login   200   45ms
────────────   ───   ──────────   ───   ────
  Kolom $1     $2        $3       $4     $5 (atau $NF)
```
Di dalam `awk`:
- **`$1`** = Kolom ke-1 (`192.168.1.10`)
- **`$2`** = Kolom ke-2 (`GET`)
- **`$3`** = Kolom ke-3 (`/api/login`)
- **`$NF`** = Kolom **paling terakhir** di baris itu (`45ms`) — berasal dari *Number of Fields*.
- **`$0`** = **Seluruh baris utuh** dari ujung kiri ke ujung kanan.
- **`NR`** = Nomor urut baris saat ini (*Number of Row*).

### Contoh Penggunaan `awk` Sehari-hari

**1. Mengambil kolom tertentu saja:**
```bash
# Ambil hanya kolom ke-1 (IP Address) dan kolom ke-3 (URL) dari file log:
awk '{print $1, $3}' access.log

# Contoh nyata SRE: Ambil daftar PID (kolom ke-2) dan Nama Perintah (kolom ke-11) dari ps aux:
ps aux | awk '{print $2, $11}'
```

**2. Mengatur pemisah kolom selain spasi (`-F` = Field Separator):**
Bagaimana jika datanya dipisahkan oleh tanda koma `,` (file CSV) atau tanda titik dua `:` (seperti file `/etc/passwd`)? Gunakan opsi **`-F`**:
```bash
# File /etc/passwd memisahkan kolom dengan titik dua ':' (username:x:uid:gid:...)
# Tampilkan hanya daftar username (kolom 1) dan shell-nya (kolom 7):
awk -F: '{print $1, $7}' /etc/passwd

# Membaca file CSV (dipisahkan koma ','): tampilkan kolom 1 dan kolom 3
awk -F, '{print $1, $3}' data_karyawan.csv
```

**3. Menyaring baris berdasarkan syarat angka/kondisi di kolom tertentu:**
Di sinilah keunggulan mutlak `awk` dibanding `grep`! `awk` mengerti hitungan matematika:
```bash
# Tampilkan hanya baris yang kolom ke-4 (Status HTTP)-nya bernilai 500:
awk '$4 == 500 {print $1, $3}' access.log

# Lewati baris pertama (NR > 1, biasanya judul tabel), lalu tampilkan yang kolom ke-3 > 100:
awk -F, 'NR > 1 && $3 > 100 {print $1, $3}' transaksi.csv

# Menjumlahkan seluruh angka yang ada di kolom ke-3 (seperti rumus SUM di Excel!):
awk -F, 'NR > 1 {total += $3} END {print "Total:", total}' transaksi.csv
```

---

## 6. `jq` — Sang Penakluk Data JSON

Di dunia Cloud-Native dan DevOps modern, hampir semua alat (Docker, Kubernetes, OpenTofu, GitLab API, hingga log aplikasi modern) mengeluarkan data dalam format **JSON (JavaScript Object Notation)**.

Membaca JSON mentah yang panjangnya satu baris penuh sangat menyiksa mata. Untuk itu, SRE menggunakan **`jq`** (anggap saja seperti `awk` + `sed` khusus untuk data JSON).

*(Jika di VM kamu belum ada `jq`, instal dengan: `sudo apt install -y jq`).*

### Mari Kita Praktikkan `jq` dari Nol!
Misalkan kita punya file `/tmp/server.json` atau respons API seperti ini:
```json
{
  "cluster": "prod-k3s",
  "nodes": [
    {"name": "node-01", "status": "Ready", "cpu_percent": 45},
    {"name": "node-02", "status": "NotReady", "cpu_percent": 92},
    {"name": "node-03", "status": "Ready", "cpu_percent": 30}
  ]
}
```

**1. Merapikan dan mewarnai JSON berantakan (`jq .`):**
```bash
cat /tmp/server.json | jq .
```

**2. Mengambil satu properti tertentu (`.nama_key`):**
```bash
jq '.cluster' /tmp/server.json
# Output: "prod-k3s"

# Gunakan opsi -r (Raw Output) untuk menghilangkan tanda kutip ""
# (Sangat penting saat hasilnya mau disimpan ke variabel script bash!)
jq -r '.cluster' /tmp/server.json
# Output: prod-k3s
```

**3. Membongkar daftar/list Array (`[]`):**
Di dalam JSON di atas, `nodes` adalah sebuah daftar (*Array* yang ditandai kurung siku `[...]`).
```bash
# Ambil elemen pertama saja (indeks dimulai dari 0):
jq '.nodes[0]' /tmp/server.json

# Ambil hanya nama dari SEMUA node di dalam daftar:
jq -r '.nodes[].name' /tmp/server.json
# Output:
# node-01
# node-02
# node-03
```

**4. Menyaring data JSON dengan kondisi (`select(...)`):**
```bash
# Cari nama node yang statusnya sedang bermasalah ("NotReady"):
jq -r '.nodes[] | select(.status == "NotReady") | .name' /tmp/server.json
# Output: node-02

# Cari node yang penggunaan CPU-nya di atas 80%:
jq -r '.nodes[] | select(.cpu_percent > 80) | .name' /tmp/server.json
# Output: node-02
```

---

## 7. Peralatan Pendukung & "Jurus Ranking Top-N" SRE

Untuk melengkapi `grep`, `sed`, `awk`, dan `jq`, ada 4 perintah kecil yang selalu dipakai bersama-sama di dalam pipa (`|`):

1. **`wc -l` (Word Count Lines):** Menghitung ada berapa jumlah baris teks.
2. **`sort`:** Mengurutkan baris teks sesuai abjad (`sort`) atau sesuai nilai angka (`sort -n`), atau dari terbesar ke terkecil (`sort -rn` — *reverse numeric*).
3. **`uniq`:** Menghapus baris duplikat yang kembar.
   - Opsi **`uniq -c`** (*count*): Menghitung berapa kali setiap baris yang kembar tersebut muncul!
   - ⚠️ **ATURAN WAJIB `uniq`:** Perintah `uniq` **HANYA bisa mengenali baris kembar jika posisinya bersebelahan persis!** Oleh karena itu, kamu **WAJIB menjalankan `sort` terlebih dahulu sebelum `uniq`** (`sort | uniq -c`).
4. **`cut` & `tr`:**
   - `cut -d: -f1 /etc/passwd` : Memotong kolom sederhana berdasarkan karakter pembatas (`-d:` pembatas titik dua, `-f1` ambil kolom 1).
   - `tr 'a-z' 'A-Z'` : Mengubah (*translate*) huruf kecil menjadi huruf besar.

### 🏆 Bedah Pipeline Legendaris SRE: Mencari "Top 5 IP Pengunjung"
Hampir di setiap wawancara kerja SRE atau saat menangani insiden serangan web (*DDoS / Brute Force*), kamu akan menggunakan rangkaian pipa ini. Mari kita lihat bagaimana data berubah di setiap sambungan pipa:

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -n 5
```

```text
1. awk '{print $1}' access.log     2. | sort               3. | uniq -c          4. | sort -rn
   (Ambil kolom IP saja)              (Urutkan agar IP        (Hitung jumlah        (Urutkan dari jumlah
                                       kembar berdekatan)      kemunculannya)        terbanyak ke terkecil)
   ───────────────────────────        ──────────────────      ────────────────      ──────────────────────
   10.0.0.2                           10.0.0.1                2 10.0.0.1            3 10.0.0.2
   10.0.0.1                           10.0.0.1                3 10.0.0.2            2 10.0.0.1
   10.0.0.2                     ───>  10.0.0.2          ───>  1 10.0.0.9      ───>  1 10.0.0.9
   10.0.0.9                           10.0.0.2
   10.0.0.2                           10.0.0.2
   10.0.0.1                           10.0.0.9
```
Lalu di ujung paling kanan, `| head -n 5` akan memotong dan menampilkan **5 peringkat teratas saja**!

---

## 8. Pola Kegagalan (Failure Modes) & Jebakan Umum Pemula

Berhati-hatilah terhadap 3 kesalahan klasik saat mengolah teks di server production:

### Jebakan 1: Membaca dan Menimpa File yang Sama Sekaligus (`> file_yang_sama`)
- **Kesalahan:** Kamu ingin membuang baris `DEBUG` dari `app.log`, lalu mengetik:
  ```bash
  # JANGAN PERNAH LAKUKAN INI!
  grep -v "DEBUG" app.log > app.log
  ```
- **Akibatnya:** Seluruh isi `app.log` **langsung hilang menjadi 0 byte (kosong melompong)!**
- **Mengapa?** Karena sebelum perintah `grep` sempat berjalan membaca file, shell Linux terlebih dahulu mengeksekusi tanda `> app.log` yang langsung mengosongkan isi `app.log`!
- **Solusi Aman:** Simpan dulu ke file sementara dengan nama berbeda, baru ganti namanya:
  ```bash
  grep -v "DEBUG" app.log > app.log.clean && mv app.log.clean app.log
  ```

### Jebakan 2: Memakai `uniq` Tanpa `sort` Terlebih Dahulu
- **Kesalahan:** Mengetik `awk '{print $1}' access.log | uniq -c`.
- **Akibatnya:** IP yang sama akan terhitung berkali-kali secara terpisah karena di dalam log urutan waktu kedatangan pengunjung berselang-seling. Selalu gunakan pasangan **`sort | uniq -c`**.

### Jebakan 3: Menjalankan `sed -i` Tanpa Uji Coba atau Backup
- **Kesalahan:** Langsung menjalankan `sed -i 's/.../.../g' /etc/nginx/nginx.conf` dengan pola regex yang salah ketik, sehingga merusak seluruh konfigurasi web server.
- **Solusi Aman:** Selalu jalankan `sed` tanpa `-i` terlebih dahulu untuk melihat hasilnya di layar, atau wajib gunakan `-i.bak` agar punya salinan penyelamat.

---

## 9. Latihan Praktik Terpadu (10 Menit di Terminal VM OrbStack)

Mari kita simulasikan tugas nyata seorang SRE yang sedang menginvestigasi insiden aplikasi web! Salin dan jalankan perintah berikut di terminal VM kamu:

```bash
# 1. Buat folder latihan dan buat simulasi file access.log web server
mkdir -p ~/latihan-teks && cd ~/latihan-teks

cat << 'EOF' > access.log
192.168.1.10 - - [16/Oct/2026:10:00:01] "GET /api/v1/products HTTP/1.1" 200 1420
10.10.99.55 - - [16/Oct/2026:10:00:02] "POST /api/v1/login HTTP/1.1" 401 120
192.168.1.10 - - [16/Oct/2026:10:00:03] "GET /api/v1/cart HTTP/1.1" 200 530
203.0.113.88 - - [16/Oct/2026:10:00:04] "GET /api/v1/checkout HTTP/1.1" 500 85
203.0.113.88 - - [16/Oct/2026:10:00:05] "GET /api/v1/checkout HTTP/1.1" 500 85
192.168.1.25 - - [16/Oct/2026:10:00:06] "GET /api/v1/products HTTP/1.1" 200 1420
203.0.113.88 - - [16/Oct/2026:10:00:07] "GET /api/v1/payment HTTP/1.1" 502 90
10.10.99.55 - - [16/Oct/2026:10:00:08] "POST /api/v1/login HTTP/1.1" 401 120
EOF

# 2. Gunakan 'grep' untuk mencari semua baris yang mengalami error server (kode 500 atau 502)
grep -E " (500|502) " access.log

# 3. Gunakan 'awk' untuk mencetak hanya Kolom 1 (IP), Kolom 7 (URL), dan Kolom 9 (Status Code)
awk '{print $1, $7, $9}' access.log

# 4. Gunakan Jurus Ranking SRE: Hitung berapa kali tiap IP mengakses server, urutkan dari yang terbanyak!
awk '{print $1}' access.log | sort | uniq -c | sort -rn

# 5. Cari tahu IP mana saja yang mengalami error 5xx (kolom ke-9 >= 500):
awk '$9 >= 500 {print $1}' access.log | sort | uniq -c | sort -rn

# 6. Gunakan 'sed' untuk menyensor IP '203.0.113.88' menjadi '[REDACTED]' dan simpan backup-nya:
sed -i.bak 's/203.0.113.88/[REDACTED]/g' access.log
cat access.log
ls -la                          # Perhatikan adanya file cadangan access.log.bak!

# 7. Latihan JSON dengan 'jq': Buat simulasi output status Pod Kubernetes
cat << 'EOF' > pods.json
[
  {"pod": "web-frontend-1", "namespace": "prod", "status": "Running", "restarts": 0},
  {"pod": "payment-worker-1", "namespace": "prod", "status": "CrashLoopBackOff", "restarts": 14},
  {"pod": "redis-cache-1", "namespace": "prod", "status": "Running", "restarts": 1}
]
EOF

# 8. Gunakan 'jq' untuk menemukan nama pod yang sedang tidak "Running" atau restarts > 5:
jq -r '.[] | select(.status != "Running") | .pod' pods.json

# 9. Bersihkan folder latihan setelah selesai
cd ~
rm -rf ~/latihan-teks
```

---

## 10. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Simpan output ke file (timpa isi lama / tambah di bawah) | `perintah > file.txt` / `perintah >> file.txt` |
| Sembunyikan pesan error ke lubang hitam | `perintah 2>/dev/null` |
| Tampilkan output di layar sekaligus simpan ke file | `perintah \| tee output.log` |
| Cari teks di dalam file (abaikan huruf besar/kecil + nomor baris) | `grep -in "error" app.log` |
| Cari teks beserta 3 baris sebelum & sesudahnya | `grep -C 3 "exception" app.log` |
| Kecualikan baris tertentu dari tampilan | `grep -v "DEBUG" app.log` |
| Ganti teks di dalam file sekaligus buat cadangan `.bak` | `sed -i.bak 's/lama/baru/g' config.yaml` |
| Ambil kolom ke-1 dan kolom terakhir dari output teks | `awk '{print $1, $NF}' file.log` |
| Ambil kolom ke-1 dari file yang dipisahkan tanda `:` | `awk -F: '{print $1}' /etc/passwd` |
| Rapikan tampilan JSON & ambil field tertentu tanpa kutip | `jq -r '.items[].name' data.json` |
| Hitung peringkat kemunculan data terbanyak (*Top-N*) | `... \| sort \| uniq -c \| sort -rn \| head -n 10` |
| Hitung jumlah total baris | `wc -l file.log` |

---

## 11. Cek Pemahaman (Self-Check)

Uji pemahamanmu dengan menjawab pertanyaan berikut sebelum melihat kunci jawaban di bawah:

1. **Apa perbedaan antara `sed 's/lama/baru/' file.txt` dengan `sed 's/lama/baru/g' file.txt`? Apakah kedua perintah tersebut mengubah isi `file.txt` di harddisk?**
2. **Mengapa dalam pipeline perhitungan frekuensi (`sort | uniq -c`), kita wajib menjalankan perintah `sort` terlebih dahulu sebelum `uniq -c`?**
3. **Jika diberikan file `access.log` dengan format standar (Kolom `$1` adalah IP Address dan Kolom `$9` adalah HTTP Status Code), tuliskan satu baris perintah (*one-liner*) untuk menampilkan 5 IP Address teratas yang paling sering menghasilkan kode error `500`!**
4. **Mengapa saat mengambil data dari `jq` untuk dimasukkan ke dalam variabel script bash, kita sangat disarankan memakai opsi `-r` (`jq -r '.name'`)?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **`sed 's/lama/baru/'` vs `sed 's/lama/baru/g'` dan efeknya ke file asli:**
   - Tanpa huruf `g` di akhir, `sed` hanya akan mengganti **kemunculan pertama saja di setiap baris** (jika dalam satu baris ada dua kata `lama`, kata kedua tidak ikut terganti). Dengan tambahan **`g` (*global*)**, seluruh kata `lama` di baris tersebut akan diganti semuanya.
   - Kedua perintah tersebut **TIDAK mengubah isi `file.txt` di harddisk**, melainkan hanya mencetak hasil perubahannya ke layar terminal (`stdout`). Untuk mengubah file aslinya, kita harus menambahkan opsi **`-i`** (misal `sed -i.bak 's/lama/baru/g' file.txt`).

2. **Mengapa wajib `sort` sebelum `uniq -c`?**
   - Karena perintah `uniq` hanya membandingkan suatu baris dengan **baris yang berada tepat di atas/bawahnya**. Jika ada data yang sama tetapi letaknya terpisah jauh di baris lain, `uniq` tidak akan mengenalinya sebagai duplikat. Dengan menjalankan `sort` terlebih dahulu, semua baris yang identik akan dikumpulkan bersebelahan sehingga `uniq -c` dapat menghitung jumlahnya secara akurat.

3. **One-liner mencari Top 5 IP penyebab error `500`:**
   ```bash
   awk '$9 == 500 {print $1}' access.log | sort | uniq -c | sort -rn | head -n 5
   ```
   *(Penjelasan: `awk` menyaring hanya baris yang kolom ke-9-nya bernilai `500` lalu mencetak kolom ke-1 yaitu IP-nya, kemudian diurutkan dengan `sort`, dihitung jumlah kemunculannya dengan `uniq -c`, diurutkan dari angka terbesar ke terkecil dengan `sort -rn`, dan diambil 5 teratas dengan `head -n 5`).*

4. **Fungsi opsi `-r` (*raw output*) pada `jq`:**
   - Secara default, `jq` mencetak teks string lengkap dengan tanda kutip ganda JSON (misalnya `"prod-k3s"`). Opsi `-r` membuang tanda kutip tersebut sehingga keluarannya bersih menjadi teks murni (`prod-k3s`) yang siap digunakan sebagai argumen perintah Linux lainnya.

</details>
