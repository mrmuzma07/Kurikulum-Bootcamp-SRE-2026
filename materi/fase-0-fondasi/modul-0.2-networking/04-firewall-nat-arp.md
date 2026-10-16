# 04 — Firewall, NAT, Port Forwarding, ARP & Routing (Keamanan & Jalur Paket)

> **Ringkasan:** Memahami bagaimana kernel Linux menyaring paket jaringan menggunakan **Firewall (`ufw` & `iptables`/`nftables`)**, menerjemahkan alamat IP menggunakan **NAT (`SNAT`/`MASQUERADE` & `DNAT`/Port Forwarding)**, menentukan jalur pengiriman lewat **Tabel Routing (`ip route`)**, menemukan alamat fisik kartu jaringan dengan **ARP (`ip neigh` & `arping`)** yang menjadi jantung **MetalLB L2 Mode** di Kubernetes Bare-Metal, serta mempraktikkan **7 Langkah Sistematis Troubleshooting Jaringan**.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Menjelaskan cara kerja **Stateful Firewall**, perbedaan aksi **`DROP` vs `REJECT`**, dan alur paket di dalam kernel Linux (*Netfilter*).
- Mengkonfigurasi firewall menggunakan **`ufw`** secara aman tanpa mengunci akses SSH sendiri.
- Membedakan fungsi **SNAT / MASQUERADE** (jalur keluar Pod/VM ke internet) dan **DNAT / Port Forwarding** (jalur masuk dari luar ke dalam).
- Menjelaskan peran **MAC Address** dan protokol **ARP (*Address Resolution Protocol*)** serta menginspeksi *ARP Cache* dengan `ip neigh` dan `arping`.
- Menjelaskan cara kerja **MetalLB L2 Mode** dan **Gratuitous ARP** pada cluster Kubernetes On-Premises.
- Membaca **Tabel Routing (`ip route`)**, memahami aturan *Longest Prefix Match*, dan menguji rute dengan `ip route get`.
- Mendiagnosa masalah *"Service tidak bisa diakses"* secara sistematis menggunakan **Urutan 7-Langkah SRE** (bukan menebak-nebak).

---

## 1. Mindset Dasar: Pos Satpam, Penerjemah Alamat, dan Petunjuk Jalan

Mari kita lengkapi peta besar perjalanan paket jaringan kita di Modul 0.2:
1. **[Topik 01 (DNS & CIDR)](01-dns-cidr.md):** Menemukan alamat IP gedung tujuan dari nama domain.
2. **[Topik 02 (TCP, UDP & TLS)](02-tcp-udp-tls.md):** Membangun pipa sambungan TCP dan brankas enkripsi TLS.
3. **[Topik 03 (HTTP & Reverse Proxy)](03-http-proxy.md):** Berbicara dengan bahasa HTTP melalui resepsionis Reverse Proxy.
4. **Topik 04 (Firewall, NAT, ARP & Routing) ini:** Menjawab pertanyaan di balik layar sistem operasi:
   - *"Saat paket tiba di pintu jaringan server, apakah paket ini **boleh masuk atau harus diusir**?"* $\rightarrow$ Tugas **Firewall**.
   - *"Bagaimana komputer atau Pod ber-IP lokal/privat bisa **berbicara ke internet publik**?"* $\rightarrow$ Tugas **NAT**.
   - *"Lewat pintu kabel mana paket ini harus dikirim: **langsung ke tetangga sebelah** atau **titipkan ke Router (Gateway)**?"* $\rightarrow$ Tugas **Routing & ARP**.

---

## 2. Firewall — Pos Satpam Penjaga Server

### A. Apa Itu Firewall dan Bagaimana Cara Kerjanya?
Bayangkan server Linux kamu adalah sebuah gedung perkantoran yang memiliki 65.535 pintu (**Port**). Tanpa **Firewall**, siapa pun dari internet bisa mengetuk pintu mana saja yang kebetulan sedang terbuka.
**Firewall** adalah **Pos Satpam** yang berdiri tepat di gerbang kartu jaringan server untuk memeriksa setiap paket data yang lewat berdasarkan daftar aturan (*Rules*):
- Dari mana asal paket ini? (*Source IP*)
- Mau menuju ke pintu nomor berapa? (*Destination Port*)
- Menggunakan protokol apa? (*TCP / UDP / ICMP*)

### B. Dua Konsep Penting Firewall yang Wajib Dipahami Pemula

#### 1. Konsep *Stateful Firewall* (Satpam yang Punya Ingatan)
Pemula sering bingung:
> *"Jika kita mengatur firewall menjadi **` Tolak Semua Paket Masuk (Deny Incoming)`** dan **`Izinkan Semua Paket Keluar (Allow Outgoing)`**, lalu bagaimana caranya server kita masih bisa mengunduh paket saat menjalankan `sudo apt update` atau `curl https://google.com`? Bukankah balasan dari Google itu adalah paket yang **masuk** ke server kita?"*

Jawabannya: Karena firewall modern di Linux bersifat **Stateful** (memiliki buku catatan status koneksi / *Connection Tracking*):
- Saat server kamu mengirim permintaan keluar ke Google (`Outgoing`), satpam Firewall mencatat di bukunya: *"Server kita sedang menunggu balasan dari IP Google"*.
- Ketika paket balasan dari Google tiba di pintu masuk (`Incoming`), satpam melihat status paket tersebut adalah **`ESTABLISHED`** (bagian dari percakapan resmi yang dimulai dari dalam), sehingga **otomatis diizinkan masuk!**
- Yang ditolak oleh aturan `Deny Incoming` adalah **tamu tak diundang dari luar yang tiba-tiba mengetuk pintu duluan** (paket pembuka `SYN` baru dari luar).

