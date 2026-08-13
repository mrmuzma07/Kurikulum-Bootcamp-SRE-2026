# LAB-03 — Debug Proses: CPU Hog & Process Yang Respawn

> **Target:** menemukan dan menghentikan proses yang bermasalah, membedakan **pembunuh proses** vs **penonaktif service**.

## Latar Belakang
Skenario produksi klasik: "CPU tiba-tiba 100%". Atau "pod/application restart terus". Tanpa `ps` + `top` + `systemctl`, kita cuma bisa menebak.

## Prasyarat
- [ ] LAB-01 selesai (VM `lab01`)
- [ ] Akses sudo di VM

## Waktu
± 90 menit

## Skenario A — CPU Hog

### 1. Buat Proses Jahat

```bash
ssh lab01 <<'EOF'
set -e
# CPU hog: dua loop yes /dev/null yang menampilkan 100% CPU
yes > /dev/null &
P1=$!
yes > /dev/null &
P2=$!
echo "P1=$P1 P2=$P2"
echo $P1 > /tmp/cpu-hog.pid
echo $P2 >> /tmp/cpu-hog.pid
EOF
```

Buka top di windows lain:
```bash
ssh lab01 "top -d 1"
```

### 2. Investigasi

```bash
# Lihat proses
ssh lab01 "ps -eo pid,ppid,user,pcpu,pmem,stat,etime,cmd --sort=-pcpu | head"

# Lihat tree
ssh lab01 "ps -ef --forest | head -30"

# Cari proses `yes`
ssh lab01 "pgrep -a yes"
```

### 3. Identifikasi

```bash
ssh lab01 "cat /tmp/cpu-hog.pid"
ssh lab01 "ps -fp $(cat /tmp/cpu-hog.pid)"
```

### 4. Mitigasi (Etika SIGTERM dulu)

```bash
ssh lab01 "kill -TERM $(cat /tmp/cpu-hog.pid)"
```

Tunggu 5 detik. Cek:
```bash
ssh lab01 "pgrep -a yes"     # harusnya tidak ada
ssh lab01 "ps -p \$(cat /tmp/cpu-hog.pid) 2>/dev/null"   # expected: no such PID
```

Jika proses masih ada → SIGKILL:
```bash
ssh lab01 "kill -KILL $(cat /tmp/cpu-hog.pid)"
```

### 5. Laporan

Catat di `lab/lab03-report.md`:
```
- Top 3 proses makan CPU saat insiden
- PID yang dibunuh
- Sinyal yang dipakai (SIGTERM / SIGKILL)
- Apakah `kill -KILL` dibutuhkan?
```

### 6. Bersihkan

```bash
ssh lab01 "pkill -f 'yes > /dev/null'"  # safety net
```

## Skenario B — Service Yang Respawn

### 1. Buat Service Jahat

```bash
ssh lab01 <<'EOF'
set -e
# Script yang exit 1 setelah 2 detik (memicu restart)
sudo tee /usr/local/bin/loon.sh >/dev/null <<'JAIL'
#!/usr/bin/env bash
echo "loon start: $(date -Is)" >&2
sleep 2
echo "loon crash" >&2
exit 1
JAIL
sudo chmod +x /usr/local/bin/loon.sh

# Unit systemd
sudo tee /etc/systemd/system/loon.service >/dev/null <<'SVCEOF'
[Unit]
Description=Loon Service (banyak restart)

[Service]
Type=simple
ExecStart=/usr/local/bin/loon.sh
Restart=on-failure
RestartSec=1
SVCEOF

sudo systemctl daemon-reload
sudo systemctl start loon.service
EOF
```

### 2. Investigasi

```bash
ssh lab01 "systemctl status loon.service"
# Lihat "Restart counter" naik

ssh lab01 "journalctl -u loon.service -n 30 --no-pager"
```

**Kesalahan pemula:** `kill -9 $(pidof loon.sh)` — gagal, karena systemd akan restart.

### 3. Mitigasi yang Tepat

```bash
ssh lab01 "sudo systemctl stop loon.service"
# atau
ssh lab01 "sudo systemctl kill --signal=TERM loon.service"

# Lihat hasilnya
ssh lab01 "systemctl status loon.service"
ssh lab01 "journalctl -u loon.service -n 5"
```

### 4. Nonaktifkan Sepenuhnya

```bash
ssh lab01 "sudo systemctl disable --now loon.service"
```

### 5. (Bonus) Lihat Resource Limit

```bash
ssh lab01 "systemctl show loon.service | grep -E 'Restart=|CPU|Memory'"
```

Tambahkan limit pada unit lalu restart:
```ini
[Service]
MemoryMax=100M
CPUQuota=50%
```

### 6. Laporan

Tambahkan ke `lab/lab03-report.md`:
```
- Gejala: service restart otomatis
- Tepat: `systemctl stop` (bukan kill PID)
- Restart counter akhir: ...
- Limit yang dipasang: ...
```

## Skenario C (Bonus) — Proses Zombie

```bash
ssh lab01 <<'EOF'
set -e
cat > /tmp/zombie.py <<'PY'
import os, time
pid = os.fork()
if pid == 0:
    # Child exit segera jadi zombie
    print("child exiting")
    os._exit(0)
# Parent tidak baca status child
while True:
    time.sleep(60)
PY
python3 /tmp/zombie.py &
EOF
```

Investigasi:
```bash
ssh lab01 "ps -eo pid,ppid,stat,cmd | awk 'NR==1 || \$2 != 1 && \$3 ~ /Z/'"
```

Cleanup:
```bash
ssh lab01 "pkill -f zombie.py"
```

## Output Akhir

`lab/lab03-report.md`:
```markdown
# LAB-03 Report

## Skenario A: CPU Hog
- PID: ...
- Durasi: ... detik
- Sinyal: SIGTERM (cukup / tidak)
- Top command: `yes > /dev/null`

## Skenario B: Service Respawn
- Unit: loon.service
- Restart counter: ...
- Tepat: `systemctl stop` & `disable`
- Limit: MemoryMax=100M, CPUQuota=50%

## Skenario C (Bonus): Zombie
- Apakah terlihat di stat? Z
- Clean up: pkill

## Pelajaran
- Bedakan service vs proses biasa
- Kill PID ≠ disable service
- Etika: SIGTERM dulu, SIGKILL upaya terakhir
```

## Acceptance Criteria

- [ ] Skenario A selesai, CPU kembali normal
- [ ] Skenario B selesai, service tidak lagi restart
- [ ] Laporan berisi PID, sinyal, command yang dipakai
- [ ] Tahu kapan pakai `kill` vs `systemctl stop`

## Quick Reference

| Gejala | Cek | Tepat |
|---|---|---|
| CPU 100% | `top`, `ps -eo --sort=-pcpu` | `kill -TERM <PID>` → `kill -KILL` |
| Service restart loop | `systemctl status X` | `systemctl stop X` / `disable --now` |
| Aplikasi hang | `journalctl -u X`, `ps --forest` | `systemctl restart X` |
| Zombie | `ps -o stat=...` | Reap dari parent; fix kode |

## Lanjut
[Evaluasi: Latihan & Kuis](../evaluasi/latihan.md)
