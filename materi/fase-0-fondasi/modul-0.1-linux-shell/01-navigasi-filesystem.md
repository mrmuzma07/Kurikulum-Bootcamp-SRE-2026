# 01 — Navigasi & Filesystem

> FHS (Filesystem Hierarchy Standard), perintah dasar, find, dan link.

## Tujuan
- Paham layout filesystem Linux dan di mana ______ biasanya hidup
- Berpindah, membuat, menyalin, menghapus dengan mulus
- Menemukan file besar / hidangan basi dengan `find`

## 1. Filesystem Hierarchy Standard (FHS)

Layout di Linux server sesuai FHS. Tidak dihafal, dipahami dengan sering lihat.

| Path | Fungsi |
|---|---|
| `/` | root, semua bermuara di sini |
| `/etc` | file konfigurasi sistem & aplikasi |
| `/var` | data variabel: log (`/var/log`), state, spool |
| `/var/log` | log utama — pertama cek saat trouble |
| `/home` | home user biasa |
| `/root` | home user root |
| `/opt` | aplikasi tambahan/vendor (mis. k3s, Grafana) |
| `/usr/local` | program yang di-install manual |
| `/usr/bin`, `/usr/sbin` | biner dari paket distro |
| `/tmp` | file sementara, sering dibersihkan saat reboot |
| `/proc` | info proses & kernel — virtual filesystem |
| `/sys` | info device & kernel — virtual filesystem |
| `/run` | data runtime, runtime socket, PID file |
| `/srv` | data untuk service (web, ftp) |
| `/mnt`, `/media` | mount point sementara & removable |

**fakta penting:** di Linux, **semua adalah file** — direktori, socket, device, proses punya representasi file.

## 2. Perintah Gerak

```bash
pwd                                  # di mana saya
ls -la                               # lihat detail (l=long, a=semua)
ls -lah /var/log                     # -h human-readable
ls -ltrh                             # sort by time, reverse (terbaru di bawah)
cd ~                                 # home
cd -                                 # balik ke direktori sebelumnya
cd /etc/systemd                      # absolut vs relatif
```

**tips:** `Ctrl+L` = clear, `Ctrl+R` = cari history, `!!` = jalanin command terakhir.

## 3. Membuat, Mengisi, Memindah

```bash
mkdir -p project/{src,test,docs}     # -p bikin parent + multiple dirs
touch file.txt                       # bikin file kosong / update timestamp
cp -av src/ dst/                     # -a archive, -v verbose
mv old.txt new.txt                   # rename atau pindah
rm file.txt                          # biasa
rm -rf dir/                          # -r rekursif, -f paksa — HATI-HATI
rm -i file.txt                       # konfirmasi satu-satu
```

**Golden Rule:** `rm -rf` jangan pernah ke root `/`, jangan ke path yang disimpan di variabel kosong. Selalu cek 2x.

## 4. Wildcards & Glob

```bash
ls *.log                  # semua file .log
ls file?.txt              # satu karakter
ls file[0-9].txt          # satu digit
ls **/*.conf              # perlu shopt -s globstar
```

## 5. `find` — Pencari Bertenaga

`find` = perintah paling sering dipakai SRE. Biasakan.

```bash
find /var/log -name "*.log"                        # by name
find / -type f -name "nginx.conf" 2>/dev/null      # -type f/d/l; 2>/dev/null sembunyikan permission denied
find /etc -mtime -7                                # dimodifikasi < 7 hari
find /etc -size +1M                                # lebih besar dari 1 MB
find /var/log -size +100M -exec ls -lh {} \;       # cari log gede, lihat ukurannya
find / -user alice 2>/dev/null                     # by owner
find /var -newer /tmp/marker                       # lebih baru dari file marker
find /tmp -type f -mtime +30 -delete               # hapus file > 30 hari di /tmp
```

**kombinasikan:**
```bash
find /var/log -name "*.log" -exec grep -l "ERROR" {} \;
```
Cari semua `.log` yang mengandung kata `ERROR`.

## 6. Hard Link vs Soft Link

```bash
echo "hello" > a.txt
ln a.txt hard.txt             # hard link, kedua file berbagi inode
ln -s a.txt soft.txt          # symlink, cuma pointer

ls -li a.txt hard.txt soft.txt
# ^^^ lihat inode: a.txt dan hard.txt sama, soft.txt beda
```

- **Hard link:** sama-sama file asli, salah satu dihapus yang lain masih utuh. Tidak bisa lintas filesystem.
- **Soft link (symlink):** pointer. Hapus target → symlink jadi "dead" (ada, tapi merah).

## 7. Operasi Lanjut

```bash
stat file.txt                  # metadata lengkap
file file.txt                  # tebak tipe file
du -sh /var/log/*              # ukuran direktori
df -h                          # disk free
mount | grep sda               # apa yang di-mount
realpath ../../etc/hosts        # path absolut
```

## Latihan Cepat (5 menit)

```bash
# 1. Buat struktur berikut:
mkdir -p latihan/{logs,data,backup}
touch latihan/data/{a,b,c}.csv
touch latihan/logs/app.log

# 2. List rekursif dan simpan ke file
ls -R latihan > latihan/list.txt

# 3. Cari file .csv dan lihat ukurannya
find latihan/data -type f -name "*.csv" -exec ls -lh {} \;

# 4. Buat symlink logs/latest -> logs/app.log
ln -s app.log latihan/logs/latest
ls -la latihan/logs             # lihat panah ->

# 5. Hapus folder latihan (opsional, lihat isinya dulu)
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Pindah direktori | `cd` |
| Lihat isi | `ls -la` |
| Cari file | `find` |
| Cari teks di file | `grep` (topik 04) |
| Lihat ukuran direktori | `du -sh` |
| Cek sisa disk | `df -h` |
| Path absolut | `realpath` |

## Cek Pemahaman

1. Apa beda `/etc` vs `/var`?
2. Kenapa `find / -name "xxx" 2>/dev/null` ditambah `2>/dev/null`?
3. Kapan hard link berguna vs symlink?
