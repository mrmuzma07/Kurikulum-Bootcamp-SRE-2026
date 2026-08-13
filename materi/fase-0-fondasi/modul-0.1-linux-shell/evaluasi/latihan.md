# Latihan — Modul 0.1 Linux & Shell

Lakukan latihan **di VM `lab01`** (bukan di Mac). Target: muscle memory, bukan hafalan.

> **Aturan:** setiap soal wajib pakai **pipeline atau skrip**, bukan klik GUI.

---

## Hari 1 — Filesystem & Permission

### 1.1 Filesystem
```bash
# 1. Buat struktur:
#   latihan/
#   ├── src/
#   │   ├── main.go
#   │   └── lib.go
#   ├── test/
#   │   └── main_test.go
#   └── README.md
mkdir -p latihan/{src,test} && touch latihan/src/{main,lib}.go latihan/test/main_test.go latihan/README.md

# 2. List rekursif, simpan ke file, lalu lihat ukurannya
find latihan -type f | sort > latihan/index.txt
wc -l latihan/index.txt

# 3. Cari file .go dan hitung jumlah baris per file
find latihan -name "*.go" -exec wc -l {} \;

# 4. Buat symlink latest -> src/main.go
ln -s src/main.go latihan/latest
ls -la latihan/

# 5. Hapus seluruh direktori latihan
rm -rf latihan
```

### 1.2 Permission
```bash
# 1. Default umask
umask

# 2. Buat file, cek permission default vs calculation
echo "x" > a.txt
ls -l a.txt     # 644?

# 3. Set permission 600 dan verifikasi
chmod 600 a.txt && ls -l a.txt

# 4. Buat script, tambahkan execute bit
echo '#!/usr/bin/env bash
echo hi' > run.sh
chmod 755 run.sh
./run.sh

# 5. Set owner ke nobody (butuh sudo)
sudo chown nobody:nogroup a.txt
ls -l a.txt

# 6. Buat direktori bersama, set setgid
mkdir -p /tmp/shared
chmod 2777 /tmp/shared   # setgid bit
ls -ld /tmp/shared

# 7. Cleanup
sudo rm -f /tmp/shared/a.txt
rm -rf /tmp/shared a.txt run.sh
```

---

## Hari 2 — Proses & Text Processing

### 2.1 Proses
```bash
# 1. Lihat semua proses dengan tree
pstree -p | head -30

# 2. Cari proses sshd
ps -ef | grep -v grep | grep sshd

# 3. Stress: jalankan 5 sleep 100 & di background, lalu kill semua
sleep 100 &
sleep 100 &
sleep 100 &
sleep 100 &
sleep 100 &

jobs
pkill -f "sleep 100"

# 4. Lihat 5 proses paling banyak makan CPU
ps -eo pid,user,pcpu,cmd --sort=-pcpu | head -6

# 5. Cek service yang gagal
systemctl --failed
```

### 2.2 Text Processing
```bash
# 1. Ambil kolom 1, 3, 7 dari /etc/passwd (user, uid, shell)
awk -F: '{print $1,$3,$7}' /etc/passwd | head

# 2. Hitung jumlah shell berbeda yang dipakai user
awk -F: '{print $7}' /etc/passwd | sort | uniq -c | sort -rn

# 3. Cari baris "ERROR" dari file log dummy
cat > /tmp/dummy.log <<'EOF'
2024-01-15 10:00:01 INFO start
2024-01-15 10:00:02 WARN low memory
2024-01-15 10:00:03 ERROR out of disk
2024-01-15 10:00:04 INFO retry
2024-01-15 10:00:05 ERROR timeout
EOF
grep -E "ERROR|WARN" /tmp/dummy.log
grep -c "ERROR" /tmp/dummy.log

# 4. Hapus info line, tampilkan error unik
grep "ERROR" /tmp/dummy.log | sed -E 's/^\S+ \S+ //' | sort -u

# 5. Replace timestamp di file
sed -E 's/2024-01-15/2024-01-16/' /tmp/dummy.log

# 6. JSON: ambil semua ipv4 address
curl -s https://api.ipify.org?format=json | jq -r .ip
```

