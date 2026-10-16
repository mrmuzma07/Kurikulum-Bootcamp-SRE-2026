# 05 — Networking CLI untuk SRE

> **Ringkasan:** Menguasai peralatan baris perintah (*Command Line*) untuk memeriksa identitas jaringan, menguji koneksi antar server (`ping`, `mtr`), melacak masalah DNS (`dig`), memeriksa port yang terbuka (`ss`), menguji layanan web/API (`curl`), menyadap paket (`tcpdump`), serta menjalankan alur diagnosa sistematis saat sebuah layanan tidak bisa diakses.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Memahami konsep dasar komunikasi jaringan (**IP Address, Port, DNS, Routing, dan Firewall**) dengan bahasa yang sederhana.
- Memeriksa alamat IP, status kartu jaringan, tabel routing, dan tabel ARP menggunakan perintah modern **`ip`**.
- Menguji keterjangkauan server menggunakan **`ping`** dan melacak jalur lompatan jaringan dengan **`traceroute` / `mtr`**.
- Mendiagnosa masalah nama domain (DNS) menggunakan **`dig`** serta memahami peran `/etc/hosts` dan `/etc/resolv.conf`.
- Menemukan port mana yang sedang aktif mendengarkan (*listening*) beserta nama prosesnya menggunakan **`ss -tulpn`**, serta memahami perbedaan krusial antara `127.0.0.1` dan `0.0.0.0`.
- Menguji layanan HTTP/HTTPS, REST API, dan memeriksa masa berlaku sertifikat TLS menggunakan **`curl`** dan **`openssl`**.
- Menjalankan urutan investigasi SRE secara sistematis saat mendapat laporan *"Aplikasi tidak bisa diakses!"*.

---

## 1. Mindset Dasar: Analogi Pengiriman Paket di Jaringan

Bagi pemula, istilah seperti *IP Address*, *Port*, *DNS*, *Routing*, dan *Firewall* sering terasa abstrak. Mari kita bayangkan server Linux kamu sebagai **sebuah Gedung Apartemen Besar**:

```text
  [ Tamu / User ]
        │
        │  1. Buka Buku Telepon (DNS: dig)
        │     "Gedung 'tokoku.com' itu alamat nomor berapa ya?" -> "10.0.0.50"
        ▼
  [ Jalan Raya (Routing & Ping) ]
        │
        ▼
  ┌─────────────────────────────────────────────────────────┐
  │  GEDUNG SERVER — Alamat IP: 10.0.0.50                   │
  │                                                         │
  │  👮 Satpam Pintu Masuk (Firewall: ufw)                  │
  │     │                                                   │
  │     ├──> 🚪 Pintu Kamar #22   (Port 22)   : Layanan SSH │
  │     ├──> 🚪 Pintu Kamar #80   (Port 80)   : Web HTTP    │
  │     ├──> 🚪 Pintu Kamar #443  (Port 443)  : Web HTTPS   │
  │     └──> 🚪 Pintu Kamar #5432 (Port 5432) : Database    │
  └─────────────────────────────────────────────────────────┘
```

1. **IP Address (`10.0.0.50`):** Adalah **Alamat Gedung**. Setiap komputer atau VM di jaringan wajib punya nomor alamat IP agar paket data tidak salah kirim ke gedung lain.
2. **DNS (*Domain Name System*):** Adalah **Buku Kontak Telepon**. Manusia sulit menghafal angka `142.250.4.100`, tapi mudah mengingat nama `google.com`. DNS bertugas menerjemahkan nama domain menjadi angka IP Address.
3. **Port (`1` s.d. `65535`):** Adalah **Nomor Pintu Kamar di dalam Gedung**. Di dalam 1 server yang sama, bisa berjalan banyak aplikasi sekaligus (SSH, Web Server, Database). Ketika paket sampai di gedung `10.0.0.50`, nomor **Port**-lah yang menentukan ke kamar aplikasi mana paket itu harus diserahkan!
4. **Routing (`ip route`):** Adalah **Peta Jalan Raya & Persimpangan** yang mengatur lewat jalur mana paket harus dikirim untuk mencapai gedung tujuan.
5. **Firewall (`ufw` / `nftables`):** Adalah **Satpam Gedung** yang berdiri di pagar depan. Meskipun di dalam gedung ada aplikasi yang membuka pintu kamar nomor `5432` (Database), jika Satpam Firewall melarang tamu luar masuk ke pintu `5432`, maka orang luar tetap tidak akan bisa masuk!

