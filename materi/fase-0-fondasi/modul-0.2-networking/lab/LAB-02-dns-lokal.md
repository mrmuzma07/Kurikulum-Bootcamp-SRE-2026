# LAB-02 — DNS Lokal dengan dnsmasq

> **Target:** mengganti `/etc/hosts` manual dengan DNS server lokal (dnsmasq) yang menyelesaikan nama internal ke IP VM, lengkap dengan SRV record untuk simulasi service discovery.

## Latar Belakang
`/etc/hosts` cukup untuk 2-3 nama. Tapi di cluster on-prem, puluhan service punya nama (grafana, prometheus, api, argo, …). Menulis `/etc/hosts` di tiap server = mimpi. dnsmasq memberi DNS lokal yang satu tempat ubah, semua dapat. Ini juga model mental untuk **CoreDNS** di Kubernetes (yang menyelesaikan nama Service).

## Prasyarat
- [ ] LAB-01 modul 0.1 selesai (VM `lab01`)
- [ ] Sudah baca [01-dns-cidr](../01-dns-cidr.md)
- [ ] LAB-01 modul 0.2 selesai (opsional, untuk integrasi nama dengan Caddy)

## Waktu
± 90 menit

## Langkah

### 1. Pasang dnsmasq di VM `lab01`

```bash
ssh lab01 <<'EOF'
set -e
sudo apt update -qq
sudo apt install -y dnsmasq dnsutils
dnsmasq --version | head -1
EOF
```

### 2. Konfigurasi DNS Lokal

dnsmasq membaca `/etc/hosts` VM **sekaligus** bisa beri record tambahan via config. Kita pakai domain `lab.local`.

```bash
ssh lab01 <<'EOF'
set -e
# Tambahkan mapping nama→IP di /etc/hosts VM
VM_IP=$(hostname -I | awk '{print $1}')
grep -q lab.local /etc/hosts || echo "$VM_IP app.lab.local grafana.lab.local api.lab.local" | sudo tee -a /etc/hosts

# Konfigurasi dnsmasq
sudo tee /etc/dnsmasq.d/lab.conf >/dev/null <<'CONF'
# Dengar hanya di interface lokal VM (jangan ganggu sistem)
listen-address=127.0.0.1
# Domain yang dikelola dnsmasq ini
domain=lab.local
# Jangan baca /etc/resolv.conf upstream untuk lab.local
local=/lab.local/
# Forward query lain ke resolver publik
server=1.1.1.1
# Aktifkan SRV record contoh (service discovery): _kube._tcp → api.lab.local:6443
srv-host=_kube._tcp.lab.local,api.lab.local,6443,10,10
# Logging untuk debugging
log-queries
CONF

sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq --no-pager
EOF
```

### 3. Uji dari VM

```bash
ssh lab01 <<'EOF'
set -e
# A record
dig @127.0.0.1 app.lab.local +short
dig @127.0.0.1 grafana.lab.local +short

# SRV record (service discovery)
dig @127.0.0.1 _kube._tcp.lab.local SRV +short

# Reverse
dig @127.0.0.1 -x $(hostname -I | awk '{print $1}') +short

# Pastikan query luar tetap forward
dig @127.0.0.1 example.com +short
EOF
```

Output `dig SRV` harus berupa `10 10 6443 api.lab.local.` — itulah SRV record (priority weight port target).

### 4. Jadikan VM Sebagai Resolver untuk Mac

Agar Mac bisa pakai dnsmasq VM sebagai DNS, ada dua opsi.

**Opsi A — Resolver per-domain (paling aman, macOS):** hanya nama `.lab.local` yang ditanyakan ke VM, sisanya tetap ke DNS Mac.

```bash
VM_IP=$(ssh lab01 'hostname -I | awk "{print \$1}"')
sudo mkdir -p /etc/resolver
echo "nameserver $VM_IP" | sudo tee /etc/resolver/lab.local

# Uji dari Mac:
dig app.lab.local +short
dig _kube._tcp.lab.local SRV +short
# Pastikan domain lain tetap normal:
dig example.com +short
```

**Opsi B — Hapus dulu `/etc/hosts` manual dari LAB-01** agar benar-benar lewat DNS:

