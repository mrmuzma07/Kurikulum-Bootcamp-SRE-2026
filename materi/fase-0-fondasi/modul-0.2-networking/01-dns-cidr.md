# 01 — DNS & CIDR

> **Ringkasan:** Memahami bagaimana **nama** (seperti `app.example.com`) diterjemahkan menjadi **alamat IP** lewat DNS, bagaimana jaringan dibagi-bagi menggunakan **CIDR/subnet** (angka setelah garis miring, misal `/24`), dan mengapa dua hal ini wajib dikuasai sebelum memasang MetalLB di Kubernetes on-prem.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Menjelaskan apa itu IP address, DNS, dan mengapa production memakai **nama, bukan IP**.
- Membedakan record DNS penting (`A`, `AAAA`, `CNAME`, `SRV`, `MX`, `TXT`, `PTR`) dan kapan memakainya.
- Menjelaskan alur resolusi DNS (resolver → root → TLD → authoritative) serta peran **cache dan TTL**.
- Memakai `/etc/hosts` untuk pemetaan nama manual dan mengetahui urutan pembacaannya.
- Membaca notasi **CIDR** (`/24`, `/26`, dll.) dan **menghitung** jumlah IP, range usable, network, dan broadcast.
- Memilih **pool IP yang aman** (tidak bentrok) untuk MetalLB. 

---

## 1. Dasar yang Perlu Dipahami Dulu: Apa Itu IP Address?

> 🏠 **Analogi Alamat Rumah:** Agar surat sampai, rumah perlu alamat unik. Di jaringan, setiap perangkat (server, laptop, VM, pod) perlu **IP address** agar paket data tahu harus dikirim ke mana.

**IPv4** ditulis sebagai **empat angka (0–255) dipisahkan titik**, contoh `192.168.1.10`.
Setiap angka disebut **oktet** karena mewakili **8 bit**. Jadi satu IPv4 = 4 × 8 = **32 bit**.

```text
   192      .   168      .    1       .   10
11000000    . 10101000   . 00000001   . 00001010     ← bentuk biner (32 bit)
 oktet 1      oktet 2      oktet 3      oktet 4
```

Mengapa maksimal 255? Karena 8 bit hanya bisa menyimpan nilai 0–255 (2⁸ = 256 kemungkinan).

> 💡 **Dua hal penting:**
> - **IP bisa berubah** (server dipindah, di-rebuild, DHCP memberi alamat baru).
> - **Manusia tidak hafal angka**, tetapi hafal nama. Maka dibuatlah DNS.

---

## 2. Kenapa DNS Penting untuk SRE

> 📖 **Analogi Buku Telepon:** DNS (*Domain Name System*) adalah buku telepon internet. Kamu mencari nama **"Toko Budi"** lalu mendapat nomor teleponnya. Begitu juga: kamu mengetik `app.example.com` dan DNS memberi tahu IP-nya.

Di production on-prem, **semua service saling bicara memakai nama, bukan IP**. Alasannya:
1. **IP berubah, nama stabil.** Database dipindah ke server baru? Cukup ubah satu record DNS, semua aplikasi tetap jalan tanpa edit konfigurasi.
2. **Satu nama, banyak IP** (load balancing) atau **banyak nama, satu IP**.
3. **Mudah dibaca & diaudit** (`db-prod.internal` jauh lebih jelas daripada `10.20.30.41`).

Saat troubleshooting, pertanyaan pertama SRE hampir selalu: **"Nama ini di-resolve ke IP yang mana?"** — bukan "berapa IP server-nya?". Ada lelucon terkenal di dunia SRE: *"It's always DNS."* (hampir selalu DNS biang keroknya).

```bash
# Alat dasar (sudah dipelajari di Modul 0.1 topik 05):
dig app.example.com +short        # IP saja
dig app.example.com A             # tampilan lengkap record A
dig -x 192.168.1.10               # reverse: IP -> nama
```
*(Jika `dig` belum ada: `sudo apt install -y dnsutils`.)*

---

## 3. Record DNS yang Wajib Dikenal

DNS menyimpan data dalam bentuk **record** (baris catatan). Setiap record punya **tipe** yang menentukan jenis informasinya.

