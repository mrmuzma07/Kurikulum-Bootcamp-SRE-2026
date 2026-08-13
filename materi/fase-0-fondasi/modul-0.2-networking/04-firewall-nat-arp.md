# 04 — Firewall, NAT, Port Forwarding & ARP/Routing

> Membatasi & meneruskan trafik, serta kenapa ARP adalah kunci MetalLB L2.

## Tujuan
- Bisa mengkonfigurasi firewall dasar dengan `ufw`
- Paham NAT & port forwarding (DNAT/SNAT/MASQUERADE)
- Mengerti ARP & tabel routing — fondasi MetalLB L2 mode
- Bisa men-trace "kenapa trafik tidak sampai" secara sistematis

## 1. Firewall — Model Mental

Firewall = aturan yang menentukan paket mana boleh masuk/keluar/lintas. Tiga tahap di kernel Linux (netfilter):

```
 paket masuk ──► PREROUTING (DNAT) ──► routing ──┬─► INPUT (ke proses lokal)
                                                 │
                                                 └─► FORWARD (diteruskan) ──► POSTROUTING (SNAT) ──► keluar

 paket keluar dari proses ──► OUTPUT ──► POSTROUTING ──► keluar
```

Dua interface ke konfigurasi firewall:
- **nftables** — backend modern, sintaks `nft`
- **ufw** — front-end sederhana untuk iptables/nft (Uncomplicated Firewall). Pakai ini untuk mulai.

## 2. `ufw` — Firewall Dasar

```bash
sudo ufw enable                     # aktifkan (hati-hati via SSH — lihat catatan)
sudo ufw default deny incoming      # tolak masuk default
sudo ufw default allow outgoing     # izinkan keluar default

sudo ufw allow 22/tcp               # SSH (WAJIB sebelum deny, kalau remote)
sudo ufw allow 80/tcp               # HTTP
sudo ufw allow 443/tcp              # HTTPS
sudo ufw allow from 192.168.97.0/24 to any port 6443 proto tcp   # khusus subnet (K8s API)

sudo ufw status verbose
sudo ufw status numbered            # lihat nomor aturan
sudo ufw delete 3                   # hapus aturan #3
sudo ufw disable                    # matikan
```

**Aturan emas saat konfigurasi firewall via SSH:**
1. **Izinkan port 22 SEBELUM** `ufw default deny incoming`.
2. Atau jalankan dengan **timeout**: `sudo ufw enable && sleep 60 && sudo ufw disable` agar kalau terkunci, firewall mati sendiri dalam 60 detik.
3. Uji dari terminal SSH lain sebelum menutup sesi aktif.

```bash
# Verifikasi aturan benar-benar dipakai kernel:
sudo ufw status verbose
sudo iptables -L -n -v              # lihat rules & hitungan paket
```

## 3. NAT (Network Address Translation)

NAT = mengubah IP/port sumber atau tujuan saat paket lewat. Dua bentuk utama:

**SNAT / MASQUERADE** — ganti sumber (outbound):
```
Pod 10.244.0.5 ─► [node: MASQUERADE] ─► internet 1.2.3.4
                   (ganti src: 10.244.0.5 → IP node)
```
Inilah kenapa Pod bisa keluar internet meski pakai IP internal. Di k3s/kubenet ini otomatis.

**DNAT (Port Forwarding)** — ganti tujuan (inbound):
```
internet ─► router:80 ─► [DNAT] ─► 192.168.1.10:8080
            (ganti dst: router:80 → 192.168.1.10:8080)
```
Cara satu IP publik melayani banyak service di port berbeda. Di lab OrbStack, port forwarding dari Mac ke VM contohnya.

```bash
# Lihat tabel NAT:
sudo iptables -t nat -L -n -v
sudo nft list ruleset | grep -A5 masquerade
```

## 4. Port Forwarding di OrbStack

Saat Anda `ssh lab01` atau akses service di VM OrbStack, OrbStack melakukan port forwarding antara Mac dan VM. Pola yang sama berlaku di production on-prem saat expose service internal ke luar.

```bash
# SSH local port forwarding (dipelajari di modul 0.1):
ssh -L 8080:localhost:80 lab01
# Mac:8080 ─► tunnel SSH ─► lab01:80
```

## 5. ARP — IP Menemukan MAC

ARP (Address Resolution Protocol) = jembatan antara dunia IP (layer 3) dan MAC address (layer 2). Komunikasi nyata di jaringan fisik pakai **MAC**, bukan IP.

```
Host A butuh kirim ke 192.168.1.10
  │
  ├─ cek ARP cache: ada 192.168.1.10?
  │   └─ ada → kirim ke MAC itu langsung
  │   └─ tidak → kirim broadcast: "siapa yang punya 192.168.1.10? balas ke aku"
  │                              ▲
  │                              │ ARP request (broadcast)
  │              Host B (192.168.1.10) ─► "aku, MAC aa:bb:cc:dd:ee:ff"
  │                                          ▲ ARP reply (unicast)
  └─ cache MAC, kirim paket
```

```bash
# Lihat & manipulasi tabel ARP:
ip neigh show                     # tabel ARP (cache)
arping -I eth0 192.168.1.10       # kirim ARP manual, cek siapa pemilik IP
ip neigh flush dev eth0           # hapus cache ARP (saat IP pindah host)
```