#### 2. Bedanya Diblokir secara `REJECT` vs `DROP`
Saat satpam Firewall memutuskan untuk menolak sebuah paket, ada dua cara penolakan yang memberikan **gejala error sangat berbeda** di layar terminal pemanggil:

| Aksi Firewall | Analogi Satpam | Apa yang Terjadi di Jaringan? | Gejala yang Dilihat oleh Pemanggil (`curl` / `ssh`) |
|:---:|---|---|---|
| **`REJECT`** | Satpam langsung membuka jendela dan berteriak: **"Maaf, Anda dilarang masuk! Pergi!"** | Firewall mengirim paket balasan penolakan (`TCP RST` atau `ICMP Port Unreachable`) kembali ke pemanggil. | **Langsung gagal detik itu juga (< 1 detik)** dengan pesan: **`Connection refused`**! |
| **`DROP`** | Satpam **pura-pura mati / diam seribu bahasa**. Paket suratmu langsung dibuang ke tempat sampah tanpa jawaban apa pun. | Tidak ada paket balasan sama sekali. Pemanggil terus menunggu di status `SYN-SENT` dan mengirim ulang paket `SYN` berkali-kali. | **Layar menggantung lama (10–60 detik)** lalu berakhir dengan pesan: **`Connection timed out`**! |

> 💡 **Tips Diagnosa SRE:**
> Jika kamu melakukan `curl` ke sebuah server dan langsung mendapat **`Connection refused`** dalam 0,01 detik, artinya paketmu berhasil sampai ke server (atau firewall me-`REJECT`-nya), namun tidak ada aplikasi yang mendengarkan di port tersebut.
> Tetapi jika `curl` kamu **diam menggantung lama lalu `Connection timed out`**, hampir pasti paketmu dibuang secara **`DROP`** oleh Firewall di tengah jalan (atau IP tujuan memang tidak aktif/salah rute)!

---

### C. Alur Paket di Kernel Linux (*Netfilter*) & Alat Pengaturnya
Di dalam jantung sistem operasi Linux (Kernel), mesin penyaring paket bernama **Netfilter** memiliki 5 pos pemeriksaan (*Chains*):

```text
                       ┌──────────────────────────────────────┐
                       │     PROSES APLIKASI LOKAL DI VM      │
                       │     (sshd, Caddy, Python, k3s)       │
                       └──────────▲────────────────┬──────────┘
                                  │                │
                         Pos 2: INPUT            Pos 4: OUTPUT
                      (Menuju aplikasi lokal)  (Keluar dari aplikasi lokal)
                                  │                ▼
  Paket Masuk ──► [ Pos 1: PREROUTING ] ──► ( Keputusan Routing ) ──► [ Pos 5: POSTROUTING ] ──► Paket Keluar
  dari Kabel      (Ubah Tujuan / DNAT)             │                  (Ubah Sumber / SNAT)       ke Kabel
                                                   │
                                            Pos 3: FORWARD
                                    (Hanya numpang lewat ke VM/Pod lain)
```

Untuk mengatur mesin **Netfilter** di kernel tersebut, kita menggunakan alat baris perintah (CLI). Ada 3 nama alat yang sering kamu dengar:
1. **`iptables`:** Alat klasik Linux yang sangat detail (masih sangat sering dipakai oleh Docker dan Kubernetes di balik layar).
2. **`nftables` (`nft`):** Penerus modern dari `iptables` di Ubuntu terbaru.
3. **`ufw` (*Uncomplicated Firewall*):** Pembungkus ramah manusia buatan Ubuntu agar kamu tidak perlu menghafal sintaks `iptables`/`nftables` yang panjang dan rumit. Saat kamu mengetik perintah `ufw` yang pendek, `ufw` otomatis menerjemahkannya menjadi aturan `iptables`/`nftables` di belakang layar!

---

## 3. Mengkonfigurasi Firewall Dasar dengan `ufw`

### 🚨 Aturan Emas Sebelum Menyalakan Firewall via SSH!
Saat kamu sedang meremote server melalui **SSH (Port 22)**, **JANGAN PERNAH** langsung menjalankan `sudo ufw enable` sebelum mengizinkan port `22/tcp` terlebih dahulu! Jika lupa, begitu firewall aktif dan memblokir semua port masuk, sesi SSH kamu akan terputus dan kamu **terkunci di luar server (*locked out*)**!

**Urutan Standar yang Aman (Best Practice SRE):**
```bash
# 1. Periksa status ufw saat ini (default di VM baru biasanya: inactive)
sudo ufw status verbose

# 2. Atur kebijakan dasar (Default Policy):
#    - Tolak semua koneksi masuk yang tidak diizinkan secara eksplisit
#    - Izinkan semua koneksi keluar dari server kita
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 3. WAJIB PERTAMA: Izinkan pintu SSH (Port 22 TCP) agar kita tidak terkunci!
sudo ufw allow 22/tcp

# 4. Izinkan port untuk layanan web publik (HTTP port 80 & HTTPS port 443):
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 5. Contoh Aturan Spesifik (Sangat Sering Dipakai di Production On-Prem):
#    Izinkan akses ke Port 6443 (Kubernetes API) HANYA dari subnet internal 192.168.64.0/24!
sudo ufw allow from 192.168.64.0/24 to any port 6443 proto tcp

# 6. Baru nyalakan firewall setelah aturan SSH dipastikan ada:
sudo ufw --force enable

# 7. Verifikasi daftar aturan beserta nomor urutnya:
sudo ufw status numbered
```

Contoh tampilan `sudo ufw status numbered`:
```text
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 443/tcp                    ALLOW IN    Anywhere
[ 4] 6443/tcp                   ALLOW IN    192.168.64.0/24
```

