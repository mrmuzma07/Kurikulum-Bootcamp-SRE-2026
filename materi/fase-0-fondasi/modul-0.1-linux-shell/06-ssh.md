# 06 — SSH

> Login ke remote server, file transfer, tunneling — semua tanpa takut.

## Tujuan
- Setup key-based auth tanpa password
- Pakai `~/.ssh/config` untuk alias dan opsi
- Transfer file pakai `scp` & `rsync`
- Tunneling `-L`, `-R`, `-D` (dasar)

## 1. Pasang Key

```bash
ssh-keygen -t ed25519 -C "macbook-mu@2024"
# Output: ~/.ssh/id_ed25519 (private) dan ~/.ssh/id_ed25519.pub (public)

# Ubah algoritma lama ke ed25519
# Ed25519 pendek, cepat, aman — default modern.

# Lihat isi public key
cat ~/.ssh/id_ed25519.pub
```

**Mode kritikal:**
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/config     # kalau ada
# SSH tolak key/folder dengan mode_permissions yang salah
```

## 2. Copy Public Key ke Server

```bash
# Cara 1: ssh-copy-id
ssh-copy-id alice@server.example.com

# Cara 2: manual
cat ~/.ssh/id_ed25519.pub | ssh alice@server "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Login
ssh alice@server.example.com
```

`authorized_keys` adalah file di **server** yang berisi daftar public key yang dipercaya.

## 3. `~/.ssh/config` — Alias & Opsi

```ini
# ~/.ssh/config
Host prod-1
    HostName 10.0.0.11
    User alice
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host prod-2
    HostName 10.0.0.12
    User alice
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump prod-1

Host bastion
    HostName bastion.example.com
    User alice

Host internal-*
    ProxyJump bastion
    User alice

Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

```bash
ssh prod-1                # langsung pakai alias
ssh internal-web-01       # otomatis leap dengan bastion
```

## 4. Transfer File

```bash
# scp = secure cp
scp file.txt alice@prod-1:/tmp/
scp -r build/ alice@prod-1:/var/www/
scp alice@prod-1:/var/log/app.log ./local-copy.log

# rsync = transfer delta, jauh lebih cepat untuk repeated
rsync -avz ./localdir/ alice@prod-1:/srv/app/
rsync -avz --delete ./localdir/ alice@prod-1:/srv/app/  # HATI-HATI: --delete sinkron persis
rsync -avz -e ssh ./localdir/ alice@prod-1:/srv/app/
```

**Tips:**
- `-a` archive (preserve permission, symlink, time)
- `-v` verbose
- `-z` compress
- `--progress` untuk lihat progres
- `--dry-run` (-n) untuk uji

## 5. Tunneling

### Local Forward (`-L`)
Buka port di **Mac** yang dialihkan ke server tujuan via SSH.

```bash
ssh -L 8080:internal-db:5432 bastion
# Akses localhost:8080 di Mac -> reach ke internal-db:5432 via bastion
```

Saat develop: `psql -h localhost -p 8080` di Mac = `psql -h internal-db -p 5432`.

### Remote Forward (`-R`)
Buka port di **server** yang dialihkan ke sisi Mac.

```bash
ssh -R 9000:localhost:3000 server
# Akses server:9000 -> reach ke Mac:3000
```

Berguna saat demo dari local yang firewallnya ketat.

### Dynamic Forward (`-D`) → SOCKS Proxy
```bash
ssh -D 1080 server
# Set browser/proxy ke SOCKS5 localhost:1080
```

## 6. SSH Key Auth Server-side

Di server, `/etc/ssh/sshd_config`:
```
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
PermitRootLogin prohibit-password     # key only
ChallengeResponseAuthentication no
UsePAM yes
```

```bash
sudo systemctl reload sshd            # atau ssh, tergantung distro
```

**Wajib nonaktifkan password auth** di production on-prem.

## 7. Hardening Server SSH

```bash
# Fail2ban: blok IP setelah beberapa percobaan gagal
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban

# Batasi IP yang boleh SSH (UFW)
sudo ufw allow from 10.0.0.0/16 to any port 22
```

## 8. Tips Praktis

```bash
# Jalankan command di remote tanpa shell interaktif
ssh prod-1 "docker ps"

# Port forwarding sesekali
ssh -L 5432:db:5432 prod-1             # di background: -fN
ssh -fN -L 5432:db:5432 prod-1

# Sesi layar di remote (instalasi lama)
ssh -t prod-1 "tmux attach"

# SOCKS proxy multiple
ssh -fN -D 1080 prod-1
```

## 9. Troubleshoot Umum

| Masalah | Solusi |
|---|---|
| `Permission denied (publickey)` | Cek `~/.ssh` mode 700, key 600, key ada di `authorized_keys` |
| `Host key verification failed` | Key server berubah; cek fingerprint, hapus `~/.ssh/known_hosts` baris terkait |
| `ssh: connect to host x port 22: Connection refused` | SSH tidak jalan atau firewall |
| `Bash: command not found` di remote | `PATH` berbeda; panggil full path atau `source /etc/profile` |
| `KexAlgorithms mismatch` | Server lawas; edit `~/.ssh/config` atau upgrade server |

Lihat verbose:
```bash
ssh -vvv prod-1
```

## 10. Agent & Forwarding

```bash
# macOS sudah jalan ssh-agent otomatis
ssh-add -l                  # list keys
ssh-add ~/.ssh/id_ed25519   # tambah key
ssh-add -K ~/.ssh/id_ed25519  # macOS keychain

# Forward agent (hati-hati di shared host)
ssh -A prod-1
# Di prod-1: ssh-add -l akan tampilkan key kita
```

## Latihan Cepat

```bash
# 1. Set key
ssh-keygen -t ed25519 -C "latihan"
ssh-copy-id alice@localhost   # jika punya user 'alice' lokal

# 2. Setup config
echo "Host lab
    HostName 127.0.0.1
    User alice
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes" >> ~/.ssh/config
ssh lab

# 3. Test scp
echo "test" > /tmp/x.txt
scp /tmp/x.txt lab:/tmp/upload-test.txt
ssh lab "cat /tmp/upload-test.txt"

# 4. Test rsync
mkdir -p /tmp/src && touch /tmp/src/{a,b,c}.txt
rsync -avz /tmp/src/ lab:/tmp/dst/
ssh lab "ls /tmp/dst"
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Login | `ssh user@host` |
| Pakai key | `ssh-keygen` + `ssh-copy-id` |
| Alias | `~/.ssh/config` |
| Transfer file | `scp`, `rsync` |
| Tunnel ke internal | `ssh -L` |
| Troubleshoot | `ssh -vvv` |

## Cek Pemahaman

1. Kenapa mode `~/.ssh` (700) dan `id_ed25519` (600) krusial?
2. Beda `ssh -L 8080:db:5432` vs `ssh -R`?
3. Kapan `ProxyJump` berguna?
4. Kenapa `PasswordAuthentication no` di server?