---

## Hari 3 — Networking & SSH

### 3.1 Networking
```bash
# 1. Cek DNS
dig example.com +short
dig @1.1.1.1 example.com +short

# 2. Cek TLS expiration
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# 3. Lihat port listening
ss -ltn

# 4. Traceroute ke 8.8.8.8 (mungkin diblok)
traceroute -n 8.8.8.8 | head

# 5. Cek ARP
ip neigh
```

### 3.2 SSH
```bash
# 1. Tambahkan host alias ke ~/.ssh/config (jika belum)
# 2. ssh lab1, lalu cek uptime
ssh lab01 "uptime"

# 3. Tunnel: buka 8080 di Mac yang terusun ke 80 di VM
#    Di terminal yang akan ditinggal (background):
#    ssh -fN -L 8080:localhost:80 lab01
#    Lalu: curl -I http://localhost:8080

# 4. Transfer file
echo "remote" > /tmp/sample.txt
scp /tmp/sample.txt lab01:/tmp/uploaded.txt
ssh lab01 "md5sum /tmp/uploaded.txt"
```

---

## Hari 4 — Bash Scripting

### 4.1 Tulis Skrip Sederhana
1. Skrip `disk-check.sh` di `lab/scripts/`:
   - Terima argumen: threshold (default 80)
   - Cek `df -h`, daftar filesystem dengan `Use% >= threshold`
   - Exit 0 jika aman, exit 1 jika ada yang kritis
   - Tulis output ke `stdout`, log info ke `stderr`
2. Set `set -euo pipefail` dan jelaskan tiap baris
3. Tulis `--help` di usage

### 4.2 Tulis Skrip Backup (mini version)
1. `lab/scripts/backup-mini.sh`:
   - Ambil sumber dari arg-1, tujuan dari arg-2
   - Tar+gzip dengan timestamp
   - Retensi 3 hari (DELETE)
   - Lock file agar tidak jalan paralel
   - Cleanup via `trap`

### 4.3 Tulis Skrip Query
1. `lab/scripts/top-errors.sh`:
   - Ambil argumen path file log
   - Print 10 baris ERROR paling sering (case-insensitive)
   - Format: `jumlah PATH`

---

## Hari 5 — Review & Troubleshooting Mini

### 5.1 Cepat (5 menit)
```bash
# 1. Semua proses python
ps -ef | grep -i python | grep -v grep

# 2. Total user login
who | wc -l

# 3. Disk usage terbesar
du -sh /var/* 2>/dev/null | sort -h | tail

# 4. Service systemd gagal
systemctl --failed
```

### 5.2 Skenario (15 menit)
**Bayangkan ini terjadi pagi ini:**

> "Tim dev bilang app tidak bisa diakses. CPU normal, memory normal, tapi port 8080 seolah closed."

Lakukan langkah berikut dan catat output:
```bash
# 1. Apakah service jalan?
systemctl status myapp.service   # jika ada

# 2. Apakah port listening?
ss -ltn | grep 8080

# 3. Apakah ada firewall?
sudo ufw status
sudo iptables -L -n -v          # alternatif

# 4. Coba akses local
curl -v http://localhost:8080

# 5. Cek log
sudo journalctl -u myapp -n 50 --no-pager

# 6. Cek DNS
dig app.example.com
```

Tulis satu paragraf diagnosis: apa hipotesismu, apa langkah berikutnya?

---

## Catatan Performa

- [ ] Semua latihan dilakukan tanpa GUI
- [ ] Jawaban disimpan di repo `sre-bootcamp` di `lab/log-latihan.md`
- [ ] Bisa menjelaskan satu per satu command yang dipakai