### Cara Menghapus Aturan atau Mematikan `ufw`
```bash
# Menghapus aturan berdasarkan nomor urut (misal ingin menghapus aturan nomor 4):
sudo ufw delete 4

# Atau menghapus berdasarkan nama aturannya:
sudo ufw delete allow 80/tcp

# Mematikan firewall sepenuhnya (kembali ke mode terbuka saat eksplorasi lab):
sudo ufw disable
```

> 🛡️ **Trik Penyelamat SRE Saat Mengutak-atik Firewall di Server Jauh:**
> Jika kamu ragu aturan baru akan memutus koneksi SSH-mu di server produksi, pasang "bom waktu pembatal otomatis" selama 60 detik:
> ```bash
> sudo ufw enable && sleep 60 && sudo ufw disable
> ```
> Jika setelah `ufw enable` koneksi SSH-mu masih aman, cukup tekan `Ctrl+C` untuk membatalkan `sleep 60`. Namun jika koneksi SSH-mu terputus dan macet, tunggu saja 60 detik — perintah `sudo ufw disable` akan otomatis berjalan dan menyelamatkanmu!

---

## 4. NAT (*Network Address Translation*) & Port Forwarding

### A. Mengapa Jaringan Membutuhkan NAT?
Di [Topik 01](01-dns-cidr.md), kita belajar bahwa alamat **IP Privat** (seperti `192.168.x.x` di Wi-Fi rumah, `198.19.x.x` di VM OrbStack, atau `10.42.x.x` di Pod Kubernetes) **hanya berlaku di jaringan lokal dan tidak dikenal di Internet publik**.
Lalu, jika sebuah Pod Kubernetes ber-IP `10.42.0.15` ingin mengunduh data dari `1.1.1.1` di internet, bagaimana paket balasan dari `1.1.1.1` tahu jalan pulang ke `10.42.0.15`?

Jawabannya adalah melalui **NAT (*Network Address Translation*)** — mekanisme **mengganti alamat IP/Port pengirim atau penerima** di amplop paket saat paket melewati komputer perantara (Router / Host / Node).

Ada dua jenis NAT yang wajib kamu kuasai:

---

### B. 1. `SNAT` / `MASQUERADE` (Mengganti Alamat Pengirim Saat Keluar)
- **Singkatan:** *Source Network Address Translation* (*Masquerade* = menyamar).
- **Analogi Kantor:** Karyawan di meja nomor 15 (`Pod 10.42.0.15`) mengirim surat keluar gedung. Sebelum surat keluar ke kantor pos umum, Resepsionis Gedung (**Node Linux**) menghapus tulisan *"Pengirim: Meja 15"* dan menggantinya dengan **"Pengirim: Gedung Utama Jl. Sudirman No. 1 (IP Node)"**. Saat balasan tiba di Gedung Utama, resepsionis melihat buku catatannya lalu meneruskannya kembali ke Meja 15!

```text
  ALUR SNAT / MASQUERADE (Dari Dalam Keluar ke Internet):

  [ Pod Kubernetes ]          [ Node Linux (Pos POSTROUTING) ]          [ Server di Internet ]
   IP: 10.42.0.15                     IP: 192.168.64.5                       IP: 1.1.1.1
         │                                   │                                    │
         │ 1. Dari: 10.42.0.15               │ 2. MASQUERADE (Ganti Pengirim!)    │
         │    Ke  : 1.1.1.1                  │    Dari: 192.168.64.5              │
         │ ────────────────────────────────► │    Ke  : 1.1.1.1                   │
         │                                   │ ─────────────────────────────────► │
```
> **Kaitan dengan Kubernetes & OrbStack:** Inilah alasan mengapa container Docker dan Pod Kubernetes bisa langsung berinternet (`apt update`, `curl`) meskipun menggunakan IP internal privat!

---

### C. 2. `DNAT` / Port Forwarding (Mengganti Alamat Tujuan Saat Masuk)
- **Singkatan:** *Destination Network Address Translation*.
- **Analogi Kantor:** Tamu dari luar mengirim paket ke **"Gedung Utama Pintu 8080"**. Begitu paket tiba di gerbang depan, satpam melihat aturan **Port Forwarding (DNAT)**: *"Semua paket ke Pintu 8080 harus langsung dibelokkan tujuannya ke Kamar Dalam `10.42.0.15` Pintu `80`!"*.

```text
  ALUR DNAT / PORT FORWARDING (Dari Luar Masuk ke Dalam):

  [ Laptop Kamu ]            [ Node Linux (Pos PREROUTING) ]           [ Pod / Server Internal ]
                                   IP: 192.168.64.5                         IP: 10.42.0.15
         │                                 │                                       │
         │ 1. Ke: 192.168.64.5:30080       │ 2. DNAT (Ganti Tujuan!)               │
         │ ──────────────────────────────► │    Ke: 10.42.0.15:80                  │
         │                                 │ ────────────────────────────────────► │
```

| Jenis NAT | Apa yang Diubah? | Arah Lalu Lintas | Contoh Nyata di Dunia SRE |
|---|---|---|---|
| **`SNAT` / `MASQUERADE`** | **Alamat Pengirim (*Source IP*)** diubah menjadi IP milik Router/Node. | **Keluar (*Outbound*)** — dari jaringan privat ke jaringan luar. | Pod Kubernetes atau VM mengakses internet luar. |
| **`DNAT` (*Port Forwarding*)** | **Alamat Tujuan (*Destination IP:Port*)** diubah menjadi IP:Port server internal. | **Masuk (*Inbound*)** — dari luar menuju layanan di dalam jaringan privat. | • **Kubernetes Service (`ClusterIP` & `NodePort`)**: `kube-proxy` menggunakan aturan `DNAT` di `iptables` untuk membelokkan trafik dari IP Service ke IP Pod!<br>• **OrbStack Port Forwarding** & **SSH Tunnel (`ssh -L`)**. |

