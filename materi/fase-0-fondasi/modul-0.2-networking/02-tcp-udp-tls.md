# 02 — TCP, UDP, dan TLS (Transport & Enkripsi Jaringan)

> **Ringkasan:** Memahami bagaimana data diangkut melewati jaringan menggunakan protokol **TCP** (andal & berurutan lewat *Three-Way Handshake*) atau **UDP** (cepat tanpa jabat tangan), membaca status koneksi (`ESTABLISHED`, `TIME-WAIT`, `CLOSE-WAIT`), serta memahami cara kerja enkripsi **TLS/HTTPS**, sertifikat, **Terminasi TLS** di Reverse Proxy/Ingress, dan menginspeksi sertifikat dengan **`openssl`**.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Menjelaskan perbedaan cara kerja **TCP** dan **UDP** beserta alasan pemilihan masing-masing protokol di dunia nyata.
- Membedah proses **TCP Three-Way Handshake (`SYN`, `SYN-ACK`, `ACK`)** dan mengamatinya secara langsung menggunakan `tcpdump`.
- Mendiagnosa kesehatan koneksi server dari status TCP (`LISTEN`, `SYN-SENT`, `ESTABLISHED`, `TIME-WAIT`, `CLOSE-WAIT`).
- Memahami konsep **Port** (*Privileged Port* `< 1024` vs *Ephemeral Port*) dan alasan mengapa `127.0.0.1` berbeda dengan `0.0.0.0` di Kubernetes/VM.
- Menjelaskan cara kerja **TLS (HTTPS)**: peran **Sertifikat**, **CA (*Certificate Authority*)**, **Private Key**, dan **SNI**.
- Membedakan 3 pola **Terminasi TLS** (Server Langsung, Reverse Proxy/Ingress Termination, dan End-to-End / **mTLS**).
- Memeriksa masa berlaku dan identitas sertifikat TLS menggunakan **`openssl`**. 

---

## 1. Mindset Dasar: Dari Alamat Gedung ke Cara Pengiriman Paket

Di [Topik 01 (DNS & CIDR)](01-dns-cidr.md), kita sudah belajar cara menemukan alamat gedung tujuan menggunakan **DNS** dan **IP Address**.
Setelah alamat gedung ditemukan, muncul dua pertanyaan baru:
1. **Bagaimana cara kurir mengantarkan paket data ke pintu kamar (*Port*) tujuan?** Apakah harus menelepon dulu dan meminta tanda tangan penerima (**TCP**), atau langsung lempar ke kotak surat supaya cepat (**UDP**)?
2. **Bagaimana agar isi paket tidak bisa dibuka dan dibaca oleh penyadap di jalan raya?** Kita harus membungkusnya di dalam brankas terenkripsi (**TLS**)!

---

## 2. TCP vs UDP — Dua Gaya Kurir Pengantar Data

Di lapisan transportasi jaringan (*Transport Layer*), ada dua protokol utama yang mengangkut data aplikasimu: **TCP** dan **UDP**.

### A. TCP (*Transmission Control Protocol*) — *Si Kurir Resmi Ber-Tanda Terima*
> 📞 **Analogi Panggilan Telepon:**
> Sebelum kamu mulai bercerita di telepon, kamu harus menyapa dulu: *"Halo, suara saya terdengar?"* — *"Ya, terdengar jelas, silakan!"*. Selama mengobrol, jika ada kalimat yang putus karena sinyal jelek, lawan bicaramu akan berkata *"Maaf, barusan putus, bisa diulangi?"*.

Itulah **TCP**. TCP bersifat **Connection-Oriented** (wajib membuat sambungan resmi dulu) dan menjamin:
1. **Andal (*Reliable*):** Setiap paket data yang dikirim wajib dibalas dengan tanda terima (**`ACK` / Acknowledgment**). Jika tanda terima tidak datang, TCP otomatis mengirim ulang paket tersebut (**Retransmission**).
2. **Berurutan (*Ordered*):** Jika paket halaman 1, 2, 3 datang terbalik menjadi 3, 1, 2 di jalan, TCP akan menyusunnya kembali menjadi 1, 2, 3 sebelum diserahkan ke aplikasi.