| Record | Fungsi | Contoh | Analogi |
|---|---|---|---|
| **`A`** | nama → alamat **IPv4** | `app.example.com. A 192.0.2.10` | "Toko Budi" → nomor telepon-nya |
| **`AAAA`** | nama → alamat **IPv6** | `app.example.com. AAAA 2001:db8::10` | Sama, tapi format nomor yang lebih panjang |
| **`CNAME`** | **alias** ke nama lain | `www.example.com. CNAME app.example.com.` | "Toko Budi cabang Barat" = alias "Toko Budi" |
| **`SRV`** | nama + **port** (service discovery) | `_kube._tcp SRV 10 10 6443 api.lab.local.` | Bukan hanya alamat gedung, tapi juga nomor pintunya |
| **`MX`** | server **email** | `example.com. MX 10 mail.example.com.` | Alamat kantor pos penerima surat |
| **`TXT`** | teks bebas (SPF, DKIM, verifikasi) | `example.com. TXT "v=spf1 -all"` | Papan catatan tempel |
| **`PTR`** | **reverse** lookup (IP → nama) | `10.1.168.192.in-addr.arpa. PTR app.example.com.` | Cari nama pemilik dari nomor telepon |

### Memahami Beberapa Record Lebih Dalam

**`A` vs `CNAME` — kapan pakai yang mana?**
- `A` menunjuk **langsung ke IP**.
- `CNAME` menunjuk **ke nama lain**, yang kemudian di-resolve lagi.
- Pakai `CNAME` saat satu service punya **banyak nama**: `grafana.lab.local`, `dashboard.lab.local`, dan `monitor.lab.local` semuanya `CNAME` ke `mon-host.lab.local`. Jika IP `mon-host` berubah, **cukup ubah satu record `A`** — ketiga alias ikut benar.
- ⚠️ `CNAME` **tidak boleh** dipasang di *apex/root domain* (`example.com` itu sendiri) dan tidak boleh berdampingan dengan record lain pada nama yang sama.

**`SRV` — relevan untuk Kubernetes:**
Record `A` hanya memberi IP. `SRV` memberi **host + port + prioritas + bobot**. Kubernetes memakai DNS (termasuk SRV) untuk *service discovery* internal, yaitu pod menemukan service lain lewat nama seperti `redis.default.svc.cluster.local`.

**Format penamaan DNS:** nama diakhiri **titik** (`app.example.com.`). Titik terakhir mewakili *root*. Dibaca **dari kanan ke kiri**: `com` (TLD) → `example` (domain) → `app` (subdomain/host).

```bash
# Coba satu per satu dan perhatikan hasilnya:
dig google.com A +short
dig google.com AAAA +short
dig google.com MX +short
dig google.com TXT +short
dig www.github.com CNAME +short
dig -x 1.1.1.1 +short             # PTR: 1.1.1.1 milik siapa?
```

---

## 4. Alur Resolusi DNS: Siapa Bertanya ke Siapa?

Saat kamu membuka `app.example.com`, ada serangkaian "tanya-jawab" di balik layar. Tidak ada satu server yang tahu semua nama di dunia, jadi DNS bersifat **hierarki** (berjenjang).

```text
Browser minta app.example.com
        │
        ▼
  [Resolver lokal / cache] ── sudah pernah ditanya? ── ya ──► jawab langsung (cepat)
        │ belum / sudah kadaluarsa
        ▼
  [Root nameserver (.)]       ── "Aku tak tahu, tanya server untuk '.com' ya"
        │
        ▼
  [TLD nameserver (.com)]     ── "Aku tak tahu, tanya authoritative 'example.com' ya"
        │
        ▼
  [Authoritative example.com] ── "app.example.com A = 192.0.2.10"  ← JAWABAN ASLI
        │
        ▼
  Resolver menyimpan jawaban di cache (selama TTL), lalu membalas ke browser
```

**Peran setiap pemain:**
| Pemain | Tugas | Analogi |
|---|---|---|
| **Stub resolver** (di OS kamu) | Menerima pertanyaan dari aplikasi, meneruskan ke resolver | Kamu yang bertanya ke operator telepon |
| **Recursive resolver** (mis. DNS router, `1.1.1.1`) | Melakukan seluruh perjalanan tanya-jawab atas namamu, lalu **menyimpan cache** | Operator yang menelepon ke mana-mana mencari jawabannya |
| **Root / TLD** | Menunjuk arah ke tingkat berikutnya | Resepsionis yang bilang "ke lantai sana" |
| **Authoritative** | **Pemilik sah data** domain tersebut | Pemilik toko yang menjawab langsung |