**ARP cache berumur ~1 menit.** Saat IP berpindah host, cache lama membuat trafik nyasar ke MAC lama. Inilah sumber masalah klasik.

## 6. Kenapa ARP = Jantung MetalLB L2

**MetalLB L2 mode** bekerja **dengan ARP**. Ini mekanismenya:

```
Service type=LoadBalancer dapat IP 192.168.1.240 (dari pool MetalLB)
        │
        ▼
MetalLB memilih SATU node (leader) untuk "mengklaim" IP itu
        │
        ▼
Node leader mengirim Gratuitous ARP ke jaringan:
   "Halo semua, 192.168.1.240 sekarang ada di MAC-ku!"
        │
        ▼
Semua host di subnet update cache ARP → trafik ke 192.168.1.240
dialihkan ke node leader → node forward ke Pod
```

**Gratuitous ARP** = ARP broadcast yang tidak menjawab permintaan, tapi **mengumumkan**. Digunakan untuk "merebut" kepemilikan IP.

**Implikasi penting:**
1. L2 mode hanya efektif dalam **satu subnet/L2 domain** (ARP broadcast tidak lewat router).
2. Saat node leader mati, node lain kirim gratuitous ARP baru. Tapi **cache ARP lama di host lain belum kedaluarsa** → ada window downtime sampai cache expire atau di-flush.
3. **ARP conflict** terjadi kalau IP pool MetalLB bentrok dengan IP lain → dua host mengklaim IP sama.

Inilah kenapa topik ARP ini **wajib sebelum** Modul 2.3 (MetalLB). Tanpa paham ARP, troubleshooting "IP LoadBalancer tidak responding" hanya tebakan.

## 7. Routing — Tabel yang Menentukan Arah

```bash
ip route show                      # tabel routing
ip route get 8.8.8.8               # lewat mana untuk ke IP ini?
ip route add 10.50.0.0/16 via 192.168.1.1   # tambah rute statis
```

```
default via 192.168.1.1 dev eth0          # gateway untuk "sisanya" (internet)
192.168.1.0/24 dev eth0 proto kernel       # subnet lokal, langsung
10.50.0.0/16 via 192.168.1.254 dev eth0    # rute ke subnet jauh via router
```

**Aturan routing:** paket cocok dengan entri paling spesifik (`10.50.0.0/16` menang atas `default`). Saat MetalLB/Pod network tidak reachable, cek `ip route` di node — sering kali rute ke subnet Pod hilang.

## 8. Urutan Troubleshooting "Trafik Tidak Sampai"

Sistematis, dari bawah ke atas:

```
1. Fisik/link   → ip link show, ip addr             (interface up? ada IP?)
2. ARP/L2       → ip neigh, arping target           (satu subnet? cache ok?)
3. Routing      → ip route get <target>             (lewat interface benar?)
4. Firewall     → ufw status, iptables -L -n -v     (aturan blokir?)
5. Port/listen  → ss -tlnp                          (service dengar? di 0.0.0.0?)
6. DNS          → dig name                          (nama resolve benar?)
7. App          → curl -v, log aplikasi             (logik error?)
```

Lompat satu punya langkah = bisa salah diagnosa. Ini akan dipraktikkan di [LAB-03](lab/LAB-03-trace-koneksi.md).

## Latihan Cepat (15 menit)

```bash
# 1. Lihat interface & IP
ip -br addr show

# 2. Lihat tabel routing
ip route show
ip route get 1.1.1.1

# 3. Lihat & flush ARP cache
ip neigh show
ip neigh flush dev eth0       # lalu akses sesuatu, lihat cache terbangun lagi

# 4. Arping sebuah IP (apakah ada yang menjawab?)
sudo arping -c 3 <IP-gateway-VM>

# 5. Konfigurasi ufw (jika via SSH, izinkan 22 DULU)
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw allow 80/tcp
sudo ufw status verbose
# (uji dari terminal SSH lain, lalu sudo ufw disable bila hanya eksplorasi)

# 6. Lihat tabel NAT (jika ada)
sudo iptables -t nat -L -n -v 2>/dev/null || echo "tidak ada aturan NAT"
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Aturan firewall sederhana | `ufw` |
| Lihat rules kernel | `iptables -L -n -v`, `nft list ruleset` |
| Lihat NAT | `iptables -t nat -L -n -v` |
| Tabel ARP | `ip neigh show` |
| Cek pemilik IP (L2) | `arping` |
| Hapus cache ARP | `ip neigh flush dev <if>` |
| Tabel routing | `ip route show`, `ip route get <IP>` |

## Cek Pemahaman

1. Kenapa harus `ufw allow 22/tcp` **sebelum** `ufw default deny incoming` saat konfigurasi via SSH?
2. Beda SNAT/MASQUERADE vs DNAT — masing-masing untuk arah mana?
3. Saat IP MetalLB berpindah node, kenapa ada downtime singkat meski Pod sudah sehat?
4. Urutan 7 langkah troubleshooting "trafik tidak sampai" — kenapa cek ARP sebelum routing?
5. Apa itu gratuitous ARP dan kenapa MetalLB memakainya?