### B. UDP (*User Datagram Protocol*) — *Si Pengirim Kilat Tanpa Basa-Basi*
> 📢 **Analogi Teriak Pakai Megafon / Mengirim Kartu Pos:**
> Kamu langsung berteriak menyampaikan pesan tanpa menyapa dulu, tidak meminta tanda terima, dan tidak peduli apakah ada kata yang tertiup angin di jalan.

Mengapa ada orang yang mau memakai **UDP** kalau paketnya bisa hilang tanpa dikirim ulang?
Jawabannya: **KARENA SANGAT CEPAT dan hemat beban!**
- Pada **Live Video Call (Zoom) / Game Online:** Jika gambar video detik ke-5 hilang di jalan, kita tidak butuh gambar detik ke-5 itu dikirim ulang pada detik ke-8 (karena percakapannya sudah lewat!). Lebih baik lanjutkan menampilkan detik ke-8 secara *real-time*.
- Pada **DNS Query (Port 53):** Pertanyaan DNS (*"Berapa IP google.com?"*) hanya berukuran 1 paket super kecil. Kalau harus jabat tangan TCP 3 kali dulu baru bertanya, waktunya habis di jalan! Dengan UDP, langsung tanya 1 paket dan langsung dijawab 1 paket.

### Tabel Perbandingan TCP vs UDP

| Aspek | **TCP** | **UDP** |
|---|---|---|
| **Persiapan Koneksi** | Wajib jabat tangan dulu (*3-Way Handshake*) | Langsung kirim (*Connectionless*) |
| **Jaminan Sampai & Urutan** | **Ya** (ada *ACK* dan kirim ulang otomatis) | **Tidak** (paket hilang tidak dikirim ulang) |
| **Kecepatan & Beban (*Overhead*)** | Lebih berat & sedikit lebih lambat | Sangat ringan & sangat cepat |
| **Contoh Penggunaan di Server** | **HTTP/HTTPS** (Web), **SSH**, **Database** (PostgreSQL/MySQL), Transfer File (`scp`/`rsync`) | **DNS** (Port 53), **Video/Voice Streaming**, **VPN WireGuard**, pengiriman log/metrik ringan |

> 💡 **Catatan Penting SRE:**
> Saat men-debug masalah *"Kenapa query DNS kadang-kadang timeout 5 detik?"*, ingatlah bahwa DNS secara default memakai **UDP**. Jika jaringan sedang padat dan 1 paket UDP pertanyaan DNS terbuang (*dropped*), OS tidak tahu paket itu hilang sampai batas waktu tunggu (*timeout*) habis, baru ia mencoba bertanya lagi!

---

## 3. Membedah TCP: *Three-Way Handshake* & Status Koneksi

### A. Proses Jabat Tangan Tiga Langkah (*Three-Way Handshake*)
Setiap kali aplikasi menggunakan TCP (misalnya saat kamu menjalankan `curl http://example.com`), komputer kamu (**Client**) dan **Server** wajib melakukan ritual 3 langkah sebelum data pertama boleh dikirim:

```text
   [ CLIENT (Laptop/VM) ]                                  [ SERVER (Port 80) ]
             │                                                       │
             │  1. SYN      ("Halo Server, boleh buka koneksi?")     │
             │ ────────────────────────────────────────────────────► │
             │                                                       │
             │  2. SYN-ACK  ("Boleh! Saya siap, kamu siap?")         │
             │ ◄──────────────────────────────────────────────────── │
             │                                                       │
             │  3. ACK      ("Siap! Mari kita mulai kirim data.")    │
             │ ────────────────────────────────────────────────────► │
             │                                                       │
             ╞══════════ KONEKSI RESMI TERBUKA (ESTABLISHED) ════════╡
             │                                                       │
             │  4. Kirim Data ("GET /index.html HTTP/1.1")           │
             │ ────────────────────────────────────────────────────► │
             │  5. ACK + Balasan HTML                                │
             │ ◄──────────────────────────────────────────────────── │
```

- **`SYN` (*Synchronize*):** Client mengetuk pintu server untuk memulai koneksi.
- **`SYN-ACK` (*Synchronize-Acknowledge*):** Server membuka pintu dan menyetujui permintaan Client.
- **`ACK` (*Acknowledge*):** Client mengonfirmasi bahwa sambungan sudah siap dari kedua arah.