```bash
# Lihat alur ini secara langsung (dari root ke bawah):
dig example.com +trace

# Lihat waktu query dan TTL:
dig example.com | grep -E "Query time|ANSWER SECTION" -A1
```

### TTL (Time To Live) — Berapa Lama Jawaban Boleh Di-cache

Pada output `dig`, angka di sebelah nama adalah **TTL dalam detik**:
```text
example.com.    3412    IN    A    93.184.215.14
                ────
                TTL: boleh di-cache 3412 detik (~57 menit)
```

| TTL | Dampak |
|---|---|
| **Tinggi** (mis. 86400 = 1 hari) | Cepat, beban resolver ringan, **tetapi** perubahan IP lambat menyebar |
| **Rendah** (mis. 60 detik) | Perubahan cepat menyebar, **tetapi** lebih banyak query |

> 🚨 **Praktik SRE saat migrasi server:** **turunkan TTL jauh hari sebelum** hari-H (misal dari 1 hari ke 60 detik, lalu tunggu minimal selama TTL lama berlalu). Pada hari migrasi, ubah record, dan perubahan menyebar dalam hitungan menit. Setelah stabil, naikkan lagi TTL. Jika lupa menurunkan TTL, sebagian pengguna akan "tersesat" ke IP lama berjam-jam.

---

## 5. `/etc/hosts` — DNS Minimalis di Mesin Sendiri

Tanpa DNS server, kamu bisa memetakan nama ke IP **secara manual** lewat file **`/etc/hosts`**. Linux membacanya **sebelum** bertanya ke DNS resolver (urutan diatur di `/etc/nsswitch.conf`).

```bash
# /etc/hosts — format: IP  nama [alias...]
127.0.0.1       localhost
192.168.97.10   app.lab.local
192.168.97.11   grafana.lab.local
192.168.97.12   api.lab.local
```

```bash
getent hosts app.lab.local       # cek hasil resolusi FINAL (termasuk /etc/hosts)
```

> 💡 **Mengapa pakai `getent hosts`, bukan `dig`?**
> `dig` **hanya bertanya ke DNS server** dan **mengabaikan `/etc/hosts`**. Jadi jika kamu menambah entri di `/etc/hosts` lalu mengeceknya dengan `dig`, hasilnya kosong, padahal aplikasimu akan berhasil me-resolve. `getent hosts` meniru cara aplikasi sungguhan me-resolve nama.

**Kapan `/etc/hosts` cocok?** Lab dan satu-dua host. **Kelemahannya:** harus diedit di *setiap* mesin, mudah lupa dan tidak sinkron. Untuk puluhan service pakai DNS server (misalnya `dnsmasq`, lihat [LAB-02](lab/LAB-02-dns-lokal.md)).

> ⚠️ **Jebakan:** entri `/etc/hosts` yang lupa dihapus bisa membuat aplikasi tetap menunjuk IP lama padahal DNS sudah diperbarui. Saat nama "resolve ke IP yang aneh", **cek `/etc/hosts` dulu**.

---

## 6. CIDR & Subnetting — Membagi Jaringan

### Apa Itu Subnet dan Mengapa Dibagi?
> 🏢 **Analogi Gedung Perkantoran:** Satu gedung besar (jaringan) dibagi per lantai (subnet): lantai 1 untuk server, lantai 2 untuk staf, lantai 3 untuk tamu. Tujuannya: **pengaturan lebih rapi, keamanan terpisah (firewall antar-lantai), dan tidak saling mengganggu**.

### Notasi CIDR (`/angka`)
CIDR (*Classless Inter-Domain Routing*) menulis sebuah rentang IP dengan **garis miring + angka**: `192.168.1.0/24`.

> **Angka setelah `/` = jumlah bit pertama yang dikunci sebagai "bagian network".**
> Sisa bit (dari total 32) bebas dipakai sebagai **"bagian host"** (nomor perangkat).

