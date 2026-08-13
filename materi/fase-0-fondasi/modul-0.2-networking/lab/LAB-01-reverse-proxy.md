# LAB-01 — Reverse Proxy Caddy + TLS Otomatis

> **Target:** menjalankan Caddy sebagai reverse proxy untuk ≥ 2 service, salah satunya dengan HTTPS (TLS internal), dan menguji host-based routing — pola yang nanti menjadi Ingress Kubernetes.

## Latar Belakang
Di production on-prem, jarang app langsung di-expose. Ada reverse proxy/Ingress di depan yang terminasi TLS & routing berdasarkan nama. Lab ini meniru pola itu dengan Caddy di VM OrbStack sebelum kita bertemu Ingress Kubernetes di Modul 2.1.

## Prasyarat
- [ ] LAB-01 modul 0.1 selesai (VM `lab01` bisa di-SSH tanpa password)
- [ ] Sudah baca [03-http-proxy](../03-http-proxy.md) & [02-tcp-udp-tls](../02-tcp-udp-tls.md)
- [ ] Repo `sre-bootcamp/m0.2` di GitLab sudah dibuat

## Waktu
± 2 jam

## Langkah

### 1. Jalankan Dua Service Sederhana di VM

```bash
ssh lab01 <<'EOF'
set -e
# Service 1: HTTP server di port 8080 (dengan tag agar dikenali)
cat >/tmp/srv1.py <<'PY'
from http.server import BaseHTTPRequestHandler, HTTPServer
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.send_header("Content-Type","text/plain"); self.end_headers()
        self.wfile.write(b"hello from SERVICE-1 (app)\n")
HTTPServer(("0.0.0.0",8080),H).serve_forever()
PY

# Service 2: port 3000
cat >/tmp/srv2.py <<'PY'
from http.server import BaseHTTPRequestHandler, HTTPServer
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.send_header("Content-Type","text/plain"); self.end_headers()
        self.wfile.write(b"hello from SERVICE-2 (grafana-like)\n")
HTTPServer(("0.0.0.0",3000),H).serve_forever()
PY

nohup python3 /tmp/srv1.py >/tmp/srv1.log 2>&1 &
nohup python3 /tmp/srv2.py >/tmp/srv2.log 2>&1 &

# Verifikasi masing-masing dengar di 0.0.0.0
ss -tlnp | grep -E ':8080|:3000'
EOF
```

Pastikan dua baris `0.0.0.0:8080` dan `0.0.0.0:3000` muncul. Kalau `127.0.0.1`, service tidak bisa diakses lewat proxy dari host lain.

### 2. Pasang Caddy

```bash
ssh lab01 <<'EOF'
set -e
if ! command -v caddy >/dev/null 2>&1; then
  # Binari ARM64 resmi
  sudo install -d /usr/local/bin
  curl -fsSL 'https://caddyserver.com/api/download?os=linux&arch=arm64' -o /tmp/caddy
  sudo install -m 0755 /tmp/caddy /usr/local/bin/caddy
fi
caddy version
EOF
```

### 3. Siapkan Nama Internal

Gunakan `/etc/hosts` di Mac (bukan VM) agar nama resolve ke IP VM:

```bash
VM_IP=$(ssh lab01 'hostname -I | awk "{print \$1}"')
echo "$VM_IP app.lab.local grafana.lab.local" | sudo tee -a /etc/hosts
getent hosts app.lab.local         # harus balas IP VM
```

### 4. Caddyfile — Host-Based Routing + TLS Internal

```bash
ssh lab01 <<'EOF'
set -e
sudo install -d /etc/caddy
sudo tee /etc/caddy/Caddyfile >/dev/null <<'CADDY'
{
    # TLS internal: Caddy jadi CA-nya sendiri untuk nama .local
    # (tidak butuh Let's Encrypt / domain publik)
}

app.lab.local {
    tls internal
    reverse_proxy localhost:8080
}

grafana.lab.local {
    tls internal
    reverse_proxy localhost:3000
}
CADDY

# Caddy jalan sebagai service sederhana (foreground dulu untuk lihat log)
sudo /usr/local/bin/caddy run --config /etc/caddy/Caddyfile --adapter caddyfile &
sleep 2
ss -tlnp | grep -E ':80|:443'
EOF
```