```bash
# Kamu bisa melihat jabat tangan ini secara nyata dengan tcpdump:
sudo tcpdump -i any -n port 80 -c 10
# Di tcpdump:
#   Flags [S]   = SYN
#   Flags [S.]  = SYN-ACK (titik '.' artinya ACK)
#   Flags [.]   = ACK
```

---

### B. Membaca Status (*State*) Koneksi TCP saat Troubleshooting
Saat kamu menjalankan `ss -tan`, kolom `State` menunjukkan kondisi setiap sambungan TCP. Seorang SRE wajib memahami 5 status ini untuk mendiagnosa masalah jaringan:

| Status TCP (`ss -tan`) | Artinya dalam Bahasa Manusia | Apa Artinya Jika Jumlahnya Menumpuk Banyak? |
|---|---|---|
| **`LISTEN`** | Aplikasi server sedang buka pintu menunggu tamu datang. | Normal (sesuai jumlah layanan yang aktif). |
| **`SYN-SENT`** | Client sudah mengirim paket `SYN` (ketuk pintu), tapi **belum dapat balasan `SYN-ACK`**. | 🚨 **Tanda Masalah Jaringan/Firewall!** Paket menuju server tujuan diblokir oleh Firewall di tengah jalan, atau IP tujuan mati. |
| **`ESTABLISHED`** | Jabat tangan selesai; koneksi sedang aktif mengalirkan data. | Menunjukkan jumlah koneksi pengguna yang sedang aktif saat ini. |
| **`TIME-WAIT`** | Koneksi sudah selesai ditutup, tapi sistem operasi sengaja menahan port selama ~60 detik untuk memastikan tidak ada sisa paket nyasar di jalan. | ⚠️ Jika jumlahnya puluhan ribu: aplikasi terlalu sering **buka-tutup koneksi baru** untuk setiap request. **Solusi:** Aktifkan fitur **HTTP Keep-Alive / Connection Pooling** agar 1 koneksi dipakai berulang kali! |
| **`CLOSE-WAIT`** | Pihak seberang sudah meminta tutup koneksi, tapi **aplikasi di server kita belum memanggil perintah `.close()`**. | 🚨 **Tanda Bug di Kode Aplikasi (Connection Leak)!** Aplikasi lupa menutup socket. Lama-kelamaan server akan kehabisan port/file descriptor dan macet! |

```bash
# Perintah cepat SRE memeriksa kesehatan status koneksi TCP:
ss -s                                   # Lihat ringkasan jumlah total koneksi per state
ss -tan state established               # Lihat semua koneksi yang sedang aktif
ss -tan state time-wait | wc -l         # Hitung ada berapa koneksi TIME-WAIT
ss -tanp state close-wait               # Lacak aplikasi mana yang bocor (CLOSE-WAIT)!
```

---

## 4. Mengenal Port: Nomor Pintu & Kaitannya dengan Kubernetes

Dalam satu alamat IP, terdapat **65.535 Port TCP** dan **65.535 Port UDP** yang dibagi menjadi 3 kelompok:

1. **Port `1` – `1023` (*Well-Known / Privileged Ports*):**
   - Port resmi untuk layanan standar. Demi keamanan di Linux, **hanya user `root` (atau program dengan izin khusus `CAP_NET_BIND_SERVICE`) yang boleh membuka port di bawah 1024!**
   - Itulah sebabnya saat kita membuat aplikasi sendiri (Python/Node.js/Go) yang dijalankan sebagai user biasa non-root, kita selalu memakai port di atas 1024 (seperti `3000`, `8000`, atau `8080`).
2. **Port `1024` – `49151` (*Registered Ports*):**
   - Dipakai oleh aplikasi populer seperti MySQL (`3306`), PostgreSQL (`5432`), Kubernetes API (`6443`), Prometheus (`9090`), dan Grafana (`3000`).
3. **Port `49152` – `65535` (*Ephemeral / Dynamic Ports*):**
   - **Port Pintu Keluar Sementara.** Saat laptopmu menghubungi `server:443`, laptopmu juga butuh satu nomor pintu sementara di sisinya sendiri (misalnya `laptop:54321` $\rightarrow$ `server:443`). Begitu koneksi selesai, port `54321` dikembalikan lagi ke sistem.

### Daftar Port Wajib Hafal untuk SRE & Kubernetes

