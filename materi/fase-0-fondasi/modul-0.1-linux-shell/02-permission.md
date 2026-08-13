# 02 — Permission, User, Group

> Siapa boleh membaca, menulis, dan menjalankan file/direktori.

## Tujuan
- Baca `ls -l` tanpa menerjemahkan satu-satu
- Pakai `chmod` (numeric + symbolic) dengan benar
- Atur pemilik dengan `chown`, default mask dengan `umask`
- Paham `sudo` dan kapan izin jadi masalah

## 1. Tiga Lapis Permission

Setiap file punya:
- **Owner** (user) — biasanya pembuat
- **Group** — kumpulan user
- **Other** — semua yang lain

Tiga aksi:
- **r** (read) — buka, tulis (write direktori = buat/hapus di dalamnya)
- **w** (write) — ubah isi
- **x** (execute) — jalanin (untuk direktori = masuk/akses isi)

Direktori wajib:
- `r` boleh `ls` isinya
- `w` boleh tambah/hapus/rename entry di dalamnya
- `x` boleh `cd` & akses file di dalamnya
- sering `x` **lebih penting** dari `r` untuk direktori

Bukti:
```bash
ls -l /etc/hosts
# -rw-r--r-- 1 root wheel 1.7K Jan 12 10:00 /etc/hosts
#   ^^^^^^^^^ ^^^^ ^^^^   ^^^  ^^^^   ^^^   ^^^^^^^^^^^^^^^^
#   permission owner grp  size  date  time  name
```

## 2. Numeric Mode (Octal)

Setiap bit = 1. Tiga pasang digit = (owner)(group)(other).

| Angka | r | w | x | Arti |
|---|---|---|---|---|
| 7 | 1 | 1 | 1 | rwx |
| 6 | 1 | 1 | 0 | rw- |
| 5 | 1 | 0 | 1 | r-x |
| 4 | 1 | 0 | 0 | r-- |
| 3 | 0 | 1 | 1 | -wx |
| 2 | 0 | 1 | 0 | -w- |
| 1 | 0 | 0 | 1 | --x |
| 0 | 0 | 0 | 0 | --- |

Yang sering muncul:
- `644` — file biasa (`rw-r--r--`), umum default
- `755` — executable + direktori (`rwxr-xr-x`)
- `700` — privat (home/keys)
- `600` — key/pribadi (`rw-------`)
- `400` — read-only eksplisit

```bash
chmod 755 deploy.sh
chmod 600 ~/.ssh/id_ed25519
chmod 644 public.txt
```

## 3. Symbolic Mode

Lebih mudah dibaca saat "tambahkan saja permission tertentu".

```bash
chmod u+x deploy.sh          # owner: +execute
chmod g-w file.txt           # group: -write
chmod o=r file.txt           # other: set ke read-only
chmod a+x script.sh          # all
chmod -R go-rwx secrets/     # rekursif, hapus semua permission group/other
```

**simbol:** `u` user, `g` group, `o` other, `a` all. Operator: `+ - =`.

## 4. Owner & Group

```bash
id                              # siapa saya, group apa
chown alice file.txt            # ubah owner
chown alice:dev file.txt        # owner:group
chown :dev file.txt             # hanya group
chgrp dev file.txt              # alternatif
chown -R alice:dev /opt/app/    # rekursif
```

## 5. `umask` — Default Permission

`umask` = default mask yang dikurangi dari `666` (file) atau `777` (direktori).

```bash
umask                # default: 0022
# file baru: 666 - 022 = 644
# direktori baru: 777 - 022 = 755
```

Atur per-user di `~/.bashrc`:
```bash
umask 027
# file: 640, dir: 750
```

## 6. `sudo` dan Prinsip Privilese

```bash
sudo whoami                 # root
sudo -u redis whoami        # jadi user redis
sudo -i                     # shell root interaktif
sudo -l                     # apa yang diizinkan untuk user saya
```

Penting di production:
- jangan biasakan `sudo su -` atau `sudo bash` — jejak hilang
- pakai `sudo systemctl restart nginx` (jejak tercatat)
- pakai akun service khusus untuk aplikasi (bukan root)

## 7. Special Permission Bits (Sekilas)

| Bit | Numeric | Arti |
|---|---|---|
| setuid | `4xxx` | file dijalankan sebagai owner (mis. `passwd`) |
| setgid | `2xxx` | direktori: file baru turun dengan group direktori |
| sticky | `1xxx` | di direktori shared, hanya owner boleh hapus miliknya (mis. `/tmp`) |

```bash
ls -ld /tmp                    # pasti drwxrwxrwt (sticky)
chmod 4755 myprog              # setuid
```

## 8. Akses Praktis: `acl` (Bonus)

Saat satu user butuh diakses tanpa jadi group:

```bash
sudo apt install -y acl
setfacl -m u:alice:rw file.txt
getfacl file.txt
```

## 9. Kasus Umum & Fix

| Masalah | Penyebab | Solusi |
|---|---|---|
| `Permission denied` saat akses file | permission/owner | `chmod`/`chown` |
| `mkdir: cannot create` | parent dir tidak writable | `chmod` parent |
| `ssh` tolak key | mode `~/.ssh` bukan 700 / key 600 | `chmod 700 ~/.ssh; chmod 600 ~/.ssh/id_*` |
| Pemula biasa kerja di `/tmp` | `/tmp` di-mount dengan `noexec` | selesai, bukan masalah |

## Latihan Cepat

```bash
# 1. Bikin file, lihat default permission
touch hello
ls -l hello                     # -rw-r--r-- (0666 - 0022)

# 2. Buat direktori, default 755
mkdir test && ls -ld test

# 3. Set menjadi executable
chmod 755 test

# 4. Ubah owner (cuma root yang bisa)
sudo chown nobody hello
ls -l hello

# 5. Selidiki umask
umask
echo "test" > new.txt
ls -l new.txt                   # hitung sama umask lalu

# 6. Bersihkan
rm -rf test hello new.txt
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Lihat permission | `ls -l` |
| Ubah mode | `chmod` |
| Ubah owner/group | `chown` / `chgrp` |
| Default mode | `umask` |
| Jalan sebagai admin | `sudo` |

## Cek Pemahaman

1. Apa beda `chmod 754` vs `chmod u=rwx,g=rx,o=r`?
2. Kenapa file `~` saya baru default 644, bukan 600?
3. Kapan setuid bermanfaat? Kapan berbahaya?