---

## 2. Mengecek Identitas Jaringan Sendiri (`ip`)

> ⚠️ **Catatan Sejarah:** Di tutorial Linux lama, kamu mungkin sering melihat perintah `ifconfig`, `route`, atau `arp`. Di distro Linux modern (seperti Ubuntu 22.04+), peralatan tua tersebut **sudah pensiun** dan digantikan oleh satu perintah serbaguna yang jauh lebih cepat dan akurat: **`ip`** (dari paket `iproute2`).

### A. Melihat Alamat IP & Kartu Jaringan (`ip addr`)
```bash
ip addr                     # Tampilkan rincian lengkap semua kartu jaringan (interface)
ip -br addr                 # TRIK SRE: Tampilkan secara ringkas, rapi, dan berwarna (-br = brief)!
```
*Contoh Output `ip -br addr`:*
```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             192.168.64.5/24 fd07:b51a:cc66::5/64
```

Mari kita pahami 2 kartu jaringan (*interface*) di atas:
1. **`lo` (*Loopback* — `127.0.0.1`):**
   - Ini adalah alamat **Diri Sendiri (*Localhost*)**. Setiap komputer punya alamat `127.0.0.1`. Jika aplikasi berbicara ke `127.0.0.1`, paket datanya tidak pernah keluar ke kabel jaringan, melainkan hanya berputar di dalam memori komputer itu sendiri.
2. **`eth0` (atau `enp0s1` — `192.168.64.5/24`):**
   - Ini adalah **Kartu Jaringan Utama** servermu yang terhubung ke jaringan luar (atau ke laptop Mac kamu di OrbStack). Status **`UP`** berarti kartu jaringan tersebut sedang menyala dan aktif.

### B. Melihat Peta Jalan Raya / Gerbang Keluar (`ip route`)
Bagaimana servermu tahu ke mana harus mengirim paket jika tujuannya ada di internet? Cek tabel rutenya:
```bash
ip route
```
*Contoh Output:*
```text
default via 192.168.64.1 dev eth0 proto dhcp src 192.168.64.5 metric 100
192.168.64.0/24 dev eth0 proto kernel scope link src 192.168.64.5
```
- Baris **`default via 192.168.64.1`** disebut **Default Gateway** (Pintu Gerbang Utama). Artinya: *"Jika server ini ingin menghubungi alamat IP yang tidak berada di satu ruangan lokal dengannya (misal ingin ke internet), serahkan paketnya ke router gerbang `192.168.64.1` lewat kartu jaringan `eth0`."*

### C. Melihat Daftar Tetangga Satu Ruangan / Tabel ARP (`ip neigh`)
```bash
ip neigh
```
*Contoh Output:*
```text
192.168.64.1 dev eth0 lladdr a2:b4:c6:d8:e0:01 REACHABLE
```
- Perintah ini menampilkan tabel **ARP (*Address Resolution Protocol*)**, yaitu daftar mesin tetangga yang berada di satu jaringan lokal beserta alamat fisik kartu jaringannya (**MAC Address** `lladdr`).
- *(Ingat perintah `ip neigh` ini baik-baik! Di **Fase 2 Kubernetes**, kamu akan memakainya untuk mendiagnosa LoadBalancer bare-metal **MetalLB**).*

---

## 3. Menguji Keterjangkauan & Jalur Jaringan (`ping`, `traceroute`, `mtr`)

### A. `ping` — Mengirim Sinyal Sonar ke Server Tujuan
Perintah `ping` mengirimkan paket kecil bernama **ICMP Echo Request** ke mesin tujuan dan menunggu mesin tersebut membalas (**ICMP Echo Reply**).