```text
192.168.1.0/24
│            │
│            └─ 24 bit pertama = NETWORK (nama "lantai"), tetap sama untuk semua perangkat
└─ 32 - 24 = 8 bit sisa = HOST (nomor "ruangan"), berbeda untuk tiap perangkat

11000000.10101000.00000001 . 00000000
└────── 24 bit NETWORK ────┘  └ 8 bit HOST ┘
```

### Rumus Sakti (Hafalkan Ini!)

```text
Bit host      = 32 − (angka CIDR)
Jumlah total IP = 2 ^ (bit host)
IP usable      = Jumlah total IP − 2
```
**Mengapa dikurangi 2?** Dua alamat di setiap subnet **dicadangkan**:
- **Alamat pertama = Network address** (nama subnet itu sendiri, mis. `192.168.1.0`).
- **Alamat terakhir = Broadcast address** (alamat "teriak ke semua", mis. `192.168.1.255`).

Keduanya tidak boleh diberikan ke perangkat.

### Tabel CIDR Penting

| CIDR | Bit host | Total IP | Usable | Netmask | Kegunaan Umum |
|:---:|:---:|---:|---:|---|---|
| `/32` | 0 | 1 | 1 (khusus) | `255.255.255.255` | Satu host tunggal |
| `/30` | 2 | 4 | 2 | `255.255.255.252` | Link point-to-point (2 router) |
| `/28` | 4 | 16 | 14 | `255.255.255.240` | Jaringan kecil / pool kecil |
| `/26` | 6 | 64 | 62 | `255.255.255.192` | Segmen kecil |
| `/24` | 8 | 256 | 254 | `255.255.255.0` | Satu subnet kantor/lab standar |
| `/23` | 9 | 512 | 510 | `255.255.254.0` | Dua `/24` digabung |
| `/22` | 10 | 1.024 | 1.022 | `255.255.252.0` | Empat `/24` digabung |
| `/16` | 16 | 65.536 | 65.534 | `255.255.0.0` | Blok besar |
| `/8` | 24 | 16.777.216 | 16.777.214 | `255.0.0.0` | Blok sangat besar |

> 🧠 **Trik mengingat:** setiap **kurangi 1** angka CIDR → jumlah IP **dikali 2**. (`/24` = 256, `/23` = 512, `/22` = 1024.) Setiap **tambah 1** → **dibagi 2**.

### Netmask: Cara Lain Menulis Hal yang Sama
`/24` identik dengan netmask `255.255.255.0`. Dalam biner, netmask adalah **deretan angka 1 (network) diikuti 0 (host)**:
```text
/24 → 11111111.11111111.11111111.00000000 = 255.255.255.0
/26 → 11111111.11111111.11111111.11000000 = 255.255.255.192
```

### Range IP Privat (RFC 1918)
Tiga blok ini **khusus jaringan internal** dan **tidak di-route di internet publik**. Semua jaringan lab/on-prem kamu memakai salah satunya:

| Blok | Rentang | Umumnya Dipakai |
|---|---|---|
| `10.0.0.0/8` | `10.0.0.0` – `10.255.255.255` | Perusahaan besar, data center |
| `172.16.0.0/12` | `172.16.0.0` – `172.31.255.255` | Menengah (Docker default memakai `172.17.0.0/16`) |
| `192.168.0.0/16` | `192.168.0.0` – `192.168.255.255` | Rumah, kantor kecil, router Wi-Fi |

---

## 7. Menghitung Subnet Secara Manual (Langkah demi Langkah)

### Contoh: `192.168.1.0/26`

**Langkah 1 — Hitung bit host:** `32 − 26 = 6` bit host.
**Langkah 2 — Hitung total IP:** `2⁶ = 64` IP per subnet.
**Langkah 3 — Cari "ukuran loncatan" (block size):** `64`. Subnet dimulai kelipatan 64 pada oktet terakhir: `0, 64, 128, 192`.
**Langkah 4 — Tentukan range subnet pertama:**

```text
Network   : 192.168.1.0        (alamat pertama)
Usable    : 192.168.1.1  s.d.  192.168.1.62   (62 IP untuk perangkat)
Broadcast : 192.168.1.63       (alamat terakhir)
```
Subnet berikutnya dimulai di `192.168.1.64` (network), usable `.65`–`.126`, broadcast `.127`, dan seterusnya.

### Visualisasi: Membelah Satu Blok Menjadi Blok Kecil