```bash
sudo sed -i '' '/lab.local/d' /etc/hosts        # macOS sed
dig app.lab.local +short                          # harus tetap resolve via dnsmasq
```

### 5. Integrasi dengan Caddy (dari LAB-01)

Jika LAB-01 modul 0.2 sudah jalan, sekarang `app.lab.local` dan `grafana.lab.local` resolve via DNS (bukan `/etc/hosts`). Caddy tetap melayani karena nama sama:

```bash
curl -k https://app.lab.local/                    # tetap berfungsi
# Beda: sekarang resolusi lewat dnsmasq, terlihat di log dnsmasq:
ssh lab01 'sudo journalctl -u dnsmasq -n 10 --no-pager | grep lab.local'
```

### 6. Eksperimen TTL & Perubahan IP

```bash
ssh lab01 <<'EOF'
set -e
# Ubah IP app.lab.local, turunkan TTL
sudo sed -i 's/.*app.lab.local.*/127.0.0.1 app.lab.local/' /etc/hosts
sudo systemctl restart dnsmasq
EOF

dig app.lab.local +stats | grep -E "ANSWER SECTION|query time"
# Lihat TTL pendek (local-cache-ttl dnsmasq default), perubahan cepat menyebar

# Kembalikan
VM_IP=$(ssh lab01 'hostname -I | awk "{print \$1}"')
ssh lab01 "sudo sed -i 's/127.0.0.1 app.lab.local/$VM_IP app.lab.local/' /etc/hosts && sudo systemctl restart dnsmasq"
```

Ini mengajarkan kenapa **TTL pendek** penting saat akan migrasi IP service — perubahan cepat terlihat di seluruh jaringan.

### 7. Simpan ke Repo

```bash
ssh lab01 'cat /etc/dnsmasq.d/lab.conf' > lab/dnsmasq-lab.conf
ssh lab01 'grep lab.local /etc/hosts' > lab/hosts-lab.txt
git add lab/dnsmasq-lab.conf lab/hosts-lab.txt
git commit -m "m0.2: dnsmasq DNS lokal + SRV record"
```

## Acceptance Criteria

- [ ] dnsmasq running sebagai service (`systemctl status dnsmasq`)
- [ ] `dig @127.0.0.1 app.lab.local` balas IP VM
- [ ] `dig @127.0.0.1 _kube._tcp.lab.local SRV` balas `... 6443 api.lab.local.`
- [ ] Mac bisa resolve `*.lab.local` via `/etc/resolver/lab.local` (Opsi A)
- [ ] Query domain publik (`example.com`) tetap normal dari Mac
- [ ] Integrasi Caddy tetap berfungsi lewat DNS (bukan `/etc/hosts`)
- [ ] Config tersimpan di repo GitLab

## Troubleshooting

| Gejala | Solusi |
|---|---|
| `dnsmasq: failed to create listening socket` port 53 bentrok | `ss -tulnp \| grep :53`; systemd-resolved bentrok → `sudo systemctl disable systemd-resolved` atau ubah port |
| `dig` balas kosong | Cek `journalctl -u dnsmasq`; pastikan `local=/lab.local/` & `/etc/hosts` punya entry |
| Mac tidak resolve `.lab.local` | Verifikasi `/etc/resolver/lab.local` ada & berisi IP VM; `scutil --dns \| grep lab.local` |
| Query publik lambat | dnsmasq forward ke `1.1.1.1`; cek `server=` di config |
| Perubahan `/etc/hosts` tidak terlihat | `sudo systemctl restart dnsmasq` (dnsmasq baca hosts saat start) |
| SRV record tidak muncul | Pastikan `srv-host=` sintaks benar: `srv-host=<name>,<target>,<port>,<priority>,<weight>` |

## Catatan SRE
- dnsmasq di sini hanya **satu VM**. Di production on-prem, DNS internal biasanya **redundan** (≥ 2 server) + di-monitor. DNS mati = seluruh layanan "tiba-tiba tidak ada".
- Ini model mental untuk **CoreDNS** di Kubernetes: ia yang menyelesaikan `servicename.namespace.svc.cluster.local` ke ClusterIP. SRV record dipakai untuk `port` service.

## Lanjut
[LAB-03 — Trace Koneksi](LAB-03-trace-koneksi.md)