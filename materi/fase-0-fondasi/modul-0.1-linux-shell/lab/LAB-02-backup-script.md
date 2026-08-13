# LAB-02 — Backup Script + Cron/Systemd Timer

> **Target:** menulis skrip backup **idempotent**, menjalankannya dengan **systemd timer**, dan **memulihkan** data pada insiden simulasi.

## Latar Belakang
Recovery > Backup. Tapi kalau backup-nya mumpung, recovery jadi mudah. Skrip ini adalah pondasi LAB-02.

## Prasyarat
- [ ] LAB-01 selesai (VM `lab01` bisa di-SSH)
- [ ] Repo `sre-bootcamp/m0.1` di GitLab sudah dibuat

## Waktu
± 2 jam

## Langkah

### 1. Repo & Struktur

```bash
cd ~/Developer           # atau folder kerja
mkdir -p sre-bootcamp/m0.1
cd sre-bootcamp/m0.1
git init
git remote add origin git@gitlab.com:<username>/sre-bootcamp.git
mkdir -p lab
```

### 2. Siapkan Data Uji

```bash
ssh lab01 <<'EOF'
set -e
sudo mkdir -p /srv/data/app
sudo chown -R ubuntu:ubuntu /srv/data
mkdir -p /srv/data/app

# 100 file dummy
for i in $(seq 1 100); do
  echo "record $i — $(date -Is)" > /srv/data/app/file_$i.txt
done

ls /srv/data/app | head
EOF
```

### 3. Tulis Skrip Backup

`lab/backup.sh`:
```bash
#!/usr/bin/env bash
#
# backup.sh — Backup direktori ke arsip terkompresi dengan retensi.
#
# Env:
#   SRC          # direktori sumber (wajib)
#   DEST         # direktori tujuan (wajib)
#   RETENTION_DAYS  # default 7
set -euo pipefail
IFS=$'\n\t'

SRC="${1:-${SRC:-}}"
DEST="${2:-${DEST:-}}"
RETENTION_DAYS="${RETENTION_DAYS:-7}"

usage() {
  cat <<EOF
Usage: $0 <SRC> <DEST>     # atau env SRC, DEST
EOF
  exit 64
}

log() { printf '%s [%s] %s\n' "$(date -Is)" "$1" "$2" >&2; }

main() {
  [[ -z "$SRC"  ]] && { usage; }
  [[ -z "$DEST" ]] && { usage; }
  [[ -d "$SRC" ]] || { log ERROR "SRC not dir: $SRC"; exit 66; }

  command -v tar  >/dev/null || { log ERROR "tar tidak ada"; exit 69; }
  command -v find >/dev/null || { log ERROR "find tidak ada"; exit 69; }

  mkdir -p "$DEST"
  local stamp out
  stamp=$(date +%Y%m%d-%H%M%S)
  out="${DEST%/}/backup-${stamp}.tar.gz"

  log INFO "packing $SRC -> $out"
  tar -czf "$out" -C "$(dirname "$SRC")" "$(basename "$SRC")"

  log INFO "retention: drop >${RETENTION_DAYS}d from $DEST"
  find "$DEST" -maxdepth 1 -name "backup-*.tar.gz" -mtime +"$RETENTION_DAYS" -delete -print

  log INFO "done"
}

main "$@"
```

### 4. Skrip Jadi Executable & Tes Pertama

```bash
chmod +x lab/backup.sh

# Taruh di server
scp lab/backup.sh lab01:/usr/local/bin/backup.sh
ssh lab01 "chmod +x /usr/local/bin/backup.sh"

# Run
ssh lab01 "/usr/local/bin/backup.sh /srv/data/app /var/backups"
```

Verifikasi:
```bash
ssh lab01 "ls -lh /var/backups"
# Tampil backup-YYYYMMDD-HHMMSS.tar.gz
```

### 5. Idempotency Check

Jalankan 2x:
```bash
ssh lab01 "/usr/local/bin/backup.sh /srv/data/app /var/backups"
ssh lab01 "/usr/local/bin/backup.sh /srv/data/app /var/backups"
ssh lab01 "ls /var/backups | wc -l"   # bertambah 2
```

Backup baru setiap kali = OK. Beda dengan restore yang idempotent (lihat 11).

### 6. Systemd Service + Timer

Di VM, simpan unit:

`backup.service`:
```ini
[Unit]
Description=Backup Direktori via backup.sh

[Service]
Type=oneshot
Environment=RETENTION_DAYS=7
ExecStart=/usr/local/bin/backup.sh /srv/data/app /var/backups
Nice=10
IOSchedulingClass=best-effort
```

`backup.timer`:
```ini
[Unit]
Description=Jalankan backup.service setiap hari 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

Install:
```bash
ssh lab01 <<'EOF'
set -e
sudo tee /etc/systemd/system/backup.service >/dev/null <<'SVCEOF'
[Unit]
Description=Backup Direktori via backup.sh

[Service]
Type=oneshot
Environment=RETENTION_DAYS=7
ExecStart=/usr/local/bin/backup.sh /srv/data/app /var/backups
Nice=10
IOSchedulingClass=best-effort
SVCEOF

sudo tee /etc/systemd/system/backup.timer >/dev/null <<'TIMEREOF'
[Unit]
Description=Jalankan backup.service setiap hari 02:30

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
TIMEREOF

sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers | grep backup
EOF
```

Jalankan sekali manual (untuk uji):
```bash
ssh lab01 "sudo systemctl start backup.service"
ssh lab01 "sudo systemctl status backup.service"
ssh lab01 "sudo journalctl -u backup.service --no-pager -n 20"
```

### 7. (Bonus) Cron Sebagai Cadangan

```bash
ssh lab01 "crontab -l 2>/dev/null | { cat; echo '30 2 * * * /usr/local/bin/backup.sh /srv/data/app /var/backups'; } | crontab -"
ssh lab01 "crontab -l"
```

### 8. Simulasi Insiden: Hapus Data, Pulihkan

```bash
ssh lab01 <<'EOF'
set -e
# Cek backup tersedia
ls -lh /var/backups

# Ambil nama file backup terbaru
LATEST=$(ls -t /var/backups/backup-*.tar.gz | head -1)
echo "Restore from: $LATEST"

# HANCURKAN data
rm -rf /srv/data/app
ls /srv/data/app || echo "lost: OK, sekarang tidak ada"

# RESTORE
mkdir -p /srv/data
sudo tar -xzf "$LATEST" -C /srv/data
ls /srv/data/app | head
ls /srv/data/app | wc -l
EOF
```

Harus kembali: 100 file.

### 9. Uji Retensi

```bash
# Buat backup palsu yang berumur 8 hari
ssh lab01 <<'EOF'
set -e
touch -d "8 days ago" /var/backups/backup-old.tar.gz
ls -lh /var/backups
EOF

# Trigger service (yang menjalankan RETENTION_DAYS=7)
ssh lab01 "sudo systemctl start backup.service"
ssh lab01 "ls -lh /var/backups"
# Yang lama diharapkan hilang
```

### 10. Laporan

`lab/lab02-report.md`:
```markdown
# LAB-02 Report

## Skrip
- Lokasi: https://gitlab.com/<username>/sre-bootcamp/-/blob/main/m0.1/lab/backup.sh
- Penjadwalan: systemd timer (backup.timer)
- Retensi: 7 hari

## Hasil
- Backup berhasil: capture timestamp
- Restore berhasil: 100 file kembali
- Retensi membuang backup lama: ya

## Catatan
- Volume data: 100 file kecil
- Ukuran backup: ... KB
- Cron: [aktif / tidak]
```

## Acceptance Criteria

- [ ] `lab/backup.sh` ada di repo dan bisa di-`chmod +x`
- [ ] `backup.service` & `backup.timer` ter-install di VM
- [ ] `systemctl list-timers` menampilkan `backup.timer`
- [ ] `journalctl -u backup.service` menampilkan log sukses
- [ ] Restore dari backup membangkitkan 100 file ke `/srv/data/app`
- [ ] Backup berumur > 7 hari otomatis terhapus
- [ ] Laporan di `lab/lab02-report.md`

## Troubleshooting

| Gejala | Solusi |
|---|---|
| `find: missing argument` | Coba langsung tanpa `-print`; flag beda di BusyBox |
| `tar: warning: skipping header` | Arsip corrupt, ulangi backup |
| Timer tidak jalan | Cek `Persistent=true` + reboot; `systemctl list-timers --all` |
| Permission denied saat menulis | `sudo chmod 755 /var/backups`; jangan jalankan sebagai root jika tidak perlu |

## Lanjut
[LAB-03 — Debug Proses](LAB-03-debug-proses.md)