```text
192.168.1.0/24  (256 IP)
├── 192.168.1.0/25    (128 IP)  .0   – .127
│   ├── 192.168.1.0/26   (64)   .0   – .63
│   └── 192.168.1.64/26  (64)   .64  – .127
└── 192.168.1.128/25  (128 IP)  .128 – .255
    ├── 192.168.1.128/26 (64)   .128 – .191
    └── 192.168.1.192/26 (64)   .192 – .255
```
Satu `/24` bisa dibelah menjadi **dua `/25`** atau **empat `/26`** **tanpa tumpang tindih**. Itulah inti subnetting: membagi satu blok besar menjadi blok-blok kecil yang tidak saling menimpa.

### Contoh Lain: `10.0.0.0/22` (soal klasik)
```text
Bit host = 32 − 22 = 10 → total 2^10 = 1024 IP → usable 1022
Range: 10.0.0.0 s.d. 10.0.3.255   (empat "kelompok /24": 10.0.0.x, 10.0.1.x, 10.0.2.x, 10.0.3.x)
```

### Menghitung dengan Alat

```bash
# Opsi 1: ipcalc (pasang dulu)
sudo apt install -y ipcalc
ipcalc 192.168.1.0/26

# Opsi 2: Python (selalu ada di Ubuntu; hasil konsisten)
python3 - << 'EOF'
import ipaddress
net = ipaddress.ip_network("192.168.1.0/26")
print("Network  :", net.network_address)
print("Broadcast:", net.broadcast_address)
print("Netmask  :", net.netmask)
print("Total IP :", net.num_addresses)
hosts = list(net.hosts())
print("Usable   :", hosts[0], "-", hosts[-1], f"({len(hosts)} host)")
EOF
```
*Verifikasi:* hasilnya harus cocok dengan hitungan manual: network `192.168.1.0`, broadcast `192.168.1.63`, usable `.1`–`.62`.

---

## 8. Kenapa Ini Penting untuk MetalLB (Modul 2.3)

Di cloud, `Service type=LoadBalancer` otomatis mendapat IP dari provider. Di **on-prem tidak ada** — **MetalLB** mengisi kekosongan ini dengan membagikan IP dari **pool** yang **kamu tentukan**. Memilih pool adalah keputusan jaringan yang bila salah akan sangat menyakitkan.

### Aturan Memilih Pool MetalLB
1. **Masih satu subnet** dengan node (mode L2 bekerja lewat ARP di jaringan lokal yang sama).
2. **Jangan tumpang tindih dengan range DHCP** (alamat yang otomatis diberikan router ke laptop/HP).
3. **Jangan tumpang tindih dengan IP static server/node** (alamat yang kamu atur manual).
4. **Jangan memakai network address, broadcast, atau IP gateway/router.**
5. Cukup besar untuk jumlah service yang akan di-expose, tetapi tidak berlebihan.

```text
Contoh peta subnet 192.168.1.0/24:

.1            → router/gateway           (JANGAN dipakai)
.10  – .20    → server & node static     (JANGAN dipakai)
.100 – .200   → range DHCP (laptop, HP)  (JANGAN dipakai)
.240 – .250   → POOL METALLB             ✅ AMAN: tidak bersinggungan dengan apa pun
.255          → broadcast                (JANGAN dipakai)
```

```yaml
# Contoh (dipakai nanti di Modul 2.3):
addresses:
  - 192.168.1.240-192.168.1.250
```

### 🔎 Khusus OrbStack di Apple Silicon
Subnet VM di OrbStack **belum tentu sama** dengan contoh di atas dan **dapat berbeda di tiap mesin**. **Jangan menyalin angka dari contoh.** Selalu cek jaringan nyata dulu:

```bash
ip -br addr             # IP & CIDR kartu jaringan VM
ip route                # subnet & default gateway
```
Pilih pool dari **subnet yang benar-benar dipakai VM kamu**, di ujung range yang tidak dipakai, dan pastikan tidak bentrok dengan IP host maupun router Wi-Fi.