```bash
# Cara mengintip aturan NAT (SNAT/MASQUERADE & DNAT) yang aktif di kernel Linux:
sudo iptables -t nat -L -n -v
```

---

## 5. ARP (*Address Resolution Protocol*) — Bagaimana IP Menemukan MAC Address

### A. Mengapa Sudah Punya IP Address Masih Butuh MAC Address?
Pemula sering bertanya: *"Kita sudah punya alamat IP (`192.168.64.5`). Kenapa komputer masih butuh **MAC Address** (alamat fisik 6 pasang heksadesimal di kartu jaringan seperti `a2:b4:c6:d8:e0:f1`)?"*

Gunakan **Analogi Pengiriman Paket di Kompleks Perumahan**:
- **IP Address (Layer 3 — *Network Layer*)** adalah **Alamat Pos Resmi** (misal: *Kompleks Griya Asri No. 10*). Alamat ini dipakai oleh kurir untuk menentukan rute dari kota ke kota hingga tiba di gerbang Kompleks Griya Asri.
- **MAC Address (Layer 2 — *Data Link / Ethernet Layer*)** adalah **Ciri Fisik / Plat Nomor Rumah di Dalam Gang** yang sama. Begitu kurir sudah masuk ke dalam gang kompleks yang sama (**satu Subnet lokal**), kabel jaringan/switch di gang itu tidak bisa membaca tulisan IP Address; kartu jaringan fisik hanya bisa mengirim sinyal listrik dari **satu MAC Address ke MAC Address lainnya**!

### B. Cara Kerja ARP: Teriak ke Satu Ruangan (*Broadcast*), Jawab Pribadi (*Unicast*)
Ketika komputer A (`192.168.64.2`) ingin mengirim paket ke komputer B (`192.168.64.5`) yang berada di **satu subnet yang sama**:

```text
  Langkah 1: Komputer A mengecek buku catatan "ARP Cache" miliknya sendiri (`ip neigh show`):
             "Apakah saya sudah tahu MAC Address milik 192.168.64.5?"
             ├── Jika SUDAH ADA di catatan ──► Langsung kirim paket ke MAC Address tersebut!
             └── Jika BELUM ADA ─────────────► Lakukan Langkah 2 (ARP Request).

  Langkah 2: ARP REQUEST (Broadcast — Teriak Pakai Megafon ke Seluruh Komputer di Subnet):
             Komputer A ──► "HALO SEMUA! Siapa yang punya IP 192.168.64.5? Tolong beritahu MAC-mu!"
                            (Dikirim ke alamat Broadcast MAC: ff:ff:ff:ff:ff:ff)

  Langkah 3: ARP REPLY (Unicast — Dijawab Langsung oleh Pemiliknya):
             Komputer B ──► "Halo Komputer A, IP 192.168.64.5 itu milikku! MAC-ku adalah aa:bb:cc:11:22:33"

  Langkah 4: Komputer A menyimpan jawaban tersebut di ARP Cache selama ~60 detik,
             lalu mulai mengirimkan paket data ke aa:bb:cc:11:22:33!
```

### C. Membaca Tabel ARP Cache (`ip neigh show`) dan Statusnya
Di Linux modern, perintah untuk melihat tabel ARP adalah **`ip neigh show`** (kependekan dari *IP Neighbor* / tetangga satu subnet):

```bash
ip neigh show
```
Contoh output:
```text
192.168.64.1 dev eth0 lladdr aa:bb:cc:dd:ee:01 REACHABLE
192.168.64.8 dev eth0 lladdr aa:bb:cc:dd:ee:08 STALE
192.168.64.99 dev eth0 INCOMPLETE
```

Arti kata di kolom paling kanan (sangat penting saat *debugging*):
- **`REACHABLE`** : Sehat! Baru saja berkomunikasi beberapa detik lalu, alamat MAC (`lladdr`) valid dan aktif.
- **`STALE`** : Catatan MAC ini sudah agak lama (> 30–60 detik) tidak dipakai. Ini **normal**; saat nanti akan dipakai lagi, Linux akan mengecek ulang apakah pemiliknya masih hidup.
- **`INCOMPLETE` / `FAILED`** : 🚨 **Masalah Lapisan 2!** Komputer kita sudah berteriak mengirim *ARP Request* berkali-kali (*"Siapa pemilik IP `192.168.64.99`?"*), tetapi **tidak ada satu pun mesin di subnet itu yang menjawab!** Artinya IP tujuan tersebut mati, kabel/VLAN-nya terpisah, atau salah ketik IP!

```bash
# Perintah praktis SRE untuk menguji ARP secara manual:
# 1. Tanya langsung lewat protokol ARP siapa pemilik sebuah IP di kartu jaringan eth0:
sudo arping -I eth0 -c 3 192.168.64.1

# 2. Bersihkan (flush) catatan ARP cache lama jika ada IP yang baru saja pindah mesin:
sudo ip neigh flush dev eth0
```

---

## 6. Mengapa ARP Adalah Jantung dari MetalLB L2 Mode di Kubernetes On-Prem!