```bash
# Kirim tepat 4 kali ping lalu berhenti (-c 4 = count 4)
ping -c 4 1.1.1.1

# Kirim 4 kali ping dengan batas waktu tunggu maksimal 2 detik per ping (-W 2)
ping -c 4 -W 2 google.com
```
*(PENTING: Di Linux, jika kamu mengetik `ping 1.1.1.1` tanpa `-c 4`, ia akan terus melakukan ping selamanya sampai kamu menekan `Ctrl + C`!).*

**Cara Membaca Output `ping`:**
```text
64 bytes from 1.1.1.1: icmp_seq=1 ttl=57 time=11.4 ms
...
--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
```
- **`time=11.4 ms` (*Latency*):** Waktu pulang-pergi yang dibutuhkan paket (dalam milidetik). Semakin kecil semakin cepat!
- **`0% packet loss`:** Tidak ada paket yang hilang di jalan. Jika ada *packet loss* (misal 25% atau 50%), berarti jaringan sedang tidak stabil atau kabel/router bermasalah.

> 🚨 **Jebakan Pemula Soal `ping`:**
> Jika kamu melakukan `ping` ke sebuah server dan hasilnya **100% packet loss (Request timeout)**, **JANGAN langsung menyimpulkan server tersebut mati!**
> Di lingkungan production atau Cloud (seperti AWS), banyak administrator sengaja memblokir protokol ICMP (`ping`) di firewall demi keamanan, padahal layanan web di dalamnya (Port 80/443) tetap hidup normal! Untuk menguji layanan web, selalu gunakan `curl`.

### B. `traceroute` & `mtr` — Melacak di Titik Mana Koneksi Putus
Saat kamu mengakses server di luar kota atau luar negeri, paket datamu sebenarnya melompat melewati belasan router (*hops*). Jika koneksi lambat atau putus di tengah jalan, gunakan **`mtr`** (gabungan modern antara `ping` dan `traceroute`):
```bash
sudo apt install -y mtr-tiny
mtr -rw -c 5 google.com         # Lacak jalur lompatan sebanyak 5 siklus lalu cetak laporannya
```

---

## 4. Mendiagnosa Masalah Nama Domain / DNS (`dig`)

Di kalangan SRE, ada lelucon terkenal yang berbunyi: ***"It's always DNS!"*** (Hampir selalu DNS biang keroknya!). Ketika aplikasi tiba-tiba gagal menghubungi database `db.internal.local`, hal pertama yang harus kamu pastikan adalah: *Apakah nama domain tersebut berhasil diterjemahkan menjadi alamat IP yang benar?*

### Bagaimana Linux Mencari Alamat sebuah Nama Domain?
Saat kamu mengetik nama domain di Linux, sistem operasi mengeceknya dengan urutan berikut (diatur di dalam file `/etc/nsswitch.conf`):
1. **Cek catatan lokal `/etc/hosts` terlebih dahulu:**
   ```bash
   cat /etc/hosts
   ```
   Jika nama domain tersebut tertulis di `/etc/hosts`, Linux langsung memakai IP yang tertulis di situ tanpa bertanya ke internet!
2. **Baru bertanya ke Server DNS yang tercatat di `/etc/resolv.conf`:**
   ```bash
   cat /etc/resolv.conf
   ```

---

### Menggunakan `dig` (Domain Information Groper)
*(Jika perintah `dig` belum tersedia di VM Ubuntu kamu, pasang dengan: `sudo apt install -y dnsutils`).*

**1. Mencari alamat IP dari sebuah nama domain:**
```bash
dig example.com                 # Tampilkan laporan DNS lengkap
dig example.com +short          # TRIK SRE: Langsung tampilkan hasil IP-nya saja tanpa basa-basi!
```

**Cara Membaca Output Lengkap `dig example.com`:**
```text
;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            3412    IN      A       93.184.215.14

;; Query time: 16 msec
;; SERVER: 192.168.64.1#53(192.168.64.1)
```
- **`ANSWER SECTION`:** Menunjukkan bahwa `example.com` memiliki alamat IPv4 (**`A`**) yaitu **`93.184.215.14`**.
- **Angka `3412` (TTL — *Time To Live*):** Berarti jawaban DNS ini boleh disimpan di memori *cache* selama 3412 detik (~56 menit) sebelum harus bertanya ulang ke server DNS.
- **`SERVER:`** Menunjukkan siapa server DNS yang menjawab pertanyaanmu tadi (`192.168.64.1` pada port `53`).

