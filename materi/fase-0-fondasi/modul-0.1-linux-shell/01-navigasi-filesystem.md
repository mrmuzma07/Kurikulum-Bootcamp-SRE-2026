# 01 — Navigasi & Filesystem Linux

> **Ringkasan:** Mengenal cara kerja penyimpanan file di Linux (Filesystem Hierarchy Standard), cara berpindah tempat di terminal tanpa tersesat, mengelola file/folder dengan aman, memahami konsep *link* (pintasan file), serta mencari file yang menyumbat disk di server production.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Memahami peta besar direktori Linux (**FHS**) dan tahu ke mana harus mencari konfigurasi, aplikasi, maupun *log error*.
- Membedakan **Absolute Path** dan **Relative Path** serta bernavigasi dengan cepat di dalam terminal.
- Membuat, menyalin, memindahkan, dan menghapus file/direktori tanpa resiko merusak sistem.
- Mencari file berdasarkan nama, ukuran, umur, atau pemilik menggunakan perintah `find`.
- Memahami konsep **Inode** serta perbedaan **Hard Link** dan **Soft Link (Symlink)** di dunia nyata.
- Mendiagnosa masalah kapasitas penyimpanan server menggunakan `df` dan `du`.

---

## 1. Mindset Dasar: Dari Klik Mouse ke Baris Perintah (CLI)

Jika selama ini kamu terbiasa menggunakan **Windows Explorer** atau **macOS Finder**, kamu berinteraksi dengan komputer menggunakan tampilan grafis (**GUI — Graphical User Interface**). Kamu melihat gambar map (folder), lalu melakukan *double-click* untuk masuk ke dalamnya.

Di dunia **Site Reliability Engineering (SRE)**, server production yang berjalan di data center (maupun VM di OrbStack) **tidak memiliki layar monitor dan tidak memiliki tampilan klik-klik (GUI)** demi menghemat memori (RAM), CPU, dan menjaga keamanan. Kita mengendalikan server sepenuhnya lewat teks menggunakan **Terminal / Shell (CLI — Command Line Interface)**.

> **Analogi Sederhana:**
> - **GUI (Finder/Explorer):** Seperti berjalan-jalan di dalam mal sambil melihat etalase toko secara visual.
> - **CLI (Terminal):** Seperti berbicara langsung dengan asisten pintar yang sangat cepat dan patuh. Kamu cukup mengetik perintah: *"Tunjukkan semua file log yang ukurannya lebih dari 100 MB di gudang belakang"*, dan dalam hitungan milidetik hasilnya muncul.

### Anatomi Perintah Linux
Hampir semua perintah yang kamu ketik di terminal memiliki pola struktur yang sama:

```text
perintah   [opsi/flag]   [argumen/target]
   │            │               │
  ls           -la          /var/log
```
1. **Perintah (`ls`):** Kata kerja — apa yang ingin kamu lakukan (*list* / tampilkan daftar isi).
2. **Opsi / Flag (`-la`):** Mengubah cara kerja perintah (biasanya diawali tanda strip `-` atau `--`). Misalnya `-l` berarti tampilkan secara *long/detail*, dan `-a` berarti tampilkan *all* termasuk file tersembunyi.
3. **Argumen (`/var/log`):** Target atau objek yang dikenai perintah (di folder mana perintah ini dijalankan).

---

## 2. Peta Penyimpanan Linux: Filesystem Hierarchy Standard (FHS)

### Bagaimana Linux Menyusun Folder?
Di Windows, setiap partisi atau penyimpanan diberi huruf terpisah seperti `C:\`, `D:\`, atau `E:\`.
Di **Linux** (dan macOS), **tidak ada istilah Drive C atau Drive D**. Seluruh sistem penyimpanan disusun seperti **sebuah pohon terbalik tunggal (Single Inverted Tree)** yang berawal dari satu titik puncak paling atas yang disebut **Root** dan dilambangkan dengan satu garis miring: **`/`**.

```text
                        /  (Root: Puncak dari segalanya)
         ┌──────┬───────┼───────┬───────┬───────┐
         ▼      ▼       ▼       ▼       ▼       ▼
       /etc   /var    /home   /usr    /tmp    /opt
                │       │
                ▼       ▼
             /var/log /home/ubuntu
