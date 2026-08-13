# Kuis & Kunci Jawaban — Modul 0.1 Linux & Shell

> **Cara kerja:** jawab sendiri tanpa buka catatan. Cek di bagian bawah. Target ≥ 80% (16 dari 20).

---

## Bagian A — Pilihan Ganda (10 soal)

**1.** Direktori `/etc` digunakan untuk menyimpan...
- A. Log sistem
- B. Konfigurasi sistem & aplikasi
- C. Biner aplikasi
- D. Data runtime

**2.** Perintah `chmod 640 file.txt` menghasilkan...
- A. `-rw-r-----`
- B. `-rw-r--r--`
- C. `-rwxr-x---`
- D. `-rw-------`

**3.** Sinyal `SIGTERM` vs `SIGKILL` yang benar adalah...
- A. Keduanya sama, hanya nama beda
- B. SIGTERM bisa di-trap, SIGKILL tidak
- C. SIGKILL lebih "sopan" dari SIGTERM
- D. SIGTERM hanya untuk foreground

**4.** Unit systemd `Type=notify` mengindikasikan...
- A. Service memberi tahu systemd saat siap via sd_notify
- B. Service mengirim email notifikasi
- C. Service auto-restart saat ada perubahan
- D. Service aktif setiap kali notifikasi sistem

**5.** One-liner `ps -eo pid,pcpu,cmd --sort=-pcru | head` bermaksud...
- A. Lihat semua PID
- B. Lihat 10 proses dengan CPU usage tertinggi
- C. Lihat 10 proses dengan memory tertinggi
- D. Hapus 10 proses

**6.** `du -sh /var/log` artinya...
- A. Ukuran total /var/log (summary, human-readable)
- B. Detail ukuran tiap file di /var/log
- C. Symbolic link /var/log
- D. Hapus /var/log

**7.** `chmod u+x,g-w file.sh` berarti...
- A. Hapus execute user, tambah write group
- B. Tambah execute user, hapus write group
- C. Tambah execute semua, hapus write semua
- D. Set 775

**8.** `set -euo pipefail` di skrip bash berfungsi untuk...
- A. Mempercepat eksekusi
- B. Berhenti saat ada error, pipe gagal, variabel tak di-set
- C. Enable syntax highlighting
- D. Hanya jalan di ZSH

**9.** `ssh -L 8080:db:5432 bastion` artinya...
- A. Buka port 8080 di bastion yang terusan ke db:5432
- B. Buka port 8080 di Mac yang terusan ke db:5432 via bastion
- C. Buka port 5432 di Mac yang terusan ke 8080 di bastion
- D. Sama dengan `ssh -R`

**10.** `find /var -size +1M` mencari...
- A. Semua file di /var
- B. File di /var yang ukurannya persis 1 MB
- C. File di /var dengan ukuran > 1 MB
- D. Direktori di /var

---

## Bagian B — Perintah (4 soal)

**11.** Tulis one-liner untuk menampilkan **10 IP yang paling sering** mengakses endpoint dan status 5xx dari `access.log`.

**12.** Tulis skrip bash yang:
- Terima argumen path direktori
- Log ke `stderr` setiap file `.log` di direktori tersebut yang lebih dari 100 MB
- Exit 0 (informasi) atau exit 1 (ada masalah)

**13.** Bagaimana cara melihat **informal** "siapa yang sedang login dan dari mana"?

**14.** Tulis pipeline `jq` yang:
- Dari data `data.json` berisi array objek `{name, score}`
- Mengembalikan nama yang skornya > 80
- Output dipisah newline

---

## Bagian C — Skenario (2 soal)

**15.** Seorang rekan mengatakan: "Server hang, CPU 100%." Prosedur diagnosis langkah awal yang Anda lakukan (maksimal 6 langkah, urut & singkat).

**16.** Anda diminta menonaktifkan service `app.service` yang berjalan terus restart. Apa bedanya:
- `kill -9 $(pidof app)`
- `systemctl stop app.service`
- `systemctl disable --now app.service`

Kapan pakai yang mana?

---

## Bagian D — Troubleshooting (2 soal)

**17.** User A login SSH gagal `Permission denied (publickey)`. Daftar cek yang sistematis.

**18.** Anda menulis skrip backup yang dijalankan via cron. Cron memanggilnya, tapi tidak pernah ada file backup yang muncul. Duga şi cek.

---

## Bagian E — Esai Singkat (2 soal)

**19.** Jelaskan kenapa `kill -KILL` bukan pilihan pertama. Apa konsekuensi jika langsung dipakai?

**20.** Anda punya 200 server on-prem. Anda diminta "rotating logout semua user yang sudah 90 hari tidak ganti password." Bayangkan skrip bash-nya (boleh pseudocode) dan jelaskan pertimbangan utamanya.

---

## Kunci Jawaban

### A — Pilihan Ganda
1. **B** — `/etc` = konfigurasi
2. **A** — 6=rw, 4=r--, 0=---
3. **B** — SIGTERM bisa di-trap (graceful), SIGKILL tidak (paksa)
4. **A** — `Type=notify` = daemon memberi tahu systemd
5. **B** — sort by CPU desc, head 10 — *catatan: ada typo "-pcru" di soal, yang benar `-pcpu`*
6. **A** — `-s` summary, `-h` human-readable
7. **B** — `u+x` tambah execute untuk user, `g-w` hapus write untuk group
8. **B** — combination: `-e` exit on error, `-u` error unset var, `-o pipefail` pipe failure naik
9. **B** — `-L` local forward: buka port di sisi lokal (Mac)
10. **C** — `+1M` artinya lebih besar dari 1 MB