**2. Menguji langsung ke Server DNS tertentu (`@ip_dns`):**
Bagaimana jika kamu curiga DNS lokal di servermu bermasalah? Kamu bisa membandingkannya dengan bertanya langsung ke DNS publik Cloudflare (`1.1.1.1`) atau Google (`8.8.8.8`):
```bash
dig @1.1.1.1 example.com +short
dig @8.8.8.8 example.com +short
```

**3. Mengenal Jenis-Jenis Catatan DNS (*DNS Records*) Penting:**
```bash
dig google.com A +short         # Record 'A'     : Menerjemahkan nama domain -> alamat IPv4
dig google.com AAAA +short      # Record 'AAAA'  : Menerjemahkan nama domain -> alamat IPv6
dig www.github.com CNAME +short # Record 'CNAME' : Nama alias (satu domain menunjuk ke domain lain)
dig google.com MX +short        # Record 'MX'    : Alamat server penerima email (Mail Exchange)
dig google.com TXT +short       # Record 'TXT'   : Catatan teks verifikasi keamanan domain (SPF/DKIM)
dig -x 1.1.1.1 +short           # Reverse Lookup : Kebalikan! Mencari tahu IP 1.1.1.1 itu milik domain apa
```

---

## 5. Memeriksa Port yang Terbuka & Koneksi Aktif (`ss`)

Ketika kamu sudah menyalakan aplikasi web atau database di server, bagaimana cara membuktikan bahwa aplikasi tersebut **benar-benar sudah membuka pintu kamar (*Port*)** dan siap menerima tamu?

Gunakan perintah **`ss`** (*Socket Statistics* — pengganti modern dari perintah tua `netstat`).

### Hafalan Wajib Setiap SRE: `sudo ss -tulpn`
Mengapa kombinasi 5 huruf **`-tulpn`** ini begitu legendaris? Mari kita bedah artinya satu per satu agar mudah diingat:
- **`-t`** (**T**CP) : Tampilkan koneksi protokol TCP (yang dipakai oleh HTTP, HTTPS, SSH, Database).
- **`-u`** (**U**DP) : Tampilkan koneksi protokol UDP (yang dipakai oleh DNS, VPN WireGuard).
- **`-l`** (**L**istening) : Hanya tampilkan port yang sedang **membuka pintu menunggu tamu datang**.
- **`-p`** (**P**rocess) : Tampilkan **nama program dan nomor PID** yang memiliki port tersebut *(wajib pakai `sudo` agar semua nama proses terlihat!)*.
- **`-n`** (**N**umeric) : Tampilkan nomor port dalam bentuk **angka asli** (misal `:80` atau `:22`, jangan diubah menjadi tulisan `:http` atau `:ssh`).

```bash
sudo ss -tulpn
```

*Contoh Output `sudo ss -tulpn`:*
```text
Netid  State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
tcp    LISTEN  0       4096    127.0.0.53%lo:53     0.0.0.0:*          users:(("systemd-resolve",pid=412,fd=14))
tcp    LISTEN  0       128           0.0.0.0:22     0.0.0.0:*          users:(("sshd",pid=655,fd=3))
tcp    LISTEN  0       511         127.0.0.1:8080   0.0.0.0:*          users:(("python3",pid=2410,fd=4))
```

---

### 🔍 Konsep Kritis SRE: Perbedaan `127.0.0.1:Port` vs `0.0.0.0:Port`!
Perhatikan kolom **`Local Address:Port`** pada contoh output di atas! Ini adalah sumber masalah nomor 1 bagi pemula saat mendeploy aplikasi:

| Tertulis di `Local Address:Port` | Artinya | Apakah Bisa Diakses dari Luar Server (Misal dari Browser Mac)? |
|---|---|:---:|
| **`127.0.0.1:8080`** | Aplikasi `python3` hanya membuka pintu di dalam kamar pribadinya sendiri (*Loopback / Localhost*). | ❌ **TIDAK BISA!** Hanya bisa diakses dari dalam terminal VM itu sendiri. |
| **`0.0.0.0:22`** (atau **`*:22`**) | Aplikasi `sshd` membuka pintu pada **SEMUA kartu jaringan** yang ada di server tersebut. | ✅ **BISA!** Bisa diakses dari luar server (selama tidak diblokir firewall). |

> 💡 **Pelajaran Penting:**
> Jika kamu menjalankan aplikasi web (seperti Flask, Node.js, atau Go) di dalam VM OrbStack lalu mencoba membukanya dari browser Mac tapi gagal (*Connection Refused*), segera jalankan **`sudo ss -tulpn`**!
> Jika aplikasi tersebut tertulis mendengarkan di `127.0.0.1:8080`, ubah pengaturan aplikasimu agar melakukan *bind/listen* ke **`0.0.0.0:8080`**.

---

## 6. `curl` — Pisau Swiss-Army Penguji Layanan Web & API

Di server production yang tidak punya tampilan grafis, kita tidak bisa membuka Google Chrome atau Firefox untuk mengetes website. Sebagai gantinya, SRE menggunakan **`curl` (Client URL)**.

### Opsi-Opsi `curl` yang Paling Sering Dipakai SRE

**1. Mengambil isi halaman web atau respons API:**
```bash
curl https://example.com
```

**2. Hanya mengecek Status Kepala / HTTP Header (`-I` = Head only):**
Saat kamu hanya ingin memastikan apakah website hidup (`HTTP 200 OK`) atau error tanpa mau layar terminalmu dipenuhi ribuan baris kode HTML, gunakan **`-I`**:
```bash
curl -I https://example.com
```
*Contoh Output:*
```text
HTTP/2 200
content-type: text/html
date: Fri, 16 Oct 2026 14:00:00 GMT
```
*(Ingat kategori kode HTTP: **`2xx`** = Sukses, **`3xx`** = Redirect/Pindah alamat, **`4xx`** = Kesalahan dari sisi pengakses/Client seperti `401 Unauthorized` atau `404 Not Found`, **`5xx`** = Server sedang rusak/error seperti `500 Internal Server Error` atau `502 Bad Gateway`).*

**3. Mode Detektif Lengkap (`-v` = Verbose):**
Jika koneksi HTTPS gagal atau lambat, tambahkan **`-v`**. `curl` akan menceritakan seluruh proses yang terjadi di balik layar:
```bash
curl -v https://example.com
```
Di output `curl -v`, perhatikan tanda di awal baris:
- Baris diawali **`*`** : Catatan proses internal `curl` (Mencari IP DNS $\rightarrow$ Mencoba terhubung ke IP dan Port $\rightarrow$ Melakukan jabat tangan keamanan SSL/TLS).
- Baris diawali **`>`** : Data *Request Header* yang **dikirim oleh kita** ke server.
- Baris diawali **`<`** : Data *Response Header* yang **dibalaskan oleh server** kepada kita.

**4. Mengikuti pengalihan alamat otomatis (`-L` = Location / Follow Redirect):**
Banyak website otomatis mengalihkan pengunjung dari `http://...` ke `https://...` (kode `301` atau `302`). Tambahkan `-L` agar `curl` otomatis mengikuti alamat barunya:
```bash
curl -IL https://google.com
```

**5. Menguji pengiriman data ke REST API (`POST` & JSON):**
```bash
# Mengirim data JSON ke endpoint API menggunakan metode POST:
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer token-rahasia-saya" \
  -d '{"server": "web-01", "status": "healthy"}'
```

**6. Mengukur kecepatan respons website dan membatasi waktu tunggu (`--max-time`):**
Jangan biarkan script monitoring kamu menggantung selamanya jika server tujuan macet! Batasi waktu tunggunya dengan `--max-time`:
```bash
# Batasi maksimal 5 detik, buang isi HTML-nya ke /dev/null, dan cetak kode status serta total waktunya:
curl -s -o /dev/null --max-time 5 -w "Kode HTTP: %{http_code} | Waktu Total: %{time_total} detik\n" https://example.com
```

