# 05 — Networking CLI

> Periksa, telusuri, dan diagnosa jaringan dari terminal.

## Tujuan
- Cek konektivitas dasar (ping, curl, dns)
- Lihat port yang listening dan koneksi aktif
- Routing, ARP, tabel IP
- Artefak troubleshooting umum

## 1. `ping` & `traceroute`

```bash
ping -c 4 1.1.1.1                      # 4 kali, berhenti
ping -c 4 -W 2 1.1.1.1                  # timeout 2 detik
ping -c 4 -i 0.5 1.1.1.1                # interval 0.5s
ping 6.google.com                       # IPv6
```

**Ingat:** `ping` banyak diblok saat egress — kemampuan keluar ≠ bisa ping.

```bash
traceroute google.com                   # jalur lompatan
mtr google.com                          # traceroute interaktif (install)
```

## 2. DNS — `dig`

```bash
dig example.com                         # A record
dig example.com +short                  # jawaban saja
dig @1.1.1.1 example.com                # pakai resolver tertentu
dig example.com A +trace                # trace dari root
dig example.com MX                      # mail
dig example.com TXT                     # TXT (SPF, DKIM, dll)
dig -x 1.1.1.1                          # reverse lookup
dig example.com +stats                  # statistik query
```

```bash
# TTL: berapa lama cache boleh simpan jawaban
dig example.com | grep -A1 "ANSWER SECTION"
```

**Catatan:** `dig` wajib dipasang karena `nslookup` minim fitur.

Konsep record:
- `A` — nama → IPv4
- `AAAA` — nama → IPv6
- `CNAME` — alias
- `MX` — mail exchanger
- `TXT` — teks bebas (SPF, DKIM, domain verification)
- `NS` — nameserver
- `SRV` — service location (Lync, Minecraft, etc.)
- `PTR` — IP → nama (reverse)
- `SOA` — start of authority

## 3. `curl` — HTTP Client

```bash
curl https://example.com                # body
curl -I https://example.com             # header saja
curl -v https://example.com             # verbose, lihat SNI, TLS
curl -o file.zip https://example.com/x.zip
curl -L https://example.com             # follow redirect
curl -X POST -d 'a=1' https://api.example.com
curl -H "Authorization: Bearer $TOK" https://api.example.com
curl -H "Content-Type: application/json" -d '{"a":1}' https://api.example.com
curl -s https://api.example.com | jq .
curl -sk https://internal                 # -k: skip cert verify (hati-hati)
curl -fSL https://example.com -o app.bin  # 404 jadi exit non-zero
curl --max-time 10 -sSL https://example.com
```

**Wget vs curl:** `curl` lebih serbaguna (libcurl di mana-mana), `wget` lebih untuk download recursive.

## 4. `ss` / `netstat` — Socket & Port

```bash
ss -ltn                          # listening TCP, numeric
ss -ltnp                         # + process (butuh root)
ss -tan                          # semua TCP
ss -uan                          # UDP
ss -s                            # ringkasan
ss -o state established '( dport = :443 )'  # koneksi yg sedang ke 443
```

**netstat udah bangkotan, default gunakan `ss`.**

## 5. `ip` — Peta Jaringan Sendiri

```bash
ip addr                          # ip address (ganti ifconfig)
ip link                          # device layer 2
ip route                         # routing table
ip neigh                         # ARP table
ip -s link show eth0             # statistik per interface
ip addr add 10.0.0.50/24 dev eth0 # tambah IP
```

Untuk cek fisik:
```bash
ethtool eth0                     # kecepatan, link, driver
```

## 6. `tcpdump` — Intip Paket

```bash
sudo tcpdump -i any port 80
sudo tcpdump -i any -n 'tcp port 443 and host 1.1.1.1'
sudo tcpdump -i any -w capture.pcap port 22
tcpdump -r capture.pcap          # baca dari file
```

**Penting:** produksi on-prem, sering satu-satunya cara memikirkan apa yang lewat.

## 7. `/etc/hosts`, `/etc/resolv.conf`, `/etc/nsswitch.conf`

```bash
cat /etc/hosts                    # override DNS lokal
cat /etc/resolv.conf              # nameserver default
getent hosts example.com          # lihat resolution final
```

**Urutan pencarian nama:** `/etc/nsswitch.conf` (file `hosts:` di sini).

## 8. Firewall — `ufw` / `nftables`

```bash
sudo ufw status verbose
sudo ufw allow 22/tcp
sudo ufw allow from 10.0.0.0/8 to any port 3306
sudo ufw delete allow 22/tcp
sudo ufw enable

# nftables (lebih modern)
sudo nft list ruleset
```

## 9. Akses HTTP & TLS

```bash
# Ambil cert publik
openssl s_client -connect example.com:443 < /dev/null
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Cek apakah cert ekspired
date                              # bandingkan
```

## 10. Kisah "Service Tidak Bisa Diakses"

Urutan cek saat ada laporan "app tidak bisa diakses":

1. `curl -v http://localhost:8080` di server (apakah service running?)
2. `ss -ltnp | grep 8080` (portnya listening?)
3. `curl -v http://server:8080` di mac (apakah bisa reach?)
4. `ping` dan `traceroute` (network?)
5. `dig app.example.com` (DNS resolving?)
6. `sudo tcpdump -i any port 8080` (apakah packet lewat?)
7. `sudo ufw status` (apakah firewall blok?)
8. Logs di service: `journalctl -u app.service -n 100`

## Latihan Cepat

```bash
# 1. Cek waktu & TLS expiration google.com
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -dates

# 2. Cari IP publik saya
curl -s https://api.ipify.org; echo

# 3. Lihat port apa yang listening
ss -ltn                              # ISSUREA _SEMUA 0.0.0.0 = berbahaya

# 4. DNS trace
dig +trace microsoft.com | head -30

# 5. Test throughput
curl -o /dev/null -w "%{http_code} time=%{time_total}s\n" https://example.com

# 6. Cek MTU
ip -d link show eth0 | grep mtu
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Cek koneksi | `ping`, `mtr` |
| DNS | `dig`, `host` |
| HTTP | `curl` |
| Lihat port | `ss -ltn` |
| Interface & routing | `ip addr`, `ip route` |
| Lihat paket | `tcpdump` |
| Firewall | `ufw`, `nft` |
| TLS | `openssl s_client` |

## Cek Pemahaman

1. Beda `A` vs `AAAA` vs `CNAME`?
2. Kenapa `-v` di `curl` penting saat troubleshooting?
3. Beda `0.0.0.0:443` vs `127.0.0.1:443` di `ss -ltn`?
4. Urutan cek yang benar untuk "service tidak reachable"?