Bagian ini adalah **jembatan terpenting menuju Fase 2 (Modul 2.3 — MetalLB)**.
Saat nanti kamu membangun cluster Kubernetes di data center kantor (*On-Premises / Bare-Metal*) atau di atas VM OrbStack, kamu akan memiliki beberapa Node pekerja (misal `node1`, `node2`, `node3`).
Ketika kamu membuat Service Kubernetes bertipe **`LoadBalancer`** dan memberinya satu **Virtual IP (VIP)** baru — misalnya **`192.168.64.240`** — padahal tidak ada satu pun kartu jaringan fisik yang dipasang IP `192.168.64.240`, **bagaimana caranya laptopmu atau router kantor tahu ke Node mana paket untuk `192.168.64.240` harus dikirim?**

Di sinilah **MetalLB (Layer 2 / ARP Mode)** beraksi menggunakan protokol **ARP**!

```text
  CARA KERJA METALLB L2 MODE & GRATUITOUS ARP:

  1. Kamu membuat Service LoadBalancer -> MetalLB memberikan Virtual IP: 192.168.64.240.
  2. MetalLB memilih SATU Node sehat (misal: Node-1) sebagai "Pemimpin (Leader)" untuk IP tersebut.
  3. Pod speaker MetalLB di Node-1 mendengarkan seluruh pertanyaan ARP di jaringan:
     • Laptop bertanya (ARP Request) : "Siapa yang punya IP 192.168.64.240?"
     • Node-1 menjawab (ARP Reply)   : "ITU MILIKKU! Kirimkan ke MAC Address Node-1 (aa:bb:cc:00:00:01)!"

  4. BAGAIMANA JIKA NODE-1 TIBA-TIBA MATI (FAILOVER)?
     • MetalLB langsung menunjuk Node-2 sebagai Leader baru.
     • Tanpa menunggu ditanya, Node-2 langsung berteriak ke seluruh jaringan menggunakan
       GRATUITOUS ARP (GARP):
       📢 "PENGUMUMAN KE SEMUA KOMPUTER! Mulai detik ini, IP 192.168.64.240
           pindah ke MAC Address Node-2 (aa:bb:cc:00:00:02)! Segera perbarui ARP Cache kalian!"
```

### Apa Itu *Gratuitous ARP (GARP)*?
**Gratuitous ARP** adalah pengumuman ARP yang dikirim secara sukarela (**tanpa ada yang bertanya terlebih dahulu**) ke seluruh komputer di satu subnet untuk memberitahukan bahwa kepemilikan suatu alamat IP telah berpindah ke MAC Address yang baru.

### Tiga Implikasi Nyata yang Wajib Diingat SRE Saat Mengoperasikan MetalLB L2:
1. **Wajib Berada di Satu Subnet yang Sama:** Karena pesan *ARP Broadcast* tidak bisa menembus melewati Router ke subnet lain, maka rentang IP Pool yang kamu berikan ke MetalLB **wajib berada di subnet yang sama** dengan IP Node Kubernetes kamu!
2. **Jangan Sampai IP Pool MetalLB Bentrok (*ARP Conflict*):** Jika kamu memberikan IP `192.168.64.240` ke MetalLB, padahal IP itu juga dipakai oleh VM lain di jaringan yang sama, maka **dua mesin akan saling berebut menjawab ARP!** Akibatnya koneksi akan putus-nyambung secara acak (50% sukses, 50% gagal). Cara mendeteksinya adalah menjalankan `sudo arping -I eth0 192.168.64.240` — jika muncul **dua MAC Address berbeda** yang menjawab, berarti terjadi tabrakan IP!
3. **Efek ARP Cache Lama Saat Failover:** Ketika Node-1 mati dan Node-2 mengirim *Gratuitous ARP*, kadang ada klien/perangkat jaringan yang lambat memperbarui *ARP Cache*-nya sehingga paket masih nyasar ke MAC Address Node-1 yang sudah mati selama beberapa detik. Menyelesaikan masalah itu secara instan bisa dilakukan dengan menghapus cache ARP (`sudo ip neigh flush dev eth0`).

---

## 7. Routing (`ip route`) — Peta Penentu Arah Paket

Jika **ARP** mengurus pengiriman ke tetangga di **dalam satu gang (subnet) yang sama**, lalu bagaimana jika komputer kita (`192.168.64.5`) ingin mengirim paket ke alamat di luar gang kita, misalnya ke Database di subnet sebelah (`10.50.0.10`) atau ke Internet (`8.8.8.8`)?
Komputer kita akan membuka buku peta miliknya yang disebut **Tabel Routing (*Routing Table*)**!

### A. Membaca Tabel Routing dengan `ip route show`
Mari kita jalankan `ip route show` di VM dan bedah artinya baris demi baris:

```text
default via 192.168.64.1 dev eth0 proto dhcp src 192.168.64.5 metric 100
192.168.64.0/24 dev eth0 proto kernel scope link src 192.168.64.5
10.42.0.0/24 via 192.168.64.10 dev eth0
```

Mari kita terjemahkan ke dalam Bahasa Manusia:
1. **Baris 2 (`192.168.64.0/24 dev eth0 scope link` — Rute Lokal Langsung):**
   *"Jika tujuan paket berada di dalam rentang `192.168.64.1` s.d. `192.168.64.254`, itu adalah tetangga satu ruangan kita! **Jangan lewat router**, langsung teriak pakai **ARP** melalui kartu jaringan `eth0`!"*
2. **Baris 3 (`10.42.0.0/24 via 192.168.64.10 dev eth0` — Rute Statis Khusus):**
   *"Jika tujuan paket menuju ke jaringan Pod Kubernetes `10.42.0.x`, titipkan paket tersebut ke mesin `192.168.64.10` lewat pintu `eth0`."*