```

### Analogi Gedung Kota Server (FHS)
Agar tidak perlu menghafal mati, bayangkan server Linux sebagai sebuah **Gedung Perkantoran Modern**. Setiap folder di bawah `/` memiliki tugas khusus sesuai standar internasional bernama **FHS (Filesystem Hierarchy Standard)**:

| Path (Lokasi) | Analogi Gedung | Fungsi Nyata di Server | Kapan SRE ke Sini? |
|---|---|---|---|
| **`/`** | Tanah / Pondasi Gedung | **Root directory.** Titik awal dari seluruh folder dan file di Linux. | Sebagai patokan awal penulisan alamat file. |
| **`/etc`** | Ruang Panel Kontrol & Buku Aturan | Menyimpan seluruh **file konfigurasi** sistem dan aplikasi (seperti konfigurasi Nginx, SSH, daftar jadwal cron). | Saat ingin mengubah pengaturan aplikasi atau jaringan server. |
| **`/var`** | Gudang Data Dinamis | Menyimpan data yang ukurannya **terus berubah/bertambah** seiring berjalannya waktu (database, cache, antrean, log). | Saat harddisk server tiba-tiba penuh! |
| **`/var/log`** | Ruang CCTV & Buku Catatan Kejadian | Di sinilah semua **rekaman log aktivitas dan error** sistem maupun aplikasi disimpan. | **Tempat pertama yang dicek SRE** ketika server atau aplikasi mengalami gangguan (*trouble*). |
| **`/home`** | Kamar Apartemen Pribadi Staf | Tempat tinggal masing-masing pengguna biasa (misal: `/home/budi`, `/home/ubuntu`). | Saat menyimpan file kerja pribadi atau konfigurasi SSH user. |
| **`/root`** | Kamar Khusus Direktur Utama | Folder pribadi milik **Superuser (`root`)**. Perhatikan: `/root` berbeda dengan `/`! | Saat login sebagai administrator penuh (`root`). |
| **`/usr/bin` & `/usr/sbin`** | Gudang Peralatan Kerja | Tempat menyimpan file program/perintah yang kita jalankan sehari-hari (seperti perintah `ls`, `cp`, `curl`, `grep`). | Saat mengecek lokasi biner program bawaan OS. |
| **`/opt`** & **`/usr/local`** | Paviliun Tambahan | Tempat menginstal aplikasi pihak ketiga atau buatan sendiri secara manual (misalnya agen monitoring, Grafana, k3s). | Saat memasang software di luar manajer paket bawaan. |
| **`/tmp`** | Meja Kerja Sementara | Tempat menaruh file sementara (*temporary*). **Isinya otomatis terhapus saat server di-restart** atau setelah beberapa hari. | Saat mengunduh file installer sementara atau menaruh hasil uji coba. |
| **`/proc` & `/sys`** | Layar Indikator Mesin (Virtual) | **Folder ajaib (Virtual Filesystem).** File di sini tidak disimpan di harddisk, melainkan dibuat langsung di memori oleh otak Linux (Kernel) secara *real-time*. | Saat ingin mengintip informasi CPU (`/proc/cpuinfo`), RAM (`/proc/meminfo`), atau proses yang sedang jalan. |
| **`/run`** | Papan Pengumuman Hari Ini | Menyimpan data *runtime* sejak server menyala (seperti nomor ID proses/PID dan socket komunikasi antar program). | Saat memeriksa apakah *socket* sebuah layanan sedang aktif. |
| **`/mnt` & `/media`** | Dermaga Bongkar Muat | Tempat mencantolkan (*mounting*) penyimpanan tambahan seperti disk eksternal atau *network storage* (NFS). | Saat menambah harddisk baru atau menghubungkan *shared storage*. |

> 💡 **Filosofi Kunci Linux: *"Everything is a File" (Segala sesuatu adalah File)***
> Di Linux, bukan cuma dokumen teks atau foto yang dianggap sebagai file. Folder, koneksi jaringan (*socket*), proses yang sedang berjalan, bahkan perangkat keras seperti harddisk (`/dev/sda`) dan layar terminalmu direpresentasikan sebagai file! Artinya, kamu bisa memeriksa hampir semua hal di Linux cukup dengan membaca "file"-nya.

---

## 3. Memahami Alamat: Absolute Path vs Relative Path

Untuk berpindah tempat di terminal, kamu harus tahu cara menyebutkan "alamat" folder (disebut **Path**). Ada 2 cara menulis alamat:

### 1. Absolute Path (Alamat Lengkap / Koordinat GPS)
Selalu **diawali dengan tanda garis miring `/`** (Root). Alamat ini menyebutkan rute lengkap dari puncak `/` sampai ke tujuan. Di mana pun posisi kamu berdiri saat ini, *Absolute Path* akan selalu mengarah ke tempat yang sama.
- Contoh: `/var/log/nginx/error.log`
- Analogi: *"Jalan Jenderal Sudirman No. 10, Jakarta Pusat, Indonesia."*

### 2. Relative Path (Alamat Relatif dari Posisi Saat Ini)
**TIDAK diawali dengan tanda `/`**. Alamat ini dihitung dari folder tempat kamu sedang berdiri sekarang.
- Contoh: Jika kamu sudah berada di dalam `/var/log`, kamu cukup mengetik `nginx/error.log` untuk mengakses file yang sama.
- Analogi: *"Dari ruanganmu sekarang, belok kiri lalu masuk pintu kedua."*

### Simbol Pintasan Penting dalam Navigasi
| Simbol | Arti | Contoh Penggunaan |
|---|---|---|
| **`~`** (Tilde) | Folder rumah (*home directory*) milikmu sendiri (misal `/home/ubuntu`). | `cd ~` (Pulang ke rumah sendiri dari mana saja). |
| **`.`** (Satu titik) | Folder tempat kamu berdiri **saat ini**. | `./script.sh` (Jalankan `script.sh` yang ada di folder ini). |
| **`..`** (Dua titik) | Satu tingkat folder **di atasmu** (*parent directory*). | `cd ..` (Naik/mundur satu folder ke atas). |
| **`-`** (Strip) | Folder tempat kamu berada **sebelumnya**. | `cd -` (Kembali ke folder terakhir sebelum pindah, seperti tombol *Back* di TV). |

---

## 4. Perintah Dasar Navigasi (Berpindah & Melihat Sekitar)

Saat pertama kali masuk ke server (SSH), biasakan melakukan ritual tiga langkah: **"Di mana saya? Ada apa di sekitar saya? Ke mana saya harus pergi?"**

### A. `pwd` (Print Working Directory) — *"Di mana posisi saya sekarang?"*
```bash
pwd
```
*Contoh Output:*
```text
/home/ubuntu
```

### B. `ls` (List) — *"Ada file dan folder apa saja di sini?"*
```bash
ls                  # Tampilkan daftar nama file secara sederhana
ls -l               # Tampilkan secara detail (Long format: izin, pemilik, ukuran, tanggal)
ls -la              # Detail + tampilkan juga file tersembunyi (yang namanya diawali titik '.')
ls -lah /var/log    # Detail + semua file + ukuran yang mudah dibaca manusia (-h: KB, MB, GB)
ls -ltrh            # Urutkan berdasarkan waktu modifikasi (-t) secara terbalik (-r)
                    # Trik favorit SRE: file yang baru saja berubah muncul paling bawah!
