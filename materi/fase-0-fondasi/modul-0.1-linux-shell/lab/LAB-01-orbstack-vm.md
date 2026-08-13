# LAB-01 — Virtual Machine Sovereign via OrbStack

> **Target:** punya **VM Linux** yang diakses **SSH tanpa password** dari Mac, siap jadi rumah eksperimen.

## Latar Belakang
SRE on-prem selalu main dengan server Linux. Latihan di Mac, tapi **targetnya adalah Linux**. OrbStack memberi Linux VM Linux asli di Mac, ringan, native ARM64.

## Prasyarat
- [ ] MacBook Air M5 dengan macOS terbaru
- [ ] Homebrew terpasang
- [ ] Koneksi internet

## Waktu
± 90 menit (pertama kali)

## Langkah

### 1. Install OrbStack

```bash
brew install --cask orbstack
open -a OrbStack
```

Tunggu sampai UI OrbStack siap (status "Ready").

### 2. Buat Linux Machine

**Lewat UI:**
1. Klik **+ New Machine** → **Linux**.
2. Nama: `lab01`.
3. Image: `Ubuntu 22.04 LTS` (atau LTS terbaru).
4. CPU: 2, RAM: 2048 MB, Disk: 20 GB (cukup untuk lab).
5. Klik **Create**.

**Lewat CLI (lebih cepat):**
```bash
orb create ubuntu:22.04 lab01 --cpu 2 --memory 2G --disk 20G
orb list                # verifikasi
```

### 3. Akses Shell

```bash
orb shell lab01
```
 
Kamu sekarang ada di dalam VM Ubuntu. Cek:
```bash
uname -a
cat /etc/os-release
whoami
```

### 4. Set Hostname & Update

```bash
sudo hostnamectl set-hostname lab01
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server curl vim htop jq
```

### 5. Aktifkan SSH Server

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
ip addr show                # cari IP, mis. 192.168.97.2
```

### 6. Pasang SSH Key dari Mac

Di Mac (shell Lokal, **bukan** di dalam VM):
```bash
# Generate key jika belum ada
[[ -f ~/.ssh/id_ed25519 ]] || ssh-keygen -t ed25519 -C "lab01"

# Ambil IP VM
VM_IP=$(orb list -j | jq -r '.[] | select(.name=="lab01") | .ip | strings' | head -1)
echo "VM IP: $VM_IP"

# Copy key
ssh-copy-id ubuntu@$VM_IP

# Test
ssh ubuntu@$VM_IP "uname -a"
```

### 7. Alias `~/.ssh/config`

Tambahkan ke `~/.ssh/config`:
```ini
Host lab01
    HostName 192.168.97.2
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Sekarang:
```bash
ssh lab01
```

### 8. Hardening Dasar VM

```bash
ssh lab01
sudo apt install -y ufw fail2ban
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
sudo ufw status verbose
```

### 9. Snapshot Pertama

Di OrbStack UI: kanan-klik `lab01` → **Snapshot** → nama `baseline`.

Kebiasaan: **snapshot sebelum install apa pun yang signifikan**.

### 10. Otomasi: Salin `dotfiles` & Pasang Paket Penting

```bash
# Di Mac
rsync -avz --exclude='.git' ~/dotfiles/ lab01:~/dotfiles/

ssh lab01 "bash -lc 'set -e; source ~/.dotfiles/install.sh'"
```

(Setiap orang beda isi `dotfiles`; yang penting ada script yang bisa dijalankan ulang.)

## Acceptance Criteria

Centang semua:

- [ ] OrbStack terpasang & jalan
- [ ] VM `lab01` Ubuntu berjalan, CPU 2, RAM 2 GB
- [ ] `ssh lab01` dari Mac tanpa password
- [ ] `ssh lab01 "uname -a"` menampilkan `Linux lab01 ... aarch64`
- [ ] `sudo ufw status` aktif, rule SSH ada
- [ ] Snapshot `baseline` dibuat
- [ ] Repo `sre-bootcamp/m0.1` di GitLab berisi folder `lab/` (akan diisi lab berikutnya)

## Verifikasi Manual

```bash
ssh lab01 <<'EOF'
  set -e
  echo "== Status check =="
  uname -a
  uptime
  ss -ltn | head
  sudo ufw status | head
  df -h / | tail -1
  systemctl is-active ssh
  echo "== OK =="
EOF
```

## Troubleshooting

| Gejala | Solusi |
|---|---|
| `orb` command not found | Tambah OrbStack ke PATH: `open -a OrbStack` dulu hingga selesai |
| `ssh: connect to host ... port 22: Connection refused` | SSH belum jalan di VM; cek `sudo systemctl status ssh` |
| `Permission denied (publickey)` | Cek mode `~/.ssh` (700) dan key (600) |
| `Host key verification failed` | `ssh-keygen -R "$VM_IP"` lalu coba lagi |
| `orb` daftar tidak keluar IP | `orb list -j` ada key `ip` atau `ipv4`; sesuaikan `jq` |

## Output yang Dikirim

Di repo `sre-bootcamp/m0.1`, buat `lab/lab01-report.md`:
```markdown
# LAB-01 Report

## Bukti
- Output `uname -a`:
  ```
  Linux lab01 6.5.0-... aarch64 GNU/Linux
  ```
- Output `sudo ufw status`:
  ```
  Status: active
  ...
  ```
- Output `df -h /`

## Catatan
- IP VM: 192.168.97.X
- SSH alias: lab01
- Snapshot: baseline (timestamp)
```

## Lanjut
[LAB-02 — Backup Script](LAB-02-backup-script.md)