3. **Baris 1 (`default via 192.168.64.1 dev eth0` — Default Gateway / Pintu Keluar Utama):**
   *"Jika tujuan paket **tidak cocok** dengan daftar subnet mana pun di atas (misalnya mau ke `8.8.8.8` di internet), titipkan saja paketnya ke **Router Utama (Default Gateway)** di alamat `192.168.64.1`!"*

### B. Aturan Emas Routing: *Longest Prefix Match* (Yang Paling Spesifik Selalu Menang!)
Bagaimana jika sebuah paket menuju `10.42.0.55`? Bukankah `10.42.0.55` cocok dengan aturan `default` (`0.0.0.0/0`) sekaligus cocok dengan aturan `10.42.0.0/24`?
Kernel Linux selalu memilih aturan dengan **angka CIDR (`/prefix`) paling besar (paling spesifik)**:
- `/24` lebih spesifik daripada `/16`, dan `/16` lebih spesifik daripada `default` (`/0`).
- Jadi paket ke `10.42.0.55` pasti memilih jalur `10.42.0.0/24`!

### C. Trik Favorit SRE: Bertanya Langsung ke Kernel dengan `ip route get`
Daripada pusing menghitung mata kepala sendiri saat tabel routing di server Kubernetes mencapai puluhan baris, tanyakan langsung kepada kernel Linux menggunakan **`ip route get <IP-Tujuan>`**:

```bash
# Tanya kernel: "Kalau saya mau kirim paket ke 8.8.8.8, lewat mana jalurnya?"
ip route get 8.8.8.8
# Output: 8.8.8.8 via 192.168.64.1 dev eth0 src 192.168.64.5 uid 1000
# Artinya: Lewat Gateway 192.168.64.1, keluar dari kartu eth0, menggunakan IP pengirim 192.168.64.5!
```

---

## 8. Urutan 7-Langkah Troubleshooting "Trafik Tidak Sampai"

Saat pengguna atau rekan tim melapor: *"Mas/Mbak SRE, aplikasi `https://app.lab.local` tidak bisa diakses!"*, seorang SRE profesional **tidak pernah langsung menebak atau me-restart server secara membabi buta**.
Gunakan **Urutan 7-Langkah dari Bawah ke Atas (*Bottom-Up*)** berikut ini — yang akan langsung kamu praktikkan di **[LAB-03: Trace Masalah Koneksi](lab/LAB-03-trace-koneksi.md)**:

```text
  ┌───────────────────────────────────────────────────────────────────────────────────┐
  │  LANGKAH 7 — APLIKASI & HTTP     : curl -v, journalctl -u <app>                   │
  │  "Apakah aplikasi membalas 500/502/403? Apa isi pesan error di log aplikasi?"     │
  ├───────────────────────────────────────────────────────────────────────────────────┤
  │  LANGKAH 6 — RESOLUSI NAMA (DNS) : dig <domain> +short, getent hosts <domain>     │
  │  "Apakah nama domain menghasilkan angka IP yang benar?"                           │
  ├───────────────────────────────────────────────────────────────────────────────────┤
  │  LANGKAH 5 — PORT & LISTENING    : sudo ss -tlnp                                  │
  │  "Apakah proses aplikasi benar-benar hidup dan LISTEN di 0.0.0.0:<port>?"         │
  ├───────────────────────────────────────────────────────────────────────────────────┤
  │  LANGKAH 4 — FIREWALL & NAT      : sudo ufw status, sudo iptables -L -n -v        │
  │  "Apakah paket diblokir (DROP/REJECT) oleh satpam firewall di server?"            │
  ├───────────────────────────────────────────────────────────────────────────────────┤
  │  LANGKAH 3 — ROUTING (L3)        : ip route get <IP-Target>                       │
  │  "Apakah server tahu jalan menuju IP target dan punya Default Gateway?"           │
  ├───────────────────────────────────────────────────────────────────────────────────┤
  │  LANGKAH 2 — TETANGGA & ARP (L2) : ip neigh show, sudo arping -I eth0 <IP>        │
  │  "Apakah target/gateway di subnet yang sama menjawab ARP (bukan INCOMPLETE)?"     │
  ├───────────────────────────────────────────────────────────────────────────────────┤
  │  LANGKAH 1 — KARTU JARINGAN (L1) : ip -br link show, ip -br addr show             │
  │  "Apakah kartu jaringan (eth0) berstatus UP dan sudah memiliki alamat IP?"        │
  └───────────────────────────────────────────────────────────────────────────────────┘
```

> 💡 **Mengapa Urutan Ini Sangat Ampuh?**
> Dalam kenyataan sehari-hari, kamu bisa memulai pengecekan cepat dari **Langkah 6 & 5** (`dig` dan `ss -tlnp`), lalu jika paket jaringan tidak kunjung sampai ke port tersebut, telusuri dari **Langkah 1 $\rightarrow$ 2 $\rightarrow$ 3 $\rightarrow$ 4** secara berurutan. Dengan cara ini, tidak ada satu pun komponen jaringan yang terlewat dari pantauanmu!

---

## 9. Pola Kegagalan (Failure Modes) di Production & Cara Diagnosanya