> 💥 **Bentrok IP = mimpi buruk debugging.** Jika pool MetalLB mencakup IP yang juga dipakai laptop seseorang, terjadi **ARP conflict**: dua perangkat mengaku punya IP yang sama sehingga traffic nyasar secara acak dan hilang-timbul. Gejalanya sulit dilacak (kadang jalan, kadang tidak). Inilah mengapa DNS & CIDR **wajib dipahami sebelum Kubernetes on-prem**.

---

## 9. Pola Kegagalan (Failure Modes) & Diagnosa

| Gejala | Penyebab Umum | Cara Diagnosa & Solusi |
|---|---|---|
| `dig` berhasil tapi aplikasi gagal resolve (atau sebaliknya) | `dig` mengabaikan `/etc/hosts`; aplikasi memakainya | Bandingkan `dig nama +short` dengan `getent hosts nama`; periksa `/etc/hosts` dan `/etc/nsswitch.conf` |
| Setelah ganti IP di DNS, sebagian user masih ke IP lama | Cache + **TTL tinggi** | `dig nama` lihat TTL; tunggu TTL habis; saat migrasi turunkan TTL lebih awal |
| `Temporary failure in name resolution` / `Could not resolve host` | Resolver tidak terjangkau / `/etc/resolv.conf` salah | `cat /etc/resolv.conf`; `dig @1.1.1.1 nama +short` untuk membandingkan |
| Nama resolve ke IP aneh yang tidak dikenal | Entri `/etc/hosts` lama, atau DNS record salah | `getent hosts nama`; `grep nama /etc/hosts`; `dig nama +trace` |
| Subnet salah hitung: IP di luar range diperlakukan "bukan tetangga" | Salah menghitung `/mask` | Hitung ulang dengan `python3 ipaddress`; pastikan IP dan gateway berada dalam subnet yang sama |
| Service LoadBalancer MetalLB kadang jalan kadang tidak | **IP bentrok** (ARP conflict) dengan perangkat lain | Cek `ip neigh`; `arping`; pindahkan pool ke range yang benar-benar bebas |

---

## 10. Latihan Praktik Mandiri (15 Menit di Terminal VM OrbStack)

```bash
# 1. Periksa record DNS domain nyata
dig google.com A +short
dig google.com AAAA +short
dig google.com MX +short
dig google.com TXT +short

# 2. Telusuri alur resolusi dari root ke bawah (perhatikan jenjang: . → com. → example.com.)
dig example.com +trace | head -40

# 3. Amati TTL: jalankan dua kali berturut-turut, bandingkan angka TTL-nya (menurun karena cache)
dig example.com | grep -A1 "ANSWER SECTION"
sleep 5
dig example.com | grep -A1 "ANSWER SECTION"

# 4. Tambahkan entri lokal, lalu uji perbedaan dig vs getent
echo "192.168.97.99  latihan.lab.local" | sudo tee -a /etc/hosts
getent hosts latihan.lab.local           # AKAN menemukan
dig latihan.lab.local +short             # TIDAK menemukan (dig abaikan /etc/hosts)

# 5. Hitung subnet manual dulu di kertas, lalu verifikasi dengan Python
python3 - << 'EOF'
import ipaddress
for cidr in ["10.10.10.0/23", "172.16.5.0/28", "10.0.0.0/22", "192.168.1.0/26"]:
    n = ipaddress.ip_network(cidr)
    h = list(n.hosts())
    print(f"{cidr:16} total={n.num_addresses:<5} usable={len(h):<5} "
          f"range={h[0]} - {h[-1]}  broadcast={n.broadcast_address}")
EOF

# 6. Periksa subnet nyata VM kamu (bekal memilih pool MetalLB nanti)
ip -br addr
ip route

# 7. Bersihkan entri hosts (jangan tinggalkan sampah)
sudo sed -i '/latihan.lab.local/d' /etc/hosts
grep latihan /etc/hosts || echo "[INFO] entri latihan sudah bersih"
```

**Acceptance Criteria:**
- [ ] Bisa menunjukkan dari output `dig +trace` tiga jenjang: root, TLD, authoritative.
- [ ] Bisa membuktikan bahwa `getent hosts` melihat entri `/etc/hosts` sedangkan `dig` tidak.
- [ ] Hitungan manual `10.10.10.0/23` (512 IP, 510 usable) dan `172.16.5.0/28` (16 IP, 14 usable, `.1`–`.14`) cocok dengan hasil Python.
- [ ] Entri latihan di `/etc/hosts` sudah dibersihkan.
- [ ] Bisa menyebutkan subnet VM sendiri dan mengusulkan satu pool IP yang tidak bentrok.