### B — Perintah
11. ```bash
    awk '$9 ~ /^5[0-9][0-9]$/ {print $1}' access.log \
      | sort | uniq -c | sort -rn | head
    ```
    (Variasi: pisahkan field dengan `-F\"` jika format lengkap dengan path ber-kutip.)

12. ```bash
    #!/usr/bin/env bash
    set -euo pipefail
    DIR="${1:?usage: $0 <dir>}"
    found=0
    while IFS= read -r -d '' f; do
      echo "BESAR: $f" >&2
      found=1
    done < <(find "$DIR" -type f -name "*.log" -size +100M -print0)
    exit "$found"
    ```

13. ```bash
    who
    # atau lebih lengkap:
    w
    # atau audit-style:
    last -n 20
    ```

14. ```bash
    jq -r '.[] | select(.score > 80) | .name' data.json
    ```

### C — Skenario
15. Urutan:
    1. `top` / `htop` untuk lihat proses paling CPU
    2. `ps -eo pid,pcpu,cmd --sort=-pcpu | head` untuk lihat PID & command
    3. `pidstat -p <PID> 1` untuk lihat breakdown CPU proses tersebut
    4. Untuk service: `systemctl status <unit>` + `journalctl -u <unit> -n 100`
    5. Cek apakah ada children: `ps --forest -p <PID>`
    6. Tindakan: `kill -TERM <PID>` (SIGTERM sopan); hanya `kill -KILL` jika tidak turun

    (Urutan boleh beda, poin penting: identifikasi dulu, baru mitigasi.)

16. Bedanya:
    - `kill -9 $(pidof app)`: hanya membunuh proses saat ini. systemd akan restart (kalau `Restart=on-failure`). TIDAK menyelesaikan masalah.
    - `systemctl stop app.service`: menghentikan service secara managed. systemd mencatat, tidak restart. langkah paling tepat untuk hentikan.
    - `systemctl disable --now app.service`: stop + nonaktif auto-start saat boot. Gunakan saat permanent disable.

    Untuk "service terus restart": pakai `systemctl stop` dulu. Jika memang harus nonaktif permanen, `disable --now`.

### D — Troubleshooting
17. Cek berurutan:
    1. **Apakah key ada di client?** `ls -la ~/.ssh/id_ed25519`
    2. **Mode key/folder benar?** `chmod 700 ~/.ssh`, `chmod 600 ~/.ssh/id_ed25519`
    3. **Apakah pubkey ada di server?** `cat ~/.ssh/authorized_keys` di server
    4. **Mode authorized_keys?** `chmod 600 ~/.ssh/authorized_keys`
    5. **Apakah server menerima pubkey?** `sudo sshd -T | grep -i pubkey` (harus `yes`)
    6. **Apakah PasswordAuthentication=no** dan pubkey tidak cocok? `sudo sshd -T | grep -i passwordauth`
    7. **Logging di server:** `sudo journalctl -u ssh -n 50` — cari "Authentication refused"
    8. **Coba verbose:** `ssh -vvv user@host` — lihat "Offering public key", "Authentications that can continue"

18. Duga dan cek:
    - **PATH tidak ada**: cron tidak load `.bashrc`. Taruh full path ke script dan tools.
    - **Permission**: file backup ditulis ke dir yang tidak writable. Cek `>` direction.
    - **Cron syntax**: format salah. `crontab -l` lalu `MAILTO=root` untuk lihat error.
    - **Service tidak jalan**: `service cron status` / `systemctl status cron`
    - **Log cron**: `grep CRON /var/log/syslog` atau `journalctl -u cron`
    - **Lakukan manual**: `bash /path/script.sh` — jalan? Environment-nya beda.

### E — Esai
19. **`SIGKILL` (kill -9) tidak bisa di-trap** oleh proses. Proses tidak diberi kesempatan:
    - Menutup file dengan benar (data bisa corrupt)
    - Membersihkan lock/socket
    - Memberitahu peer (mis. database, message queue)
    - Flush buffer

    Jika langsung `-KILL` terus-menerus, log bisa korup, file lock tertinggal, peer bingung. Yang tepat: `SIGTERM` → tunggu → jika masih ada, `SIGKILL`.

20. Skrip:
    - Iterasi user dari `/etc/passwd` (UID >= 1000)
    - Baca `chage -l <user>` untuk last password change
    - Jika > 90 hari, paksa logout: `pkill -u <user>` (atau `loginctl terminate-user <user>`)
    - Untuk SSH session: `pkill -u <user> -t sshd` (hati-hati jangan bunuh sesi `root`)
    - Pastikan audit log: `logger -p auth.info "force logout <user>"`
    - Untuk ws/sftp: `passwd -l <user>` (lock akun) agar tidak bisa login lagi

    Pertimbangan:
    - **Dampak**: cron/job user ikut terhenti — komunikasikan sebelumnya
    - **Idempoten**: cek dulu apakah sudah pernah logout
    - **Audit**: catat ke syslog untuk compliance
    - **Whitelist**: skip user service (www-data, mysql, dsb.)
    - **Bukan hari-H**: dry-run dulu, kirim warning
    - **Batasi blast radius**: eksekusi per-user, jangan `pkill` masal

---

## Penilaian

| Benar | Skor |
|---|---|
| 18–20 | Expert — lanjut Modul 0.2 |
| 16–17 | Lulus — boleh lanjut, perbaiki yang salah |
| 12–15 | Belum lulus — ulang materi, kerjakan ulang latihan |
| < 12 | Ulangi semua materi, lanjut mentor |
