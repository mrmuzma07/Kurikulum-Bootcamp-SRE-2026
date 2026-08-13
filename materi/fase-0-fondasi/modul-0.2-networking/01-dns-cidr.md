# 01 — DNS & CIDR

> Cara nama jadi IP, dan cara membagi jaringan dengan angka di belakang garis miring.

## Tujuan
- Paham record DNS penting dan alur resolusi (browser → resolver → authoritative)
- Bisa menghitung subnet CIDR & memilih range IP yang tidak bentrok
- Mengerti kenapa pemilihan subnet penting sebelum pasang MetalLB

## 1. Kenapa DNS Penting untuk SRE

Di production on-prem, **semua service bicara pakai nama, bukan IP**. IP berubah, nama stabil. Saat troubleshooting, pertanyaan pertama selalu: "nama ini resolve ke IP mana?" — bukan "berapa IP server-nya?".

```bash
# Tool dasar (dipelajari di modul 0.1):
dig app.example.com +short
dig app.example.com A
dig -x 192.168.1.10        # reverse: IP → nama
```

## 2. Record DNS yang Wajib Dikenal

| Record | Fungsi | Contoh |
|---|---|---|
| `A` | nama → IPv4 | `app.example.com. A 192.0.2.10` |
| `AAAA` | nama → IPv6 | `app.example.com. AAAA 2001:db8::10` |
| `CNAME` | alias ke nama lain | `www.example.com. CNAME app.example.com.` |
| `SRV` | nama + port (service discovery) | `_kube._tcp SRV 10 10 6443 api.lab.local.` |
| `MX` | mail server | `example.com. MX 10 mail.example.com.` |
| `TXT` | metadata (SPF, DKIM, verifikasi) | `example.com. TXT "v=spf1 -all"` |
| `PTR` | reverse lookup (IP → nama) | `10.1.168.192.in-addr.arpa. PTR app.example.com.` |

**SRV** relevan untuk Kubernetes: service discovery internal pakai DNS SRV. `CNAME` penting saat satu service punya banyak nama (mis. `grafana`, `dashboard`, `monitor` semua nunjuk ke host yang sama).

## 3. Alur Resolusi DNS

```
Browser minta app.example.com
        │
        ▼
  [Resolver lokal] ── cache? ── ya ──► jawab langsung
        │ tidak
        ▼
  [Root nameserver] ── "tanya .com"
        │
        ▼
  [.com TLD] ── "tanya example.com authoritative"
        │
        ▼
  [Authoritative example.com] ── "A record = 192.0.2.10"
        │
        ▼
  Resolver cache (selama TTL), balas ke browser
```

```bash
# Lihat alur ini secara langsung:
dig app.example.com +trace          # dari root ke bawah
dig app.example.com +stats          # lihat query time & TTL
dig app.example.com | grep -A2 "ANSWER SECTION"
```

**TTL (Time To Live)** = berapa lama resolver boleh cache jawaban. TTL tinggi = cepat tapi lambat saat IP berubah. Saat migrasi server, **turunkan TTL jauh hari sebelumnya** agar perubahan menyebar cepat.

## 4. `/etc/hosts` — DNS Minimalis

Tanpa DNS server, nama bisa dipetakan manual di `/etc/hosts`:

```bash
# /etc/hosts — dibaca SEBELUM query ke resolver
192.168.97.10   app.lab.local
192.168.97.11   grafana.lab.local
192.168.97.12   api.lab.local
```

```bash
getent hosts app.lab.local       # cek resolusi (termasuk /etc/hosts)
```

`/etc/hosts` cocok untuk lab & satu-dua host. Untuk puluhan service, pakai dnsmasq (lihat [LAB-02](lab/LAB-02-dns-lokal.md)).

## 5. CIDR & Subnetting — Dasar

CIDR (Classless Inter-Domain Routing) = cara nulis rentang IP dengan garis miring: `192.168.1.0/24`.

Angka setelah `/` = jumlah bit network (dari total 32 bit IPv4). Sisanya = bit host.