| Gejala / Masalah yang Terjadi | Penyebab Utama | Cara Diagnosa & Solusi SRE |
|---|---|---|
| Setelah mengaktifkan `ufw`, sesi SSH langsung putus dan tidak bisa login lagi | Lupa menjalankan `sudo ufw allow 22/tcp` sebelum `sudo ufw enable`. | Masuk melalui *Console Lokal* (di OrbStack: gunakan `orb shell lab01` yang tidak lewat jalur jaringan SSH!), lalu jalankan `sudo ufw allow 22/tcp`. |
| IP LoadBalancer MetalLB tiba-tiba gagal diakses separuh waktu (kadang berhasil, kadang error) | **ARP Conflict (IP Bentrok)!** Ada VM/komputer lain di jaringan yang memakai IP yang sama dengan IP Pool MetalLB. | Jalankan `sudo arping -I eth0 <VIP-MetalLB>`. Jika muncul 2 MAC Address berbeda yang membalas, ubah konfigurasi `IPAddressPool` MetalLB ke rentang IP yang benar-benar kosong. |
| `ping` atau `curl` ke tetangga satu subnet menghasilkan **`Destination Host Unreachable`** | Kegagalan resolusi **ARP** (mesin tujuan mati atau tidak ada di VLAN tersebut). | Cek `ip neigh show`. Jika tertulis **`INCOMPLETE`** atau **`FAILED`**, artinya tidak ada mesin yang menjawab *ARP Request* untuk IP tersebut. |
| Muncul pesan instan **`Network is unreachable`** saat mencoba menghubungi internet | Tabel routing kehilangan baris **`default` (*Default Gateway*)** atau kartu jaringan berstatus `DOWN`. | Cek `ip -br addr` dan `ip route show`. Pastikan interface `UP` dan rute `default via <IP-Gateway>` tersedia. |

---

## 10. Latihan Praktik Mandiri (15 Menit di Terminal VM OrbStack)

Mari kita uji langsung tabel ARP, tabel Routing, aturan Firewall `ufw`, serta tabel NAT di terminal VM `lab01` kamu:

```bash
# 1. Cek status kartu jaringan (Langkah 1) dan catat nama interface utama (biasanya eth0):
ip -br link show
ip -br addr show

# 2. Cek Tabel Routing dan tanyakan rute menuju 1.1.1.1 kepada kernel (Langkah 3):
ip route show
ip route get 1.1.1.1

# 3. Simpan IP Default Gateway VM kita ke dalam variabel agar mudah diuji:
GW_IP=$(ip route | awk '/^default/ {print $3}')
IFACE=$(ip route | awk '/^default/ {print $5}')
echo "Gateway VM adalah $GW_IP melalui kartu jaringan $IFACE"

# 4. Lihat isi Tabel ARP Cache saat ini (Langkah 2):
ip neigh show

# 5. Uji mengirim paket ARP secara langsung ke Gateway menggunakan arping:
#    (Jika arping belum terpasang, pasang dulu dengan: sudo apt-get update && sudo apt-get install -y iputils-arping)
sudo arping -I "$IFACE" -c 3 "$GW_IP"

# 6. Coba bersihkan (flush) ARP Cache, lalu lihat bagaimana Linux otomatis mengisinya lagi:
sudo ip neigh flush dev "$IFACE"
ip neigh show
ping -c 1 "$GW_IP" >/dev/null
ip neigh show

# 7. Latihan Konfigurasi Firewall ufw yang Aman (Langkah 4):
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp                # WAJIB: Amankan SSH dulu!
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable

#    Lihat daftar aturan beromor:
sudo ufw status numbered

#    Matikan kembali ufw agar tidak mengganggu eksplorasi lab berikutnya:
sudo ufw disable

# 8. Intip tabel NAT di kernel Linux:
sudo iptables -t nat -L -n -v
```

**Acceptance Criteria:**
- [ ] Mampu menyebutkan alamat IP *Default Gateway* VM `lab01` dari output `ip route show`.
- [ ] Mampu menampilkan MAC Address milik *Default Gateway* menggunakan `ip neigh show` dan `sudo arping`.
- [ ] Berhasil mengaktifkan `ufw` dengan mengizinkan port `22/tcp`, `80/tcp`, dan `443/tcp` tanpa terputus dari sesi SSH, lalu mematikannya kembali (`sudo ufw disable`).

---

## 11. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Lihat status kartu jaringan & alamat IP secara ringkas | `ip -br link show` dan `ip -br addr show` |
| Lihat tabel routing lengkap | `ip route show` |
| Tanya kernel jalur mana yang dipakai untuk menuju suatu IP | `ip route get <IP-Tujuan>` |
| Lihat tabel *ARP Cache* (daftar IP tetangga $\leftrightarrow$ MAC Address) | `ip neigh show` |
| Hapus/reset *ARP Cache* pada interface tertentu | `sudo ip neigh flush dev eth0` |
| Tes siapa pemilik fisik (MAC Address) dari sebuah IP di jaringan lokal | `sudo arping -I eth0 -c 3 <IP-Target>` |
| Izinkan port SSH lalu aktifkan firewall `ufw` | `sudo ufw allow 22/tcp && sudo ufw enable` |
| Lihat daftar aturan `ufw` beserta nomor urutnya | `sudo ufw status numbered` |
| Hapus aturan `ufw` nomor tertentu | `sudo ufw delete <nomor>` |
| Lihat aturan firewall & jumlah paket di level kernel (`iptables`) | `sudo iptables -L -n -v` |
| Lihat aturan translasi alamat (`SNAT`/`DNAT`) di tabel NAT | `sudo iptables -t nat -L -n -v` |

---

## 12. Cek Pemahaman (Self-Check)

Uji pemahamanmu dengan menjawab pertanyaan berikut sebelum membuka kunci jawaban:

1. **Mengapa kita WAJIB menjalankan `sudo ufw allow 22/tcp` sebelum mengaktifkan firewall dengan kebijakan `default deny incoming` saat bekerja melalui SSH? Dan jika telanjur terkunci di VM OrbStack, bagaimana cara masuknya?**
2. **Apa perbedaan arah dan fungsi antara `SNAT / MASQUERADE` dengan `DNAT (Port Forwarding)`? Berikan masing-masing 1 contoh di lingkungan Kubernetes!**
3. **Apa itu *Gratuitous ARP*, dan mengapa saat Node tempat berjalannya IP LoadBalancer MetalLB tiba-tiba mati, bisa terjadi jeda gangguan (*downtime*) singkat selama beberapa detik meskipun Pod aplikasi di Node lain sehat?**
4. **Mengapa saat seorang pemula mencoba `curl` ke port yang ditutup dengan aksi firewall `DROP` rasanya sangat berbeda dengan port yang ditutup dengan aksi `REJECT`?**
5. **Dalam urutan 7 langkah troubleshooting jaringan, mengapa kita mengecek ARP (`ip neigh`) pada koneksi satu subnet sebelum menyalahkan aplikasi?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **Alasan wajib `ufw allow 22/tcp` sebelum `enable` & cara penyelamatan di OrbStack:**
   - SSH berjalan di atas port `22/tcp`. Jika kita mengaktifkan firewall dengan kebijakan `default deny incoming` tanpa mengizinkan port `22/tcp` terlebih dahulu, firewall akan memblokir koneksi SSH yang baru maupun memutus akses pengelolaan jarak jauh kita.
   - Di lingkungan OrbStack, jika telanjur terkunci dari SSH, kita masih bisa masuk melalui jalur konsol hypervisor langsung dari terminal Mac dengan perintah **`orb shell lab01`** (karena `orb shell` tidak melewati port jaringan TCP 22), lalu memperbaiki aturan `ufw`.

2. **Perbedaan `SNAT / MASQUERADE` vs `DNAT`:**
   - **`SNAT / MASQUERADE` (Arah Keluar / *Outbound*):** Mengganti alamat **IP Pengirim (*Source IP*)** privat menjadi IP milik Node/Router saat paket keluar menuju internet. *Contoh di Kubernetes:* Saat Pod (`10.42.0.15`) mengunduh data dari internet, Node melakukan `MASQUERADE` agar paket terlihat berasal dari IP Node.
   - **`DNAT` / Port Forwarding (Arah Masuk / *Inbound*):** Mengganti alamat **IP:Port Tujuan (*Destination*)** saat paket masuk dari luar agar dibelokkan ke server/container di dalam. *Contoh di Kubernetes:* Saat trafik masuk ke `NodeIP:30080` (*NodePort*) atau `ClusterIP`, komponen `kube-proxy` melakukan `DNAT` ke IP:Port Pod yang sebenarnya.

3. **Peran *Gratuitous ARP* & jeda *failover* di MetalLB L2 Mode:**
   - **Gratuitous ARP** adalah paket pengumuman ARP broadcast yang dikirim oleh Node baru tanpa menunggu ditanya, untuk memberi tahu seluruh jaringan bahwa alamat Virtual IP (VIP) MetalLB kini telah berpindah ke MAC Address milik Node baru tersebut.
   - Jeda gangguan singkat bisa terjadi karena pertama MetalLB butuh beberapa detik untuk mendeteksi Node lama sudah mati (*heartbeat timeout*), dan kedua beberapa perangkat/komputer di jaringan mungkin masih menyimpan catatan MAC Address Node lama di dalam **ARP Cache** mereka sampai cache tersebut diperbarui oleh *Gratuitous ARP* atau kedaluwarsa.

4. **Perbedaan gejala `DROP` vs `REJECT`:**
   - **`REJECT`** langsung membalas pemanggil dengan paket penolakan (`TCP RST`), sehingga `curl` langsung berhenti detik itu juga dengan pesan **`Connection refused`**.
   - **`DROP`** membuang paket secara diam-diam tanpa mengirim balasan apa pun, sehingga `curl` terus menunggu di status `SYN-SENT` sampai batas waktu habis dan menampilkan **`Connection timed out`**.

5. **Mengapa mengecek ARP penting sebelum menyalahkan aplikasi:**
   - Karena jika di level Layer 2 status ARP target adalah **`INCOMPLETE` / `FAILED`**, berarti komputer kita bahkan belum menemukan alamat fisik (MAC Address) mesin tujuan di jaringan lokal — sehingga **tidak ada satu pun paket TCP atau HTTP yang pernah sampai ke mesin tujuan**, apalagi sampai ke aplikasinya!

</details>

---

## 13. Langkah Selanjutnya

Selamat! Kamu telah menyelesaikan seluruh materi teori di **Modul 0.2 — Networking untuk SRE** ([01-dns-cidr](01-dns-cidr.md), [02-tcp-udp-tls](02-tcp-udp-tls.md), [03-http-proxy](03-http-proxy.md), dan [04-firewall-nat-arp](04-firewall-nat-arp.md)).

Sekarang saatnya mengubah pengetahuan ini menjadi *muscle memory* di terminal melalui rangkaian laboratorium dan evaluasi:
1. **[LAB-01: Reverse Proxy Caddy + TLS Otomatis](lab/LAB-01-reverse-proxy.md)** — Membangun reverse proxy multi-domain dengan HTTPS di VM `lab01`.
2. **[LAB-02: DNS Lokal dengan dnsmasq](lab/LAB-02-dns-lokal.md)** — Membangun server DNS internal untuk meresolusi domain `*.lab.local`.
3. **[LAB-03: Trace Masalah Koneksi (Troubleshooting Sistematis)](lab/LAB-03-trace-koneksi.md)** — Mempraktikkan kerangka **7-Langkah Troubleshooting** untuk memecahkan 3 skenario insiden jaringan nyata.
4. **[Latihan Mandiri](evaluasi/latihan.md)** dan **[Kuis & Jawaban Modul 0.2](evaluasi/kuis-dan-jawaban.md)** — Menguji kesiapanmu sebelum melangkah ke modul berikutnya!