| Nomor Port | Protokol | Layanan / Komponen |
|:---:|:---:|---|
| **`22`** | TCP | **SSH** (Kendali jarak jauh & jalur Ansible) |
| **`53`** | **UDP & TCP** | **DNS** (UDP untuk query biasa; TCP untuk jawaban besar / transfer zona) |
| **`80`** / **`443`** | TCP | **HTTP** (Web tanpa enkripsi) / **HTTPS** (Web terenkripsi TLS) |
| **`3306`** / **`5432`** | TCP | Database **MySQL** / **PostgreSQL** |
| **`6379`** | TCP | **Redis** (In-memory cache) |
| **`6443`** | TCP | **Kubernetes API Server** (Jantung kendali cluster k3s/Kubernetes) |
| **`10250`** | TCP | **Kubelet API** (Agen pekerja di setiap node Kubernetes) |

### Pengingat Kritis untuk Kubernetes: `0.0.0.0` vs `127.0.0.1`
Di Kubernetes nanti, setiap Pod memiliki alamat IP sendiri dan saling berbicara lewat TCP. Jika Pod A gagal menghubungi Pod B di port `8080`, urutan ceknya selalu:
1. Cek di dalam Pod B dengan `ss -tlnp`: Apakah port `8080` benar-benar `LISTEN`?
2. Pastikan aplikasi di Pod B mendengarkan di **`0.0.0.0:8080`** (semua pintu), **BUKAN `127.0.0.1:8080`** (hanya dirinya sendiri)!
3. Cek apakah ada Firewall / *Kubernetes NetworkPolicy* yang memblokir jalur antar-Pod.

---

## 5. TLS (*Transport Layer Security*) — Brankas Pengaman di Atas TCP

### Mengapa TCP Biasa Saja Tidak Cukup?
TCP memang menjamin paket data sampai dengan utuh dan berurutan, tetapi **TCP mengirim semua data dalam bentuk teks telanjang (*Plaintext*)!**
Jika kamu login ke sebuah website menggunakan **`http://`** biasa (Port 80), siapa pun yang berada di jaringan Wi-Fi yang sama atau mengelola router di tengah jalan bisa menyadap paket TCP menggunakan `tcpdump` dan langsung membaca username serta password kamu!

Untuk mengatasi itu, diciptakanlah **TLS (*Transport Layer Security*)** — generasi modern dari protokol yang dulu dikenal dengan nama **SSL**.
> **Rumus Sederhana:** **`HTTPS` = Protokol `HTTP` yang dibungkus di dalam pipa aman `TLS` di atas `TCP`.**

---

### Tiga Fungsi Utama TLS & Analogi KTP Resmi
TLS tidak hanya mengacak data (**Enkripsi**), tetapi juga membuktikan bahwa server yang kamu kunjungi adalah **asli dan bukan server palsu buatan penipu** (**Otentikasi**).

Bagaimana browser atau `curl` tahu bahwa website `klikbca.com` yang kamu buka benar-benar milik bank asli? Melalui 3 komponen berikut:

```text
  ┌─────────────────────────────────────────────────────────────────────┐
  │ 🪪 SERTIFIKAT TLS (Public Certificate — cert.pem / fullchain.pem)   │
  │   • Nama Pemilik (Subject / SAN) : *.example.com, example.com       │
  │   • Masa Berlaku (Validity)      : 15 Jan 2026 s.d. 15 Jan 2027     │
  │   • Kunci Publik (Public Key)    : [Data Kriptografi Publik]        │
  │   • Tanda Tangan Resmi (Issuer)  : Ditandatangani oleh Let's Encrypt│
  └─────────────────────────────────────────────────────────────────────┘
```