---

## 7. Memeriksa Sertifikat TLS (`openssl`) & Menyadap Paket (`tcpdump`)

### A. Mengecek Kapan Sertifikat HTTPS (TLS/SSL) Kedaluwarsa
Salah satu insiden paling memalukan di dunia kerja adalah website perusahaan tidak bisa diakses pengguna karena **sertifikat HTTPS-nya lupa diperpanjang (*Certificate Expired*)**.
Kita bisa memeriksa tanggal kedaluwarsa sertifikat HTTPS website mana pun langsung dari terminal menggunakan `openssl`:

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```
*Contoh Output:*
```text
notBefore=Jan 15 00:00:00 2026 GMT
notAfter=Jan 15 23:59:59 2027 GMT
```
*(Perhatikan baris **`notAfter`**: itulah tanggal dan jam kapan sertifikat tersebut akan mati!).*

---

### B. `tcpdump` — Kamera X-Ray Penyadap Lalu Lintas Jaringan
Bayangkan situasi ini: Temanmu berkata *"Saya sudah mengirim request ke server kamu di port 8080!"*, tapi di log aplikasi servermu **tidak ada catatan apa pun yang masuk**. Siapa yang salah? Apakah paketnya terblokir di router tengah jalan, atau memang sudah sampai ke servermu tapi ditolak?

Untuk membuktikannya secara mutlak, SRE memasang kamera penyadap paket bernama **`tcpdump`** di kartu jaringan server:

```bash
# Pantau semua paket yang keluar-masuk di port 8080 pada semua kartu jaringan (-i any)
# Opsi -n: tampilkan angka IP dan port apa adanya tanpa perlu resolusi DNS
sudo tcpdump -i any -n port 8080
```
Biarkan perintah di atas menyala, lalu minta temanmu mengirim request lagi:
- **Jika layar `tcpdump` tetap sepi tidak bergerak:** Berarti paket tersebut **memang belum pernah sampai ke servermu** (diblokir oleh jaringan luar/router/cloud security group).
- **Jika di layar `tcpdump` muncul tulisan paket masuk:** Berarti jaringan luar aman! Paket sudah tiba di pintu servermu, tinggal cek firewall lokal atau aplikasimu.

*(Tekan `Ctrl + C` untuk menghentikan `tcpdump`).*

---

## 8. Mengenal Satpam Pintu Lokal: Firewall (`ufw`)

Di Ubuntu, alat pengatur firewall standar yang paling ramah pengguna adalah **`ufw` (Uncomplicated Firewall)**.

```bash
# 1. Cek apakah firewall sedang aktif (active) atau mati (inactive):
sudo ufw status verbose

# 2. Contoh cara membuka pintu untuk koneksi SSH (Port 22) dan Web (Port 80/443):
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 3. Contoh mengizinkan akses ke port Database (5432) HANYA dari IP internal tertentu:
sudo ufw allow from 192.168.64.0/24 to any port 5432 proto tcp
```
> ⚠️ **Peringatan Penting Sebelum Mengetik `sudo ufw enable`:**
> Jika kamu sedang terhubung ke server melalui SSH, **PASTIKAN kamu sudah menjalankan `sudo ufw allow 22/tcp` terlebih dahulu** sebelum mengaktifkan firewall dengan `sudo ufw enable`! Jika lupa, kamu akan langsung mengunci dirimu sendiri di luar server!

---

## 9. Pola Kegagalan (Failure Modes) & Urutan Investigasi SRE

Ketika muncul laporan insiden: ***"Aplikasi web di `https://app.perusahaan.com:8080` tidak bisa diakses!"***, seorang SRE tidak menebak-nebak secara acak. Ikuti **Urutan Diagnosa dari Dalam ke Luar (*Inside-Out Troubleshooting*)** berikut ini:

```text
[ Laporan: "Aplikasi tidak bisa diakses!" ]
   │
   ├─► Langkah 1: Cek di DALAM server itu sendiri dulu!
   │     • Apakah proses aplikasinya hidup?     ──> systemctl status myapp
   │     • Apakah port 8080 sedang LISTEN?      ──> sudo ss -tulpn | grep 8080
   │       (Pastikan listen di 0.0.0.0:8080, BUKAN di 127.0.0.1:8080!)
   │     • Coba sapa dari localhost sendiri:    ──> curl -v http://localhost:8080
   │
   ├─► Langkah 2: Jika dari dalam server lancar, cek dari LUAR (Laptop Mac / Client)!
   │     • Apakah nama domain mengarah ke IP yang benar? ──> dig app.perusahaan.com +short
   │     • Apakah rute jaringan ke IP server tembus?     ──> ping -c 4 <IP_SERVER>
   │     • Coba tembak langsung ke IP dan Port-nya:      ──> curl -v http://<IP_SERVER>:8080
   │
   └─► Langkah 3: Jika dari dalam bisa tapi dari luar gagal (Timeout / Refused):
         • Cek apakah paket sampai ke server:   ──> sudo tcpdump -i any -n port 8080
         • Cek apakah Satpam Firewall memblokir:──> sudo ufw status verbose
```

### Arti 3 Pesan Error Koneksi yang Wajib Dibedakan SRE:
1. **`Could not resolve host`:** Masalah murni pada **DNS** (nama domain salah ketik atau server DNS gagal menerjemahkan nama menjadi IP).
2. **`Connection refused` (Koneksi Ditolak Mentah-Mentah):** Paketmu **berhasil sampai ke server tujuan**, TETAPI di nomor port yang kamu tuju **tidak ada aplikasi yang sedang buka warung (`LISTEN`)** (atau aplikasinya hanya mendengarkan di `127.0.0.1`).
3. **`Connection timed out` (Waktu Habis Tidak Ada Jawaban):** Paketmu dikirim, tapi **tidak pernah mendapat balasan**. Biasanya karena IP tujuan salah/mati, rute jaringan terputus, atau paketmu dibuang diam-diam (*DROP*) oleh **Firewall**.

---

## 10. Latihan Praktik Mandiri (10 Menit di Terminal VM OrbStack)

Mari kita uji seluruh peralatan detektif jaringan di atas secara langsung di dalam VM `lab01` kamu:

```bash
# 1. Lihat ringkasan alamat IP dan kartu jaringan di VM kamu:
ip -br addr

# 2. Lihat siapa Default Gateway (gerbang keluar) dan tetangga ARP di jaringanmu:
ip route
ip neigh

# 3. Uji koneksi ke server DNS Cloudflare (1.1.1.1) sebanyak 3 kali:
ping -c 3 1.1.1.1

# 4. Pastikan paket dnsutils terpasang, lalu cek alamat IP dari github.com:
sudo apt update && sudo apt install -y dnsutils
dig github.com +short

# 5. Periksa port apa saja yang saat ini sedang terbuka (LISTEN) di dalam VM kamu:
sudo ss -tulpn

# 6. Mari kita nyalakan web server mini dadakan menggunakan Python di port 8888 (di background):
python3 -m http.server 8888 &
WEB_PID=$!

# 7. Buktikan bahwa sekarang port 8888 muncul di daftar ss -tulpn:
sudo ss -tulpn | grep 8888

# 8. Uji akses ke web server dadakan tersebut menggunakan curl -I dan curl -v:
curl -I http://localhost:8888

# 9. Cek waktu respons dan tanggal kedaluwarsa sertifikat HTTPS milik google.com:
curl -s -o /dev/null -w "Status: %{http_code} | Waktu: %{time_total}s\n" https://www.google.com
echo | openssl s_client -servername www.google.com -connect www.google.com:443 2>/dev/null | openssl x509 -noout -dates

# 10. Matikan kembali web server latihan tadi:
kill -TERM $WEB_PID
```

---

