# 03 — Proses & Systemd

> Apa yang jalan, siapa yang punya, dan bagaimana memulai/mematikan dengan benar.

## Tujuan
- Baca `ps`/`top`, cari proses, lihat konsumsi resource
- Kirim sinyal yang tepat (`TERM`, `KILL`, `HUP`)
- Bikin dan kelola systemd unit

## 1. Mengintip Proses

```bash
ps                              # proses di shell saat ini
ps aux                          # semua proses, format BSD
ps -ef                          # semua proses, format System V
ps -ef | grep nginx             # filter
ps -u alice -o pid,cmd,etime    # proses user 'alice', kolom tertentu
ps axo pid,ppid,cmd,pcpu,pmem   # kolom custom
```

**Kolom penting:**
- `PID` — process ID
- `PPID` — parent PID
- `STAT` — state: `R` running, `S` sleeping, `D` disk sleep (perlu H/W), `Z` zombie, `T` stopped
- `ETIME` — sudah berapa lama jalan
- `PCPU`/`PMEM` — pakai CPU/memori
- `VSZ`/`RSS` — virtual/resident memory

## 2. `top` & `htop`

```bash
top
# tombol: k=kill PID, P=sort CPU, M=sort MEM, c=command full, 1=cpu per core
htop       # lebih user-friendly, perlu install
```

Alternatif modern:
```bash
btop        # paling cantik
glances
```

## 3. Sinyal Proses

Sinyal = cara kita ngomong ke proses. Jangan langsung `kill -9`.

| Sinyal | Nomor | Arti |
|---|---|---|
| SIGHUP | 1 | reload config (nginx, sshd) |
| SIGINT | 2 | interupsi (Ctrl+C) |
| SIGTERM | 15 | minta berhenti dengan baik (default `kill`) |
| SIGKILL | 9 | matikan paksa — memberi proses tidak kesempatan cleanup |
| SIGSTOP | 19 | pause |
| SIGCONT | 18 | lanjutkan |

```bash
kill 1234               # SIGTERM (default)
kill -TERM 1234         # eksplisit
kill -HUP $(pidof nginx) # reload nginx
killall nginx           # by name
pkill -f "python app.py" # by pattern
kill -9 1234            # SIGKILL — upaya terakhir
```

**Urutan etika:** `SIGTERM` → tunggu 5–10s → `SIGKILL`.

## 4. Foreground, Background, Daemon

```bash
python app.py                # foreground; Ctrl+C = SIGINT
python app.py &              # background; PID dicetak
jobs                         # lihat job di shell ini
fg %1                        # bawa ke foreground
bg %1                        # lanjutkan di background
nohup python app.py &        # tahan saat shell tutup
disown                       # lepas dari job table
```

Zombie process: proses yang sudah mati tapi parent belum baca statusnya. Tidak makan resource, hanya PID. Biasanya selesai saat parent reap.

## 5. `systemd` — Penjaga Service

```bash
systemctl status nginx              # status lengkap
systemctl start nginx               # jalan
systemctl stop nginx                # stop
systemctl restart nginx             # restart
systemctl reload nginx              # reload config saja (kalau support)
systemctl enable nginx              # auto-start saat boot
systemctl disable nginx
systemctl list-units --type=service --state=running
systemctl list-unit-files --state=enabled
```

**Herarki runlevel** di systemd:
- poweroff.target, rescue.target, multi-user.target, graphical.target
- ganti default: `sudo systemctl set-default multi-user.target`

Lihat kenapa service gagal:
```bash
systemctl status app.service
journalctl -u app.service -n 50 --no-pager
journalctl -u app.service --since "10 minutes ago"
```

## 6. Log dengan `journalctl`

```bash
journalctl                              # semua log
journalctl -f                           # follow (seperti tail -f)
journalctl -u nginx                     # filter unit
journalctl -u nginx -f                  # follow + unit
journalctl -p err                       # priority >= error
journalctl --since "2024-01-01" --until "2024-01-02"
journalctl -k                           # kernel log
journalctl -o json | jq '._HOSTNAME'    # format JSON
```

**Persistent journal:** biasanya `/var/log/journal/`. Pengaturan di `/etc/systemd/journald.conf`.

## 7. Tulis Unit Sendiri

Mis. aplikasi `/opt/app/server.py` mau dijalankan oleh user `app` saat boot.

### `/opt/app/server.py` (pura-pura)
```python
#!/usr/bin/env python3
import time, signal, sys, logging
logging.basicConfig(level=logging.INFO)
# jangan pakai port < 1024 tanpa root/akun tertentu
print("app up", flush=True)
while True:
    time.sleep(10)
```

### `/etc/systemd/system/myapp.service`
```ini
[Unit]
Description=My Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=app
WorkingDirectory=/opt/app
ExecStart=/usr/bin/python3 /opt/app/server.py
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment=PORT=8080

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl status myapp
```

**Tipe service:**
- `Type=simple` — default, proses ExecStart jadi main pid
- `Type=exec` — mirip, tapi ExecStart di-fork baru? Tidak: jalan langsung
- `Type=forking` — daemon yang double-fork
- `Type=oneshot` — jalan sekali, mis. migrasi DB
- `Type=notify` — daemon kasih tahu "siap" via sd_notify
- `Type=notify` plus `WatchdogSec=` — health check oleh systemd

## 8. `systemd` Timer (Pengganti Cron)

Timer lebih baik dari cron (jadwal termanage, log dari journal, more fleksibel).

### `/etc/systemd/system/backup.service`
```ini
[Unit]
Description=Backup Direktori

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

### `/etc/systemd/system/backup.timer`
```ini
[Unit]
Description=Jalankan backup tiap hari 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers --all
```

## 9. `cron` (Warisan, Masih Sering)

Format: `m h dom mon dow command`
```
30 2 * * * /usr/local/bin/backup.sh   # tiap hari 02:30
*/5 * * * * /usr/local/bin/check.sh   # tiap 5 menit
```

```bash
crontab -e         # edit cron user
crontab -l         # lihat
service cron status
```

**Cron tidak load environment dari shell.** Taruh env di atas command:
```
*/5 * * * * PATH=/usr/bin:/bin FOO=bar /usr/local/bin/check.sh
```

## 10. Cek Cepat Troubleshoot

| Gejala | Cek |
|---|---|
| Port tidak bisa di-bind | proses lain? `ss -ltnp` |
| Service crash terus | `journalctl -u xxx -n 200` |
| CPU 100% | `top` lalu `ps -ef` cari root process |
| App sering restart | `Restart=on-failure` → cek logika crash |
| Disk penuh | `df -h` lalu `du -sh /*` |

## Latihan Cepat

```bash
# 1. Lihat pohon proses
ps auxf          # ascii tree
# atau
pstree -p

# 2. Stress & kill
sleep 1000 &
PID=$!
kill -TERM $PID
wait; echo "exit=$?"

# 3. Lihat boot log
journalctl -b -n 50

# 4. Lihat timer yang ada
systemctl list-timers

# 5. Lihat service yang gagal
systemctl --failed
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Lihat proses | `ps aux`, `top`, `htop` |
| Hentikan proses | `kill PID` (SIGTERM dulu) |
| Kelola service | `systemctl start/stop/restart/enable` |
| Lihat log | `journalctl -u xxx` |
| Penjadwalan | `systemd` timer (disarankan) atau cron |

## Cek Pemahaman

1. Kenapa `kill -9` bukan pilihan pertama?
2. Bedanya `systemctl stop` vs `systemctl disable`?
3. Kapan perlu `Type=notify`?
4. Cron vs systemd timer — mana pilihanmu dan kenapa?