```

**Cara Membaca Output `ls -lah`:**
```text
drwxr-xr-x  4 ubuntu ubuntu 4.0K Oct 16 10:00 .
drwxr-xr-x  3 root   root   4.0K Oct 10 08:15 ..
-rw-------  1 ubuntu ubuntu 1.2K Oct 16 09:45 .bash_history
-rw-r--r--  1 ubuntu ubuntu  256 Oct 16 10:00 catatan.txt
```
- Huruf pertama paling kiri menunjukkan jenisnya:
  - **`d`** = **Directory** (Folder)
  - **`-`** = **Regular File** (File biasa)
  - **`l`** = **Symbolic Link** (Pintasan/shortcut ke file lain)
- Perhatikan `.bash_history`: Di Linux, **file atau folder apa pun yang namanya diawali tanda titik (`.`) adalah file tersembunyi (*hidden file*)**, biasanya berisi konfigurasi aplikasi. File ini hanya terlihat jika kamu memakai opsi `-a` (`ls -la`).

### C. `cd` (Change Directory) — *"Pindah ke folder lain"*
```bash
cd /var/log         # Pindah ke /var/log (menggunakan Absolute Path)
cd nginx            # Masuk ke sub-folder nginx di dalam /var/log (Relative Path)
cd ..               # Mundur satu langkah ke atas (kembali ke /var/log)
cd ../..            # Mundur dua langkah ke atas (kembali ke /)
cd ~                # Langsung pulang ke /home/ubuntu (bisa juga cukup ketik: cd)
cd -                # Lompat balik ke folder sebelumnya
```

> ⚡ **Tips Kecepatan di Terminal (Wajib Dicoba!):**
> 1. **Tombol `Tab` (Auto-complete):** Jangan pernah mengetik nama file/folder panjang secara penuh! Ketik 2–3 huruf pertama (misal `cd /var/l`), lalu tekan tombol **`Tab`**. Terminal akan otomatis melengkapinya menjadi `cd /var/log/`! Jika tekan `Tab` dua kali, terminal akan menampilkan semua pilihan yang cocok.
> 2. **`Ctrl + C`:** Membatalkan perintah yang sedang berjalan atau mereset baris ketikan.
> 3. **`Ctrl + L`:** Membersihkan layar terminal yang penuh tulisan (sama dengan mengetik `clear`).
> 4. **`Ctrl + R`:** Mencari perintah yang pernah kamu ketik sebelumnya. Tekan `Ctrl+R`, lalu ketik potongan kata perintahnya.
> 5. **Panah Atas `↑` / Bawah `↓`:** Menelusuri riwayat perintah sebelumnya.

---

## 5. Membuat, Membaca, Menyalin, Memindah, dan Menghapus

### A. Membuat Folder (`mkdir` — Make Directory)
```bash
mkdir laporan                           # Buat satu folder bernama 'laporan'
mkdir -p project/{src,test,docs}        # Buat folder bertingkat sekaligus!
```
- **Mengapa pakai `-p` (*parents*)?** Tanpa `-p`, jika kamu mengetik `mkdir a/b/c` padahal folder `a` dan `b` belum ada, Linux akan menolak dengan pesan error. Dengan `mkdir -p a/b/c`, Linux otomatis membuatkan folder `a`, lalu `b`, lalu `c` sekaligus!
- **Apa arti kurung kurawal `{src,test,docs}`?** Ini disebut *Brace Expansion*. Daripada capek mengetik `mkdir project/src`, `mkdir project/test`, dan `mkdir project/docs` tiga kali, kamu cukup menulisnya sekali dalam kurung kurawal tanpa spasi.

### B. Membuat File Kosong & Mengintip Isi File (`touch`, `cat`, `less`, `head`, `tail`)
```bash
touch app.log                           # Membuat file kosong bernama app.log
                                        # (Jika file sudah ada, touch hanya memperbarui jam modifikasinya)