1. **Sertifikat TLS (*Certificate*):** Ibarat **KTP Digital** milik server. Di dalamnya tertulis nama domain pemiliknya, kapan masa berlakunya habis, dan *Public Key*. Sertifikat ini dikirimkan ke setiap pengunjung yang datang.
2. **CA (*Certificate Authority*):** Ibarat **Kantor Dukcapil / Instansi Pemerintah Resmi** yang menerbitkan dan membubuhkan stempel tanda tangan digital pada KTP tersebut (contoh CA dunia: *Let's Encrypt*, *DigiCert*, *Cloudflare*).
   - Di dalam sistem operasi Linux (di folder `/etc/ssl/certs/`) dan di dalam browser sudah tertanam daftar stempel CA resmi dunia (**Root CA Store**).
   - Jika kamu membuat sertifikat tanda tangan sendiri (**Self-Signed Certificate**) tanpa stempel CA resmi, maka `curl` dan browser akan langsung berteriak: ***"Peringatan! KTP server ini tidak ditandatangani oleh instansi resmi yang saya kenal!"***
3. **Private Key (`privkey.pem`):** Kunci rahasia yang disimpan rapat di dalam server (tidak pernah dikirim ke luar). Hanya server yang memegang *Private Key* pasangan dari sertifikat tersebut yang bisa membuka pesan rahasia dari pengunjung.

---

### Alur Jabat Tangan TLS (*TLS Handshake*) & Peran `SNI`
Tepat setelah jabat tangan TCP (*3-Way Handshake*) selesai, Client dan Server langsung melakukan jabat tangan kedua, yaitu **TLS Handshake**:

```text
   [ CLIENT (Browser / curl) ]                            [ SERVER (Port 443) ]
             │                                                      │
             │ ── 1. ClientHello + SNI ("Saya mau ke example.com") ►│
             │                                                      │
             │ ◄─ 2. ServerHello + Sertifikat KTP (example.com) ─── │
             │                                                      │
     (Client memeriksa KTP:                                         │
      ✓ Apakah nama domain cocok?                                   │
      ✓ Apakah belum kadaluarsa?                                    │
      ✓ Apakah stempel CA-nya resmi?)                               │
             │                                                      │
             │ ── 3. Key Exchange (Sepakati Kunci Rahasia Sesi) ───►│
             │ ◄─ 4. Finished ("Mulai sekarang kita bicara sandi!") │
             │                                                      │
             ╞═══════ PIPA TERENKRIPSI AMAN (ENCRYPTED TLS) ════════╡
```

> 🔍 **Apa Itu `SNI` (*Server Name Indication*) pada Langkah 1?**
> Di dunia modern, **satu alamat IP server bisa menampung puluhan website berbeda** (misal `toko.com`, `blog.com`, dan `grafana.kantor.com` semuanya berada di IP `10.0.0.50`).
> Ketika Client baru mengetuk pintu `443`, server bingung: *"Tamu ini mau ke `toko.com` atau `blog.com`? KTP mana yang harus saya tunjukkan?"*
> Dengan fitur **SNI**, Client langsung menyebutkan nama domain yang ia tuju pada sapaan pertama (`ClientHello`), sehingga server bisa menyerahkan sertifikat TLS yang tepat!

---

### Membedah Sertifikat السitus Nyata dengan `openssl`
Sebagai SRE, kamu wajib bisa menginspeksi sertifikat server langsung dari terminal:

```bash
# Periksa Siapa Pemilik (subject), Siapa CA Penandatangannya (issuer), dan Tanggal Kadaluarsanya (dates):
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```
*Contoh Output:*
```text
subject=CN = www.example.org
issuer=C = US, O = DigiCert Inc, CN = DigiCert Global G2 TLS RSA SHA256 2020 CA1
notBefore=Jan 15 00:00:00 2026 GMT
notAfter=Jan 15 23:59:59 2027 GMT
```
- Perhatikan opsi **`-servername example.com`**: Opsi ini mengirimkan informasi **SNI** tadi! Tanpa `-servername`, server yang menampung banyak website mungkin akan memberikan sertifikat default yang salah.
- Perhatikan **`notAfter`**: Jika tanggal hari ini sudah melewati tanggal `notAfter`, maka website akan mati tidak bisa diakses oleh seluruh pengguna!

---

## 6. Tiga Pola Terminasi TLS (*TLS Termination*) di Production

**Terminasi TLS (*TLS Termination*)** artinya: **"Di titik mana bungkus enkripsi HTTPS dibuka dan diubah kembali menjadi HTTP biasa?"**
Di arsitektur server dan Kubernetes, ada 3 pola desain yang wajib kamu pahami:

```text
Pola A — Server Aplikasi Langsung:
  [ Client ] ═══════ TLS (HTTPS :443) ═══════► [ Aplikasi Web ]

Pola B — Reverse Proxy / Ingress Termination (PALING UMUM DI PRODUCTION!):
  [ Client ] ═══ TLS (HTTPS :443) ═══► [ Reverse Proxy / Ingress ] ─── HTTP (:8080) ───► [ Pod App ]
                                       (Caddy / Nginx / Traefik)                         (Di jaringan
                                       (Sertifikat disimpan di sini!)                     internal)

Pola C — End-to-End TLS / mTLS (Keamanan Tingkat Tinggi):
  [ Client ] ═══ TLS ═══► [ Proxy / LoadBalancer ] ═══ TLS / mTLS ═══► [ Pod App ]
```

| Pola Arsitektur | Kelebihan | Kekurangan | Kapan Dipakai? |
|---|---|---|---|
| **A. Langsung di Aplikasi** | Arsitektur paling sederhana (hanya 1 komponen). | Kode aplikasi harus repot mengurus file sertifikat & perpanjangannya sendiri. | Server tunggal kecil / alat internal sederhana. |
| **B. Terminasi di Reverse Proxy / Ingress** | **Sangat praktis!** Ratusan aplikasi di belakang tidak perlu pusing soal HTTPS. Cukup 1 pintu depan (Reverse Proxy/Ingress) yang mengurus dan memperpanjang semua sertifikat otomatis! | Lalu lintas dari Proxy ke Pod di dalam jaringan internal berupa HTTP biasa (jaringan internal harus aman). | **Standar utama di Production On-Prem & Kubernetes** (akan kamu praktikkan di [LAB-01](lab/LAB-01-reverse-proxy.md) dengan Caddy dan di Fase 2 dengan Ingress). |
| **C. End-to-End / mTLS** | Terenkripsi penuh sampai ke titik paling ujung. | Jauh lebih rumit; setiap pod/komponen harus punya sertifikat masing-masing. | Lingkungan perbankan ketat, *Service Mesh*, dan komunikasi internal antar-komponen Kubernetes. |

### Sekilas tentang **mTLS (*Mutual TLS*)** di Dalam Kubernetes
Pada HTTPS biasa (ketika kamu membuka website), **hanya Server yang menunjukkan KTP Sertifikat ke Client**.
Tetapi di dalam mesin internal cluster Kubernetes (misalnya saat agen pekerja `kubelet` di Node berbicara dengan otak `kube-apiserver`), keduanya menggunakan **mTLS (*Mutual TLS*)**:
- Server menunjukkan sertifikat ke Client, **DAN Client juga wajib menunjukkan sertifikat ke Server!**
- Itulah mengapa di dalam instalasi Kubernetes (`k3s`) terdapat banyak file sertifikat internal. Jika sertifikat internal cluster kedaluwarsa, komponen Kubernetes tidak lagi saling percaya dan cluster akan berhenti bekerja (skenario ini akan kamu hadapi dan perbaiki di *Fase 8 & Fase 9 Game Day*!).

---

## 7. Pola Kegagalan (Failure Modes) di Production & Cara Diagnosanya

Berikut adalah masalah nyata terkait TCP, UDP, dan TLS yang sering memicu insiden di lingkungan production:

| Gejala / Pesan Error | Penyebab Utama | Cara Diagnosa & Solusi SRE |
|---|---|---|
| **`SSL certificate problem: certificate has expired`** | Masa berlaku sertifikat TLS (`notAfter`) sudah lewat. | Cek tanggalnya dengan `openssl s_client ... \| openssl x509 -noout -dates`. Segera perbarui sertifikat (misal reload Caddy / Cert-Manager). |
| **`SSL: no alternative certificate subject name matches target host name`** | Nama domain yang kamu akses di URL **tidak cocok** dengan nama domain yang tertulis di dalam KTP sertifikat (misal kamu mengakses pakai angka IP `https://10.0.0.50`, padahal sertifikatnya dibuat untuk `https://app.lab.local`). | Akses menggunakan nama domain yang sesuai, atau pastikan `-servername` (SNI) dan DNS sudah mengarah dengan benar. |
| **`SSL certificate problem: self-signed certificate` / `unable to get local issuer certificate`** | Server menggunakan sertifikat buatan sendiri (*Self-Signed*) atau rantai sertifikat perantara (*Intermediate CA*) lupa disertakan di server. | Untuk server publik: pasang file `fullchain.pem` (bukan hanya `cert.pem`). Untuk CA internal kantor: daftarkan CA ke `/usr/local/share/ca-certificates/` lalu jalankan `sudo update-ca-certificates`. |
| Koneksi menggantung di status **`SYN-SENT`** lalu berakhir **`Connection timed out`** | Paket pembuka jabat tangan TCP (`SYN`) dikirim, tapi tidak pernah dibalas `SYN-ACK`. | Jalur jaringan putus atau port tersebut diblokir secara *DROP* oleh **Firewall** (`ufw` / router). Cek firewall di sisi server tujuan. |
| Server melambat dan ribuan koneksi berstatus **`CLOSE-WAIT`** | Aplikasi di server lupa menutup koneksi socket setelah selesai dipakai (*Connection Leak*). | Cari PID aplikasinya dengan `sudo ss -tanp state close-wait`, lakukan *restart* layanan sebagai mitigasi darurat, dan laporkan bug ke tim pengembang aplikasi. |

---

## 8. Latihan Praktik Mandiri (10 Menit di Terminal VM OrbStack)

Mari kita buktikan konsep *3-Way Handshake*, perbedaan UDP vs TCP pada DNS, serta inspeksi sertifikat TLS langsung di terminal VM `lab01` kamu:

```bash
# 1. Lihat ringkasan statistik soket TCP dan UDP di VM kamu saat ini:
ss -s

# 2. Lihat semua port TCP (-t) dan UDP (-u) yang sedang membuka pintu (LISTEN):
sudo ss -tulpn

# 3. Tangkap proses TCP Three-Way Handshake secara langsung!
#    Jalankan tcpdump di background untuk menangkap 6 paket pertama ke port 80,
#    lalu panggil curl ke example.com:
sudo tcpdump -i any -n -c 6 'tcp port 80' &
TCPDUMP_PID=$!
sleep 1
curl -s http://example.com >/dev/null
wait $TCPDUMP_PID
# Perhatikan urutan 3 baris pertama dari output tcpdump di atas:
#   Flags [S]   -> SYN (Langkah 1)
#   Flags [S.]  -> SYN-ACK (Langkah 2)
#   Flags [.]   -> ACK (Langkah 3 - Handshake selesai!)

# 4. Bandingkan query DNS menggunakan UDP (default) vs memaksa lewat jalur TCP (+tcp):
dig @1.1.1.1 example.com | grep -E "SERVER:|flags:"
dig +tcp @1.1.1.1 example.com | grep -E "SERVER:|flags:"
# Perhatikan di bagian SERVER paling bawah tertulis (UDP) vs (TCP)!

# 5. Periksa KTP Sertifikat TLS milik gitlab.com menggunakan openssl:
echo | openssl s_client -connect gitlab.com:443 -servername gitlab.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# 6. Buat sertifikat Self-Signed sederhana buatan sendiri dan intip isinya:
mkdir -p ~/latihan-tls && cd ~/latihan-tls
openssl req -x509 -newkey rsa:2048 -nodes -keyout server.key -out server.crt \
  -days 365 -subj "/CN=app.lab.local" 2>/dev/null

#    Periksa isi file sertifikat yang baru kita buat di disk:
openssl x509 -in server.crt -noout -subject -issuer -dates

# 7. Bersihkan folder latihan
cd ~ && rm -rf ~/latihan-tls
```

**Acceptance Criteria:**
- [ ] Bisa menunjukkan 3 paket pertama (`Flags [S]`, `Flags [S.]`, `Flags [.]`) dari hasil tangkapan `tcpdump`.
- [ ] Bisa membedakan output `ss -tlnp` (TCP) dan `ss -ulnp` (UDP).
- [ ] Berhasil membaca `subject`, `issuer`, dan `notAfter` dari sertifikat `gitlab.com` maupun file `server.crt` lokal.
- [ ] Folder `~/latihan-tls` sudah dibersihkan kembali setelah latihan.

---

## 9. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Lihat ringkasan jumlah koneksi TCP/UDP per status | `ss -s` |
| Lihat port TCP & UDP yang sedang *Listening* beserta prosesnya | `sudo ss -tulpn` |
| Lihat koneksi TCP yang sedang aktif (*Established*) | `ss -tan state established` |
| Hitung jumlah koneksi `TIME-WAIT` / cari `CLOSE-WAIT` | `ss -tan state time-wait \| wc -l` / `sudo ss -tanp state close-wait` |
| Tangkap paket jabat tangan TCP di port tertentu | `sudo tcpdump -i any -n port 80` |
| Paksa query DNS menggunakan jalur TCP (bukan UDP) | `dig +tcp @1.1.1.1 nama.domain` |
| Cek pemilik, penerbit (CA), dan kadaluarsa sertifikat TLS server | `echo \| openssl s_client -connect host:443 -servername host 2>/dev/null \| openssl x509 -noout -subject -issuer -dates` |
| Baca isi file sertifikat `.crt` / `.pem` yang ada di disk | `openssl x509 -in file.crt -noout -text` |

---

## 10. Cek Pemahaman (Self-Check)

Uji pemahamanmu dengan menjawab pertanyaan berikut sebelum membuka kunci jawaban:

1. **Mengapa di dalam cluster Kubernetes, aplikasi di Pod A yang mendengarkan di `0.0.0.0:8080` bisa diakses oleh Pod lain, sedangkan jika mendengarkan di `127.0.0.1:8080` tidak bisa diakses oleh Pod lain?**
2. **Apa perbedaan penyebab antara menumpuknya ribuan koneksi berstatus `TIME-WAIT` dibandingkan menumpuknya koneksi berstatus `CLOSE-WAIT` di sebuah server?**
3. **Mengapa query DNS kadang-kadang bisa mengalami *timeout* tanpa pengiriman ulang otomatis, padahal protokol TCP terkenal selalu mengirim ulang paket yang hilang?**
4. **Apa yang dimaksud dengan *TLS Termination* di Reverse Proxy (atau Kubernetes Ingress), dan mengapa pola ini paling disukai di lingkungan production on-prem?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **`0.0.0.0:8080` vs `127.0.0.1:8080` antar-Pod:**
   - `127.0.0.1` adalah antarmuka *loopback* internal milik Pod itu sendiri. Jika aplikasi hanya melakukan *bind* ke `127.0.0.1:8080`, maka paket dari luar Pod (termasuk dari Pod tetangga atau Service Kubernetes) akan ditolak.
   - Sebaliknya, `0.0.0.0:8080` menginstruksikan aplikasi untuk mendengarkan pada **seluruh kartu jaringan** yang dimiliki Pod tersebut, sehingga dapat menerima koneksi TCP dari Pod lain.

2. **`TIME-WAIT` vs `CLOSE-WAIT`:**
   - Banyak **`TIME-WAIT`** terjadi karena koneksi dibuka dan ditutup secara sangat cepat (misalnya ribuan request HTTP pendek tanpa memakai *Keep-Alive / Connection Pooling*). Ini adalah perilaku normal protokol TCP untuk membersihkan sisa paket di jaringan.
   - Banyak **`CLOSE-WAIT`** terjadi karena pihak lawan sudah menutup koneksi, tetapi **kode aplikasi di server kita lupa memanggil fungsi `.close()`** pada socket tersebut (*bug connection leak*).

3. **Mengapa DNS bisa kehilangan paket tanpa kirim ulang otomatis?**
   - Karena secara bawaan (default), query DNS menggunakan protokol **UDP (Port 53)**, bukan TCP. UDP bersifat *connectionless* dan tidak memiliki mekanisme *ACK* maupun pengiriman ulang otomatis di level transport; jika paket UDP jatuh di jaringan yang padat, aplikasi harus menunggu hingga *timeout* habis sebelum mencoba bertanya ulang.

4. **TLS Termination di Reverse Proxy / Ingress:**
   - Artinya proses pembukaan enkripsi HTTPS (TLS) dilakukan satu pintu di gerbang depan (**Reverse Proxy** seperti Caddy, Nginx, atau Ingress Controller), lalu diteruskan sebagai HTTP biasa ke aplikasi di belakangnya.
   - Pola ini disukai karena **memusatkan manajemen sertifikat di satu titik**: kita tidak perlu memasang dan memperpanjang sertifikat HTTPS di puluhan container/aplikasi satu per satu, serta meringankan beban CPU aplikasi backend.

</details>

---

## 11. Langkah Selanjutnya

Kamu sudah memahami bagaimana nama menjadi IP ([01-dns-cidr.md](01-dns-cidr.md)) dan bagaimana pipa koneksi TCP/TLS dibangun. Selanjutnya, di [03-http-proxy.md](03-http-proxy.md), kita akan melihat **isi pesan HTTP** yang mengalir di dalam pipa tersebut serta cara kerja **Reverse Proxy** di depan aplikasi!