## 11. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Lihat daftar kartu jaringan & alamat IP secara ringkas | `ip -br addr` (atau `ip a`) |
| Lihat tabel jalur routing & Default Gateway | `ip route` (atau `ip r`) |
| Lihat tabel tetangga lokal / MAC Address (ARP) | `ip neigh` |
| Uji koneksi & latency ke sebuah host (4 kali) | `ping -c 4 <host/IP>` |
| Lacak jalur lompatan router (*hops*) secara interaktif | `mtr <host/IP>` |
| Cek terjemahan nama domain ke IP (DNS lookup singkat) | `dig nama.domain +short` |
| Uji DNS langsung ke server resolver tertentu | `dig @1.1.1.1 nama.domain +short` |
| Lihat seluruh port TCP/UDP yang sedang *Listening* + nama prosesnya | `sudo ss -tulpn` |
| Cek header HTTP saja / mode investigasi detail | `curl -I https://...` / `curl -v https://...` |
| Cek tanggal kedaluwarsa sertifikat HTTPS (TLS) | `echo \| openssl s_client -connect host:443 2>/dev/null \| openssl x509 -noout -dates` |
| Sadap paket jaringan yang lewat di port tertentu | `sudo tcpdump -i any -n port 8080` |
| Cek status aturan Firewall Ubuntu | `sudo ufw status verbose` |

---

## 12. Cek Pemahaman (Self-Check)

Coba jawab pertanyaan berikut sebelum mengintip kunci jawaban di bawah:

1. **Apa perbedaan fungsi antara catatan DNS bertipe `A`, `AAAA`, dan `CNAME`?**
2. **Ketika kamu memeriksa port dengan `sudo ss -tulpn`, kamu melihat aplikasi web berjalan di `127.0.0.1:3000`. Mengapa aplikasi tersebut tidak bisa diakses dari laptop temanmu yang berada di jaringan yang sama, dan apa solusinya?**
3. **Saat menjalankan `curl -v http://10.0.0.50:8080`, apa perbedaan arti antara pesan error `Connection refused` dengan `Connection timed out`?**
4. **Mengapa kegagalan perintah `ping` ke sebuah server di internet tidak selalu berarti layanan web di server tersebut sedang mati (*down*)?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **Perbedaan DNS Record `A`, `AAAA`, dan `CNAME`:**
   - **`A` Record:** Memetakan sebuah nama domain langsung ke alamat **IPv4** (misal `example.com` $\rightarrow$ `93.184.215.14`).
   - **`AAAA` Record:** Memetakan sebuah nama domain langsung ke alamat **IPv6** (128-bit).
   - **`CNAME` Record (*Canonical Name*):** Membuat nama alias yang menunjuk ke **nama domain lain** (misal `www.tokoku.com` diarahkan sebagai alias ke `tokoku.com`).

2. **`127.0.0.1:3000` vs `0.0.0.0:3000`:**
   - Alamat `127.0.0.1` adalah antarmuka *loopback* (`localhost`). Aplikasi yang hanya melakukan *bind* ke `127.0.0.1:3000` hanya mau menerima koneksi yang berasal dari dalam mesin itu sendiri.
   - **Solusinya:** Ubah konfigurasi aplikasi agar melakukan *bind/listen* ke **`0.0.0.0:3000`** (semua kartu jaringan) atau ke alamat IP spesifik kartu jaringan server tersebut.

3. **`Connection refused` vs `Connection timed out`:**
   - **`Connection refused`:** Paket permintaan kita berhasil mencapai server `10.0.0.50`, namun sistem operasi server tersebut langsung membalas dengan penolakan karena **tidak ada proses aplikasi yang sedang mendengarkan (*LISTEN*) pada port `8080`** (atau aplikasinya sudah mati).
   - **`Connection timed out`:** Paket kita dikirim, tetapi sampai batas waktu habis tidak ada balasan sama sekali. Biasanya disebabkan oleh **Firewall** yang membuang paket secara diam-diam (*DROP*), rute jaringan yang putus, atau mesin server tujuan sedang mati total.

4. **Mengapa `ping` gagal belum tentu web mati?**
   - Karena `ping` menggunakan protokol **ICMP**, sedangkan layanan web menggunakan protokol **TCP (Port 80/443)**. Banyak administrator server dan penyedia layanan Cloud memblokir lalu lintas ICMP (`ping`) di firewall untuk alasan keamanan, sementara lalu lintas TCP port 80 dan 443 tetap dibuka untuk publik.

</details>