echo "Server menyala dengan normal" > app.log    # Menulis teks ke dalam file (menimpa isi lama)
echo "Baris kedua log" >> app.log                # Menambah teks di baris paling bawah (append)
```

Bagaimana cara membaca isi file tanpa membuka aplikasi teks editor?
```bash
cat app.log             # Tampilkan seluruh isi file langsung di layar (cocok untuk file pendek)
head -n 5 app.log       # Tampilkan 5 baris PERTAMA saja
tail -n 5 app.log       # Tampilkan 5 baris TERAKHIR saja
tail -f app.log         # Pantau baris terakhir secara LIVE (real-time) saat ada log baru masuk!
                        # (Tekan Ctrl+C untuk keluar dari mode tail -f)
less app.log            # Buka file panjang halaman demi halaman (tekan 'q' untuk keluar)
```

### C. Menyalin File & Folder (`cp` — Copy)
```bash
cp app.log app.log.bak                  # Salin file app.log menjadi file cadangan app.log.bak
cp -r project/ project_backup/          # Salin seluruh folder beserta isinya (-r = recursive)
cp -av project/ project_backup/         # Salin folder secara Archive (-a: pertahankan izin, pemilik,
                                        # dan jam asli file) serta Verbose (-v: tampilkan prosesnya)
```
> **Catatan SRE:** Saat mem-backup folder konfigurasi di server production, selalu gunakan **`cp -a`** (atau `cp -av`) agar hak akses (*permission*) dan stempel waktu (*timestamp*) asli file tidak berubah!

### D. Memindahkan atau Mengganti Nama (`mv` — Move)
Di Linux, **memindahkan file** dan **mengganti nama file (*rename*)** menggunakan perintah yang sama persis, yaitu `mv`:
```bash
mv lama.txt baru.txt                    # Mengganti nama file dari lama.txt menjadi baru.txt
mv baru.txt /tmp/                       # Memindahkan file baru.txt ke dalam folder /tmp/
mv /tmp/baru.txt ./disini.txt           # Memindahkan sekaligus mengganti namanya!
```

### E. Menghapus File & Folder (`rm` — Remove)
```bash
rm file.txt                             # Menghapus satu file biasa
rm -i file.txt                          # Menghapus dengan pertanyaan konfirmasi (y/n)
rm -r folder_kosong_atau_isi/           # Menghapus folder beserta seluruh isinya (-r = recursive)
rm -rf folder_sampah/                   # Menghapus paksa (-f = force) tanpa bertanya apa pun!
```

> ⚠️ **PERINGATAN KERAS (GOLDEN RULE SRE):**
> Di terminal Linux, **TIDAK ADA TRASH ATAU RECYCLE BIN!** Begitu kamu menekan `Enter` pada perintah `rm`, file tersebut langsung lenyap dari sistem file.
> 1. **Jangan pernah** menjalankan `rm -rf /` atau `rm -rf /*` (ini akan menghapus seluruh sistem operasi server).
> 2. **Hati-hati menggunakan variabel di dalam script:** Jika kamu menulis `rm -rf $TARGET_DIR/*` dan ternyata variabel `$TARGET_DIR` lupa diisi (kosong), maka perintah itu berubah menjadi `rm -rf /*`!
> 3. **Kebiasaan Aman:** Sebelum menjalankan `rm -rf`, biasakan cek dulu targetnya menggunakan `ls` (misalnya jalankan `ls -la folder_sampah/` terlebih dahulu, baru ganti `ls -la` menjadi `rm -rf`).

---

## 6. Wildcards & Globbing (Pola Pencocokan Nama Cepat)

Saat mengelola ratusan file, kamu tidak perlu mengetik namanya satu per satu. Gunakan karakter **Wildcard** (pola pencocokan):

| Simbol Wildcard | Arti | Contoh Perintah | Penjelasan |
|---|---|---|---|
| **`*`** (Bintang) | Cocok dengan **karakter apa saja, berapa pun panjangnya** (termasuk kosong). | `ls *.log` | Menampilkan semua file yang berakhiran `.log` (`app.log`, `error.log`, `a.log`). |
| **`?`** (Tanda Tanya) | Cocok dengan **tepat 1 karakter apa saja**. | `ls server-?.conf` | Cocok dengan `server-1.conf` dan `server-A.conf`, tapi **tidak** cocok dengan `server-10.conf`. |
| **`[...]`** (Kurung Siku) | Cocok dengan **1 karakter yang ada di dalam daftar/rentang**. | `ls app-[1-3].log` | Hanya cocok dengan `app-1.log`, `app-2.log`, atau `app-3.log`. |

```bash
# Contoh praktis membersihkan semua file .tmp di dalam folder kerja:
ls *.tmp            # 1. Pastikan dulu file apa saja yang kena jaring wildcard!
rm *.tmp            # 2. Jika sudah yakin benar, baru hapus.
```

---

## 7. `find` — Mesin Pencari Bertenaga Tinggi untuk SRE

Saat server mengalami masalah (misalnya harddisk tiba-tiba penuh atau kita lupa di mana letak file konfigurasi tertentu), `find` adalah senjata utama SRE.

### Struktur Dasar `find`
```text
find   [lokasi-pencarian]   [kriteria-filter]   [aksi-lanjutan]
```

### Contoh Penggunaan Populer di Production

**1. Mencari berdasarkan nama file:**
```bash
# Cari file bernama persis "nginx.conf" di dalam folder /etc
find /etc -name "nginx.conf"

# Cari semua file berakhiran ".log" tanpa peduli huruf besar/kecil (-iname = case-insensitive)
find /var/log -iname "*.log"
```

**2. Mencari di seluruh server tanpa terganggu pesan "Permission denied":**
```bash
find / -type f -name "my-config.yaml" 2>/dev/null
```
- **`-type f`**: Hanya cari **File biasa** (gunakan `-type d` jika ingin mencari **Direktori/Folder**, atau `-type l` untuk mencari **Symlink**).
- **Apa arti `2>/dev/null`?** Saat mencari di seluruh `/` sebagai user biasa, ada banyak folder rahasia milik `root` yang tidak boleh kamu buka. Terminal akan memuntahkan ratusan baris pesan error `"Permission denied"`. Menambahkan `2>/dev/null` artinya: *"Buang semua pesan error (aliran nomor 2) ke dalam lubang hitam (`/dev/null`) supaya layar hanya menampilkan hasil yang berhasil ditemukan saja."*

**3. Mencari berdasarkan ukuran file (`-size`) — *Jurus Mengatasi Disk Penuh*:**
```bash
# Cari semua file yang ukurannya LEBIH BESAR dari 100 Megabyte (+100M) di /var
find /var -type f -size +100M
```
*(Satuan ukuran di `find`: `k` = Kilobyte, `M` = Megabyte, `G` = Gigabyte).*

**4. Mencari berdasarkan waktu perubahan (`-mtime` & `-mmin`):**
```bash
# Cari file di /etc yang baru saja diubah dalam 7 hari terakhir (-7)
find /etc -type f -mtime -7

# Cari file di /etc yang baru saja diubah dalam 30 menit terakhir (-30)
# Sangat berguna saat investigasi: "Siapa yang baru saja mengubah konfigurasi server barusan?!"
find /etc -type f -mmin -30

# Cari file sampah di /tmp yang sudah berumur LEBIH DARI 30 hari (+30)
find /tmp -type f -mtime +30
```

**5. Menggabungkan `find` dengan Aksi Otomatis (`-exec` atau `-delete`):**
```bash
# Cari file lebih dari 50MB di /var/log, lalu tampilkan detail ukurannya dengan ls -lh
find /var/log -type f -size +50M -exec ls -lh {} \;

# Cari semua file .log di /var/log yang di dalamnya mengandung kata "ERROR"
find /var/log -type f -name "*.log" -exec grep -l "ERROR" {} \;

# Hapus semua file di /tmp yang umurnya sudah lebih dari 30 hari
find /tmp -type f -mtime +30 -delete
```
> 🔍 **Bedah Sintaks `-exec ls -lh {} \;`:**
> - `-exec`: Jalankan perintah berikut terhadap setiap file yang ditemukan.
> - `{}`: Simbol *placeholder* (wadah) yang otomatis diganti dengan path file yang ditemukan oleh `find`.
> - `\;`: Penanda bahwa perintah di dalam `-exec` sudah selesai.

---

## 8. Memahami Inode, Hard Link, dan Soft Link (Symlink)

Untuk memahami cara kerja *link* (tautan file) di Linux, kita harus kenal dulu dengan konsep **Inode (Index Node)**.

### Apa itu Inode?
Di dalam sistem penyimpanan Linux, **nama file** dan **isi data file** disimpan secara terpisah!
Setiap file yang dibuat akan diberikan sebuah **Nomor Identitas Unik (seperti NIK KTP)** yang disebut **Inode**.
- **Inode** menyimpan seluruh informasi penting file: siapa pemiliknya, berapa ukurannya, kapan dibuat, dan **di titik mana data fisik file tersebut berada di kepingan harddisk**.
- Sedangkan **Nama File** (misal `laporan.txt`) hanyalah sebuah label nama di dalam buku daftar isi folder yang menunjuk ke nomor Inode tersebut.

```text
[Nama File di Folder]          [KTP File (Inode)]           [Harddisk Fisik]
  "laporan.txt"  ──────────>   Inode #105422   ──────────>  [Blok Data Asli]
```

Dari konsep inilah lahir dua jenis tautan (*link*):

### 1. Soft Link / Symbolic Link (Symlink) — *Seperti Shortcut di Desktop*
**Soft Link** adalah file khusus yang isinya hanya menunjuk ke **nama/alamat file lain**.
- Memiliki nomor Inode sendiri yang berbeda dari file aslinya.
- Bisa menunjuk ke file ataupun folder, bahkan lintas partisi harddisk yang berbeda.
- **Kelemahan:** Jika file aslinya dihapus atau dipindahkan, maka Soft Link akan rusak (**Broken / Dangling Symlink** — biasanya berwarna merah berkedip di terminal) karena alamat yang ditunjuk sudah tidak ada.

### 2. Hard Link — *Dua Pintu Menuju Ruangan Fisik yang Sama*
**Hard Link** membuat **nama file baru yang menunjuk langsung ke Nomor Inode yang sama persis** dengan file aslinya.
- Kedua nama file memiliki kedudukan yang setara (sama-sama file asli!).
- **Kelebihan:** Jika file pertama (`a.txt`) dihapus, datanya **TIDAK HILANG** dan tetap bisa dibuka lewat `hard.txt`! Data di harddisk baru benar-benar dihapus oleh Linux jika seluruh Hard Link yang menunjuk ke Inode tersebut sudah dihapus semuanya.
- **Batasan:** Tidak bisa digunakan untuk folder, dan tidak bisa lintas partisi harddisk yang berbeda.

```text
PERBANDINGAN VISUAL:

Hard Link (Berbagi Inode yang sama):
  a.txt    ──────┐
                 ├──> [ Inode #105422 ] ──> [ Data Fisik di Disk ]
  hard.txt ──────┘

Soft Link / Symlink (Menunjuk ke Nama File):
  soft.txt (Inode #999111) ──> menunjuk ke nama "a.txt" ──> [ Inode #105422 ]
```

### Praktik Membuat & Memeriksa Link
```bash
# 1. Buat file asli
echo "Versi Aplikasi 1.0" > a.txt

# 2. Buat Hard Link (menggunakan perintah ln tanpa opsi)
ln a.txt hard.txt

# 3. Buat Soft Link / Symlink (menggunakan opsi -s = symbolic)
ln -s a.txt soft.txt

# 4. Buktikan dengan melihat nomor Inode (-i)
ls -li a.txt hard.txt soft.txt
```
*Contoh Output:*
```text
105422 -rw-r--r-- 2 ubuntu ubuntu 19 Oct 16 10:30 a.txt
105422 -rw-r--r-- 2 ubuntu ubuntu 19 Oct 16 10:30 hard.txt
999111 lrwxrwxrwx 1 ubuntu ubuntu  5 Oct 16 10:30 soft.txt -> a.txt
```
Perhatikan:
1. Nomor paling kiri (`105422`) pada `a.txt` dan `hard.txt` **sama persis**!
2. Pada `soft.txt`, huruf pertamanya adalah **`l`** (link), nomor Inode-nya berbeda (`999111`), dan ada tanda panah **`-> a.txt`**.

> 🎯 **Contoh Pemakaian Symlink oleh SRE di Production:**
> Saat men-deploy aplikasi di server, SRE sering meletakkan rilis aplikasi di `/opt/myapp/releases/v1.0.0/` dan `/opt/myapp/releases/v1.1.0/`, lalu membuat **Symlink** bernama `/opt/myapp/current -> /opt/myapp/releases/v1.1.0/`.
> Jika rilis `v1.1.0` ternyata bermasalah (*error*), SRE bisa melakukan *rollback* dalam waktu 1 detik cukup dengan mengarahkan ulang symlink `current` kembali ke `v1.0.0`!

---

## 9. Inspeksi File & Diagnosa Kapasitas Disk (`df`, `du`, `stat`, `file`)

Sebagai SRE, salah satu alarm (*alert*) yang paling sering berbunyi di tengah malam adalah **"Disk Space > 85%"**. Berikut adalah alat bedah untuk memeriksanya:

### A. `df` (Disk Free) — *"Berapa sisa ruang di partisi harddisk saya?"*
```bash
df -h               # Cek sisa ruang disk dalam satuan GB/MB (-h = human-readable)
df -i               # Cek sisa jatah INODE di setiap partisi!
```
*Contoh Output `df -h`:*
```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        20G  8.5G   11G  45% /
tmpfs           992M  1.2M  991M   1% /run
```

### B. `du` (Disk Usage) — *"Folder atau file mana yang bikin disk saya penuh?"*
Jika `df -h` menunjukkan penggunaan `/` sudah `95%`, gunakan `du` untuk melacak pelakunya:
```bash
du -sh /var/log                 # Lihat total ukuran satu folder /var/log saja (-s = summary)
du -sh /var/log/*               # Lihat rincian ukuran tiap item di dalam /var/log
du -sh /var/* 2>/dev/null | sort -hr | head -n 10
                                # Jurus sakti SRE: Urutkan 10 folder terbesar di dalam /var!
```

### C. `stat`, `file`, dan `realpath` — *"Mengintip Identitas Asli File"*
```bash
stat app.log                    # Menampilkan KTP lengkap file (Ukuran, Inode, Permission,
                                # waktu Access, Modify, dan Change)

file dokumen_misterius          # Menebak jenis asli file dari isinya (bukan dari nama ekstensinya!)
                                # Contoh: meskipun file bernama 'foto.jpg', jika isinya adalah
                                # script bash, perintah 'file' akan membongkarnya!

realpath ../../etc/hosts        # Mengubah alamat relatif yang membingungkan menjadi
                                # alamat Absolute Path yang jelas (/etc/hosts)
```

---

## 10. Pola Kegagalan (Failure Modes) di Production & Cara Diagnosanya

Berikut adalah 3 skenario masalah nyata di lingkungan kerja SRE yang berkaitan dengan materi ini:

### Kasus 1: Error `"No space left on device"`, padahal `df -h` masih sisa 50% kosong!
- **Gejala:** Aplikasi gagal berjalan dan menulis pesan error *No space left on device*, tetapi saat dicek dengan `df -h`, kapasitas harddisk masih tersisa puluhan Gigabyte.
- **Penyebab:** **Inode Exhaustion (Kehabisan jatah Inode).** Biasanya terjadi karena ada aplikasi atau *cronjob* yang membuat jutaan file berukuran sangat kecil (1 KB) di `/tmp` atau `/var/spool`, sehingga jatah nomor KTP file (Inode) habis 100%.
- **Diagnosa & Solusi:**
  1. Jalankan `df -i` dan lihat kolom `IUse%` yang mencapai `100%`.
  2. Cari folder yang menyimpan ribuan file kecil tersebut, lalu bersihkan dengan `find /tmp -type f -delete`.

### Kasus 2: File Log Raksasa (10 GB) Sudah Dihapus dengan `rm`, tapi `df -h` Tetap Penuh 99%!
- **Gejala:** Kamu menemukan file `/var/log/app/huge.log` berukuran 10 GB, lalu menghapusnya dengan `rm /var/log/app/huge.log`. Namun saat dicek lagi dengan `df -h`, ruang disk tidak berkurang sama sekali!
- **Penyebab:** Sesuai prinsip Inode di Bagian 8, selama **masih ada program/proses yang sedang membuka dan menulis ke file tersebut**, Linux belum benar-benar membebaskan blok datanya di disk meskipun namanya sudah dihapus dari folder.
- **Diagnosa & Solusi:**
  1. Cek file yang sudah dihapus tapi masih dipegang proses: `sudo lsof +L1` (materi ini akan kita perdalam di Modul 0.1 Topik 03).
  2. Solusi aman ke depannya: daripada langsung `rm` file log yang sedang dipakai aplikasi aktif, kosongkan isinya menggunakan `truncate -s 0 /var/log/app/huge.log` atau *restart* layanan aplikasinya.

---

## 11. Latihan Praktik Mandiri (10 Menit di Terminal VM OrbStack)

Buka terminal kamu, masuk ke VM Linux (atau terminal latihan), dan jalankan langkah-langkah berikut satu per satu sambil memperhatikan hasilnya:

```bash
# 1. Pastikan kamu berada di home directory milikmu sendiri
cd ~
pwd

# 2. Buat struktur simulasi proyek aplikasi beserta sub-foldernya sekaligus
mkdir -p ~/latihan-sre/{logs,data,config,releases/v1.0}

# 3. Buat beberapa file contoh
touch ~/latihan-sre/data/{users,orders,products}.csv
echo "PORT=8080" > ~/latihan-sre/config/app.conf
echo "2026-10-16 INFO Server started" > ~/latihan-sre/logs/app.log
echo "2026-10-16 ERROR Database timeout" >> ~/latihan-sre/logs/app.log

# 4. Lihat seluruh struktur yang baru kamu buat secara rekursif (-R)
ls -laR ~/latihan-sre

# 5. Gunakan 'find' untuk mencari semua file bereksensi .csv dan lihat detail ukurannya
find ~/latihan-sre/data -type f -name "*.csv" -exec ls -lh {} \;

# 6. Gunakan 'find' untuk mencari file .log mana yang mengandung kata "ERROR"
find ~/latihan-sre -type f -name "*.log" -exec grep -H "ERROR" {} \;

# 7. Buat Symlink bernama 'latest.log' yang menunjuk ke 'app.log'
cd ~/latihan-sre/logs
ln -s app.log latest.log
ls -li
cat latest.log

# 8. Cek berapa total ukuran folder latihan-sre milikmu
du -sh ~/latihan-sre
du -sh ~/latihan-sre/*

# 9. Bersihkan kembali folder latihan setelah selesai
cd ~
rm -rf ~/latihan-sre
```

---

## 12. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Cek posisi folder saat ini | `pwd` |
| Pindah direktori | `cd /path/tujuan` (atau `cd ~`, `cd ..`, `cd -`) |
| Lihat daftar isi folder + file tersembunyi | `ls -lah` |
| Lihat daftar file diurutkan dari yang terbaru | `ls -ltrh` |
| Buat satu atau banyak sub-folder bertingkat | `mkdir -p folder/sub1/sub2` |
| Buat file kosong | `touch namafile.txt` |
| Salin file atau folder beserta atribut aslinya | `cp -av sumber/ tujuan/` |
| Pindah lokasi atau ganti nama file/folder | `mv lama baru` |
| Hapus file / folder | `rm file.txt` / `rm -rf folder/` |
| Cari file berdasarkan nama / ukuran / umur | `find /path -name "*.log" -size +50M -mtime -7` |
| Buat pintasan file (Symbolic Link) | `ln -s target_asli nama_shortcut` |
| Cek sisa kapasitas partisi & Inode harddisk | `df -h` dan `df -i` |
| Cek ukuran spesifik tiap folder | `du -sh /path/*` |
| Lihat alamat lengkap (Absolute Path) sebuah file | `realpath namafile` |

---

## 13. Cek Pemahaman (Self-Check)

Coba jawab pertanyaan berikut dengan kata-katamu sendiri sebelum melihat kunci jawaban di bawah:

1. **Apa perbedaan fungsi antara direktori `/etc` dengan `/var` di server Linux?**
2. **Mengapa saat menjalankan `find / -name "app.conf" 2>/dev/null` sebagai user biasa, kita sering menambahkan `2>/dev/null` di ujung perintah?**
3. **Apa yang terjadi pada *Hard Link* dan *Soft Link (Symlink)* jika file aslinya dihapus? Kapan kita lebih memilih memakai *Symlink*?**
4. **Apa bedanya fungsi perintah `df -h` dengan `du -sh` saat menginvestigasi masalah harddisk penuh?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **`/etc` vs `/var`:**
   - **`/etc`** digunakan untuk menyimpan **file konfigurasi statis** sistem dan aplikasi (isinya jarang berubah kecuali kita mengubah pengaturan).
   - **`/var`** (*variable*) digunakan untuk menyimpan **data dinamis yang ukurannya terus bertambah/berubah** selama server berjalan, seperti file log (`/var/log`), cache, dan database.

2. **Fungsi `2>/dev/null`:**
   - Saat mencari dari puncak `/`, terdapat banyak folder sistem yang tidak boleh diakses oleh user biasa sehingga memunculkan ratusan pesan error `"Permission denied"`. Menambahkan `2>/dev/null` akan mengalihkan dan membuang pesan error tersebut ke `/dev/null` agar layar terminal bersih dan hanya menampilkan file yang berhasil ditemukan.

3. **Hard Link vs Soft Link (Symlink):**
   - Jika file asli dihapus, **Hard Link tetap utuh dan datanya masih bisa dibaca** karena Hard Link menunjuk langsung ke nomor Inode dan blok data fisik yang sama. Sebaliknya, **Soft Link akan rusak (*broken/dangling link*)** karena ia hanya menunjuk ke nama/jalur file asli yang kini sudah hilang.
   - Kita memilih **Symlink** ketika ingin membuat pintasan untuk **folder**, menunjuk file yang berada di **partisi disk berbeda**, atau agar mudah terlihat secara visual (`ls -l` menampilkan panah `->`) saat menunjuk versi rilis aplikasi aktif (misal `current -> releases/v1.2`).

4. **`df -h` vs `du -sh`:**
   - **`df -h` (Disk Free)** melihat kapasitas dari sudut pandang **partisi/filesystem secara keseluruhan** (berapa persen yang terpakai dan berapa GB yang masih kosong).
   - **`du -sh` (Disk Usage)** menghitung total ukuran dari sudut pandang **folder dan file satu per satu** untuk menemukan folder atau file mana yang menjadi penyebab utama penuhnya harddisk.

</details>