Catatan: untuk nama domain **publik**, Caddy otomatis minta sertifikat Let's Encrypt tanpa `tls internal`. Di sini pakai `.local` jadi gunakan TLS internal (Caddy generate CA sendiri).

### 5. Uji dari Mac

```bash
# HTTP (redirect ke HTTPS otomatis)
curl -v http://app.lab.local/                # lihat 308 redirect

# HTTPS (Caddy CA internal belum dipercaya Mac → --insecure)
curl -k https://app.lab.local/               # harus: "hello from SERVICE-1 (app)"
curl -k https://grafana.lab.local/           # harus: "hello from SERVICE-2 ..."

# Lihat sertifikat internal
echo | openssl s_client -connect app.lab.local:443 -servername app.lab.local 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
# issuer: Caddy Local Authority ← CA internal Caddy
```

### 6. Percayai CA Internal (opsional, agar tanpa `-k`)

```bash
ssh lab01 'sudo cat /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt' \
  > ~/caddy-root-ca.crt
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/caddy-root-ca.crt
curl https://app.lab.local/                  # sekarang tanpa -k, valid
```

### 7. Jadikan Service systemd (Idempotent)

```bash
ssh lab01 <<'EOF'
set -e
sudo tee /etc/systemd/system/caddy.service >/dev/null <<'UNIT'
[Unit]
Description=Caddy reverse proxy
After=network.target

[Service]
ExecStart=/usr/local/bin/caddy run --config /etc/caddy/Caddyfile --adapter caddyfile
ExecReload=/usr/local/bin/caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
Restart=on-failure
User=root

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now caddy
sudo systemctl status caddy --no-pager
sudo journalctl -u caddy -n 20 --no-pager
EOF
```

### 8. Simpan ke Repo

```bash
cd ~/Developer/Playgrounds/devops/sre/training01   # sesuaikan
mkdir -p materi/fase-0-fondasi/modul-0.2-networking/lab
# Salin Caddyfile ke repo:
ssh lab01 'cat /etc/caddy/Caddyfile' > lab/Caddyfile
git add lab/Caddyfile
git commit -m "m0.2: Caddyfile reverse proxy + TLS internal"
```

## Acceptance Criteria

- [ ] Dua service dengar di `0.0.0.0:8080` & `0.0.0.0:3000` (`ss -tlnp`)
- [ ] Caddy running sebagai systemd service (`systemctl status caddy` active)
- [ ] `curl -k https://app.lab.local/` → balasan service 1
- [ ] `curl -k https://grafana.lab.local/` → balasan service 2
- [ ] Sertifikat internal terlihat lewat `openssl s_client`
- [ ] HTTP otomatis redirect ke HTTPS (lihat `308` di `curl -v`)
- [ ] `Caddyfile` tersimpan di repo GitLab

## Troubleshooting

| Gejala | Solusi |
|---|---|
| `curl: (60) certificate verify failed` | Normal untuk CA internal; pakai `-k` atau install CA root (langkah 6) |
| `bind: address already in use :80` | Caddy bentrok service lain: `ss -tlnp \| grep ':80'`, matikan dulu |
| `app.lab.local` resolve ke `127.0.0.1` | Cek `/etc/hosts` — pastikan baris untuk VM IP ada, bukan localhost |
| Reverse proxy `502` | Service backend mati: `ss -tlnp` pastikan port 8080/3000 listening |
| Caddy tidak otomatis reload | `sudo systemctl reload caddy`; cek `journalctl -u caddy` |
| TLS internal gagal generate | Pastikan direktori `/var/lib/caddy` writable oleh user Caddy |

## Lanjut
[LAB-02 — DNS Lokal](LAB-02-dns-lokal.md)