---

## 11. Ringkasan (Cheat Sheet)

| Kebutuhan | Perintah / Konsep |
|---|---|
| Cari IP dari nama | `dig nama A +short` |
| Cari nama dari IP | `dig -x IP +short` |
| Telusuri alur resolusi | `dig nama +trace` |
| Resolusi seperti aplikasi nyata (termasuk `/etc/hosts`) | `getent hosts nama` |
| Pemetaan nama manual (lab) | `/etc/hosts` |
| Hitung subnet | `ipcalc CIDR` atau `python3 -c "import ipaddress..."` |
| Rumus bit host / IP | `32 − CIDR` → `2^bit` → `usable = total − 2` |
| Service discovery (nama + port) | record `SRV` |
| Saat migrasi server | Turunkan **TTL** jauh hari sebelumnya |
| Pool MetalLB | Subnet sama dengan node; hindari DHCP, IP static, gateway, broadcast |

---

## 12. Cek Pemahaman (Self-Check)

1. **Apa beda `A` dan `CNAME`, dan kapan sebaiknya memakai `CNAME`?**
2. **Mengapa TTL perlu diturunkan sebelum migrasi server?**
3. **Berapa jumlah usable host pada `10.0.0.0/22`? Berapa pada `172.16.5.0/28`, dan apa range usable-nya?**
4. **Saat memilih pool MetalLB, apa dua range utama yang harus dihindari, dan apa akibatnya jika dilanggar?**
5. **Kamu menambah `10.9.9.9 test.lab.local` di `/etc/hosts`. `dig test.lab.local +short` kosong, tetapi `curl http://test.lab.local` berhasil menjangkau server. Mengapa?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **`A` vs `CNAME`:** `A` memetakan nama **langsung ke IPv4**. `CNAME` adalah **alias yang menunjuk ke nama lain** (yang lalu di-resolve ke IP-nya). Pakai `CNAME` ketika satu service punya banyak nama (mis. `grafana`, `dashboard`, `monitor` → satu host) agar saat IP berubah **cukup mengubah satu record `A`**. Catatan: `CNAME` tidak boleh dipasang di apex domain.

2. **TTL sebelum migrasi:** Resolver menyimpan jawaban lama selama TTL. Jika TTL masih tinggi (misal 1 hari), setelah IP diganti sebagian pengguna tetap diarahkan ke IP lama hingga cache kedaluwarsa. Dengan menurunkan TTL (mis. 60 detik) **jauh hari sebelum** migrasi dan menunggu minimal selama TTL lama, perubahan saat hari-H menyebar dalam hitungan menit.

3. **Hitungan subnet:**
   - `10.0.0.0/22`: bit host = 10 → 2¹⁰ = 1.024 IP → **1.022 usable** (`10.0.0.1` – `10.0.3.254`).
   - `172.16.5.0/28`: bit host = 4 → 16 IP → **14 usable**, range `172.16.5.1` – `172.16.5.14` (network `.0`, broadcast `.15`).

4. **Dua range yang dihindari:** (a) **range DHCP** dan (b) **IP static server/node** (juga gateway, network, broadcast address). Jika dilanggar terjadi **IP conflict / ARP conflict**: dua perangkat memakai IP yang sama sehingga traffic nyasar, hilang-timbul, dan sangat sulit didiagnosa.

5. **Perbedaan `dig` dan resolver aplikasi:** `dig` hanya bertanya ke **DNS server** dan **tidak membaca `/etc/hosts`**. Sedangkan `curl` (lewat resolver sistem/`nsswitch.conf`) membaca `/etc/hosts` terlebih dahulu sehingga berhasil. Untuk meniru perilaku aplikasi gunakan `getent hosts test.lab.local`.

</details>

---

## 13. Langkah Selanjutnya

Kamu sudah paham **bagaimana nama menjadi IP** dan **bagaimana jaringan dibagi**. Berikutnya, di [02-tcp-udp-tls.md](02-tcp-udp-tls.md), kita pelajari **bagaimana data benar-benar dikirim** (TCP vs UDP) dan **bagaimana koneksi diamankan** (TLS).