| CIDR | Bit host | Jumlah IP | Mask | Range contoh |
|---|---|---|---|---|
| `/32` | 0 | 1 | `255.255.255.255` | satu host |
| `/30` | 2 | 4 (2 usable) | `255.255.255.252` | link point-to-point |
| `/24` | 8 | 256 (254 usable) | `255.255.255.0` | satu subnet kantor |
| `/16` | 16 | 65 536 | `255.255.0.0` | satu blok besar |
| `/8`  | 24 | 16 juta | `255.0.0.0` | kelas A |

**Range privat (RFC 1918)** — IP untuk jaringan internal, tidak routing di internet:
- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

## 6. Menghitung Subnet Manual

```bash
# IP utils bantu:
ipcalc 192.168.1.0/24           # tampilkan network, broadcast, host range
ipcalc 192.168.1.0/26           # /26 → 64 IP, 62 usable
```

Logika `/26` dari `192.168.1.0`:
- 32 - 26 = 6 bit host → 2^6 = 64 IP
- Network: `192.168.1.0`, Broadcast: `192.168.1.63`
- Usable: `192.168.1.1` – `192.168.1.62`

```
/24 (256)  ─┬─ /25 (128) ─┬─ /26 (64)
            │              └─ /26 (64)
            └─ /25 (128) ─┬─ /26 (64)
                           └─ /26 (64)
```

Satu `/24` bisa dipecah jadi empat `/26`. Ini esensi subnetting: membagi satu blok besar jadi banyak blok kecil tanpa tumpang tindih.

## 7. Kenapa Ini Penting untuk MetalLB (Modul 2.3)

MetalLB butuh **pool IP** untuk dibagikan ke Service `type=LoadBalancer`. Aturannya:

1. Pilih subnet yang **tidak dipakai DHCP** server (jangan ambil IP yang bisa di-assign ke laptop/server lain).
2. Pisahkan dari range static IP server fisik.
3. Cukup besar untuk service yang akan di-expose.

```yaml
# Contoh (dipakai nanti di Modul 2.3):
# Jika DHCP = 192.168.1.100-192.168.1.200
# Dan server fisik = 192.168.1.10-192.168.1.20
# Maka MetalLB pool AMAN: 192.168.1.240-192.168.1.250
addresses:
  - 192.168.1.240-192.168.1.250
```

**Bentrok IP = nightmare debugging.** Service dapat IP yang ternyata juga dipakai laptop seseorang → ARP conflict, traffic nyasar. Inilah kenapa DNS & CIDR wajib dipahami sebelum Kubernetes on-prem.

## Latihan Cepat (10 menit)

```bash
# 1. Cek record domain nyata
dig google.com A +short
dig google.com AAAA +short
dig google.com MX +short

# 2. Trace alur resolusi
dig example.com +trace | head -30

# 3. Tambahkan entry lokal
echo "192.168.97.99  latihan.lab.local" | sudo tee -a /etc/hosts
getent hosts latihan.lab.local
ping -c 1 latihan.lab.local

# 4. Hitung subnet
ipcalc 10.10.10.0/23        # berapa IP? berapa usable?
ipcalc 172.16.5.0/28        # range usable?

# 5. Bersihkan entry hosts (jangan tinggalkan sampah)
sudo sed -i '/latihan.lab.local/d' /etc/hosts
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Cari IP dari nama | `dig name A +short` |
| Cari nama dari IP | `dig -x IP +short` |
| Trace alur resolusi | `dig name +trace` |
| Petakan nama manual (lab) | `/etc/hosts` + `getent hosts` |
| Hitung subnet | `ipcalc CIDR` |
| Record service discovery | `SRV` |

## Cek Pemahaman

1. Apa beda `A` vs `CNAME`, dan kapan pakai `CNAME`?
2. Kenapa TTL perlu diturunkan sebelum migrasi server?
3. Berapa usable host di `10.0.0.0/22`?
4. Saat memilih pool MetalLB, apa dua range yang harus dihindari?