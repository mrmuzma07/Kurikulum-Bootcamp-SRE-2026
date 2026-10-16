# 03 — HTTP/HTTPS, Reverse Proxy & Caddy (Lapisan Aplikasi & Gerbang Lalu Lintas)

> **Ringkasan:** Memahami anatomi percakapan **HTTP** (*Request* & *Response*, *Method*, *Status Code*, dan *Header*), konsep **Idempotency** saat mengulang permintaan, perbedaan krusial kode error (`401` vs `403`, serta `502` vs `503` vs `504`), peran **Reverse Proxy** di depan aplikasi, serta cara mengkonfigurasi **Caddy** untuk *Host-Based* dan *Path-Based Routing* yang menjadi fondasi **Kubernetes Ingress**.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Membedah anatomi **HTTP Request** dan **HTTP Response** secara utuh menggunakan `curl -v`.
- Menjelaskan perbedaan fungsi **HTTP Method** (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) dan pentingnya sifat **Idempotent** saat terjadi *retry* otomatis.
- Mendiagnosa masalah aplikasi dan infrastruktur dari **HTTP Status Code** (`2xx`, `3xx`, `4xx`, `5xx`), khususnya membedakan **`401` vs `403`** serta **`502` vs `503` vs `504`**.
- Membaca dan memanipulasi **HTTP Header** penting bagi SRE (`Host`, `Content-Type`, `Authorization`, `X-Forwarded-For`, `X-Forwarded-Proto`).
- Menjelaskan perbedaan **Forward Proxy** vs **Reverse Proxy** dan alasan mengapa aplikasi production tidak pernah diekspos langsung ke internet tanpa Reverse Proxy.
- Mengkonfigurasi **Caddy** sebagai Reverse Proxy dengan **Host-Based Routing** dan **Path-Based Routing** (mental model langsung menuju **Kubernetes Ingress** di Fase 2).

---

## 1. Mindset Dasar: Dari Pipa Koneksi ke Bahasa Percakapan

Mari kita urutkan perjalanan paket data yang sudah kita pelajari sejauh ini:
1. Di **[Topik 01 (DNS & CIDR)](01-dns-cidr.md)**: Komputer menemukan alamat gedung tujuan (**IP Address**) dari nama domain.
2. Di **[Topik 02 (TCP, UDP & TLS)](02-tcp-udp-tls.md)**: Komputer membangun pipa sambungan telepon yang andal (**TCP**) dan membungkusnya dengan brankas anti-sadap (**TLS**).
3. Di **Topik 03 (HTTP & Reverse Proxy)** ini: Setelah pipa telepon tersambung, **bahasa apa yang diucapkan di dalam pipa tersebut?**

Bahasa standar yang digunakan oleh browser, aplikasi mobile, API antar-microservice, hingga alat monitoring seperti Prometheus untuk saling berbicara adalah **HTTP (*Hypertext Transfer Protocol*)**.

### Tiga Sifat Dasar HTTP yang Wajib Dipahami Pemula
1. **Berbasis Teks (*Text-Based*) & Mudah Dibaca Manusia:**
   Pada HTTP/1.1, pesan yang dikirim bukanlah kode biner misterius, melainkan teks biasa baris demi baris yang bisa langsung kamu baca di layar terminal.
2. **Pola Tanya-Jawab (*Request – Response*):**
   Server tidak akan pernah mengirim data duluan tanpa diminta. **Client** (pemanggil, misalnya browser atau `curl`) selalu mengirim **1 Permintaan (*Request*)**, lalu **Server** membalas dengan **1 Jawaban (*Response*)**.
3. **Tanpa Ingatan (*Stateless*):**
   Secara bawaan, setiap permintaan HTTP bersifat independen. Server tidak "mengingat" siapa kamu dari permintaan sebelumnya. Agar server tahu bahwa permintaan ke-2 berasal dari orang yang sudah login pada permintaan ke-1, Client wajib menyertakan tanda pengenal (seperti *Token* atau *Cookie*) di **setiap** permintaan!

---

## 2. Membedah Anatomi HTTP Request & Response

Mari kita ibaratkan HTTP seperti **memesan makanan lewat surat resmi di restoran**. Surat pesananmu adalah **HTTP Request**, dan nampan hidangan beserta struk balasannya adalah **HTTP Response**.

```text
   [ CLIENT (curl / Browser) ]                           [ SERVER (Aplikasi Web) ]
                │                                                    │
                │  1. HTTP REQUEST (Surat Pesanan)                   │
                │  ┌──────────────────────────────────────────────┐  │
                │  │ GET /api/health HTTP/1.1       ◄─ Start Line │  │
                │  │ Host: app.lab.local            ◄─ Headers    │  │
                │  │ User-Agent: curl/8.5.0                       │  │
                │  │ Accept: application/json                     │  │
                │  │                                ◄─ Baris Kosong  │
                │  │ (Body opsional untuk POST/PUT) ◄─ Body       │  │
                │  └──────────────────────────────────────────────┘  │
                │ ─────────────────────────────────────────────────► │
                │                                                    │
                │  2. HTTP RESPONSE (Struk & Hidangan Balasan)       │
                │  ┌──────────────────────────────────────────────┐  │
                │  │ HTTP/1.1 200 OK                ◄─ Status Line│  │
                │  │ Content-Type: application/json ◄─ Headers    │  │
                │  │ Content-Length: 15                           │  │
                │  │                                ◄─ Baris Kosong  │
                │  │ {"status":"ok"}                ◄─ Body       │  │
                │  └──────────────────────────────────────────────┘  │
                │ ◄───────────────────────────────────────────────── │
```

### A. Tiga Bagian dari HTTP Request (Surat Pesanan Client)
1. **Baris Pertama (*Request Line*):** `GET /api/health HTTP/1.1`
   Berisi 3 hal yang dipisahkan spasi:
   - **Method (`GET`):** Kata kerja / aksi apa yang ingin dilakukan? (Baca data, kirim data baru, atau hapus?).
   - **Path / URI (`/api/health`):** Ruangan atau alamat dokumen apa di dalam server yang diminta?
   - **Versi (`HTTP/1.1`):** Versi protokol HTTP yang digunakan.
2. **Deretan *Request Headers* (Amplop & Catatan Tambahan):**
   Pasangan `Nama-Header: Nilai` yang memberi informasi konteks ke server (misal: tujuan nama domain `Host`, identitas pemanggil `User-Agent`, atau kartu akses `Authorization`).
3. **Baris Kosong (`\r\n`) + *Request Body* (Isi Paket Data):**
   Wajib ada **1 baris kosong** sebagai pemisah antara Header dan Body! Untuk permintaan baca (`GET`), Body biasanya kosong. Untuk pengiriman formulir/data (`POST` atau `PUT`), Body berisi data (misalnya teks JSON `{"nama": "budi"}`).

### B. Tiga Bagian dari HTTP Response (Balasan Server)
1. **Baris Pertama (*Status Line*):** `HTTP/1.1 200 OK`
   Berisi versi HTTP, **Kode Status 3 Digit (`200`)**, dan keterangan singkat (`OK`).
2. **Deretan *Response Headers*:**
   Informasi tentang balasan server, misalnya `Content-Type: application/json` (memberitahu client bahwa isi balasannya berformat JSON) dan `Content-Length: 15` (ukuran data 15 byte).
3. **Baris Kosong + *Response Body*:**
   Isi data sesungguhnya yang dikirim oleh server (bisa berupa kode HTML halaman web, data JSON, atau gambar).

### Cara Melihat Seluruh Percakapan HTTP dengan `curl -v`
Di terminal, perintah `curl` biasa hanya menampilkan *Response Body*. Untuk melihat amplop Request dan Response secara lengkap (sangat penting saat *debugging*!), tambahkan flag **`-v` (*verbose*)**:

```bash
curl -v http://example.com/
```
Cara membaca simbol pada output `curl -v`:
- Baris yang diawali **`*`** : Informasi proses di balik layar (resolusi DNS, IP tujuan, jabat tangan TCP & TLS).
- Baris yang diawali **`>`** : **HTTP Request** yang **dikirim keluar** oleh `curl` ke server.
- Baris yang diawali **`<`** : **HTTP Response Header** yang **diterima masuk** dari server.
- Baris tanpa simbol di bagian akhir: **Response Body** (isi halaman/data).

---

## 3. HTTP Method & Konsep Penting *Idempotency*

Saat Client berbicara ke Server, ia menggunakan **HTTP Method** (kata kerja) untuk menyatakan niatnya:

| Method | Arti dalam Bahasa Manusia | Punya Body? | Apakah *Idempotent*? | Contoh Penggunaan |
|:---:|---|---|:---:|---|
| **`GET`** | **"Tolong tampilkan/baca data ini"** (tanpa mengubah apa pun di server). | Tidak | ✅ **Ya** (Aman) | `GET /users/42` (Lihat profil user ID 42) |
| **`POST`** | **"Tolong buatkan data baru / jalankan proses ini"**. | Ya | ❌ **TIDAK** | `POST /orders` (Buat pesanan baru / bayar tagihan) |
| **`PUT`** | **"Tolong timpa/ganti seluruh data di alamat ini dengan data baru ini"**. | Ya | ✅ **Ya** | `PUT /users/42` (Ganti seluruh data profil user 42) |
| **`PATCH`** | **"Tolong ubah sebagian kolom saja dari data ini"**. | Ya | ⚠️ Tergantung implementasi | `PATCH /users/42` (Ubah nomor HP-nya saja) |
| **`DELETE`** | **"Tolong hapus data di alamat ini"**. | Tidak | ✅ **Ya** | `DELETE /users/42` (Hapus user 42) |
| **`HEAD`** | Sama persis seperti `GET`, tapi **hanya minta Header-nya saja tanpa Body** (sering dipakai untuk cek apakah file ada / *health check* ringan). | Tidak | ✅ **Ya** | `HEAD /index.html` |

### Mengapa SRE Wajib Paham Konsep *Idempotent*?
> 🔑 **Definisi *Idempotent*:** Sebuah aksi disebut **Idempotent** apabila **dijalankan 1 kali maupun diulang 10 kali berturut-turut dengan input yang sama, kondisi akhir di server tetap sama persis** (tidak menimbulkan efek samping ganda).

Bayangkan skenario nyata di production:
1. Aplikasi pembayaran mengirim request ke server bank, tetapi tiba-tiba jaringan putus di detik terakhir (timeout). Aplikasi tidak tahu apakah request tadi sudah diproses atau belum, sehingga sistem **otomatis mengulang (*auto-retry*)** request tersebut.
2. Jika endpoint tersebut menggunakan **`GET /saldo`** atau **`DELETE /keranjang/item-99`**: Diulang berkali-kali pun hasilnya tetap sama (item-99 tetap terhapus satu kali). **Aman untuk di-*retry* otomatis!**
3. Jika endpoint tersebut menggunakan **`POST /transfer-uang`** (tanpa kunci *idempotency key*): Mengulang `POST` dua kali akan membuat **dua transaksi transfer baru** dan memotong saldo nasabah dua kali!
   - Itulah sebabnya Reverse Proxy / Load Balancer secara default **hanya berani melakukan *auto-retry* pada method yang idempotent (seperti `GET`)**, dan tidak berani mengulang `POST` secara sembarangan!

```bash
# Contoh mengirim request dengan berbagai HTTP Method menggunakan curl:
# 1. GET (default, tidak perlu ditulis -X GET):
curl http://localhost:8080/users

# 2.HEAD (ambil header saja dengan flag -I):
curl -I http://localhost:8080/users

# 3. POST dengan mengirim data JSON (-d) dan memberi tahu tipe datanya (-H):
curl -X POST http://localhost:8080/users \
  -H 'Content-Type: application/json' \
  -d '{"name": "budi", "role": "sre"}'
```

---

## 4. HTTP Status Code — Membaca Bahasa Kode Server

Setiap kali server membalas HTTP Request, ia memberikan angka 3 digit yang disebut **Status Code**. Bagi seorang SRE, angka ini adalah **petunjuk utama saat terjadi insiden** dan menjadi dasar perhitungan **SLI/SLO (*Error Budget*)** di Fase 7 & 8!

### A. Pengelompokan 5 Kategori Status Code

| Kategori | Makna Umum | Analogi Restoran | Kode yang Paling Sering Muncul |
|:---:|---|---|---|
| **`1xx`** | **Informational** (Informasi sementara) | *"Tunggu sebentar, pesanan sedang disambungkan..."* | `101 Switching Protocols` (Saat koneksi berubah dari HTTP biasa menjadi *WebSocket*) |
| **`2xx`** | **Success** (Berhasil!) | *"Ini hidangan pesanan Anda, selamat menikmati!"* | • **`200 OK`**: Sukses standar.<br>• **`201 Created`**: Data baru berhasil dibuat (biasanya balasan `POST`).<br>• **`204 No Content`**: Sukses diproses, tapi memang tidak ada isi Body yang perlu ditampilkan (misal balasan `DELETE`). |
| **`3xx`** | **Redirection** (Pindah Alamat) | *"Restoran cabang ini sudah pindah ke gedung sebelah, silakan ke sana."* | • **`301 Moved Permanently`** / **`308 Permanent Redirect`**: Pindah alamat selamanya (misal dari `http://` otomatis diarahkan ke `https://`).<br>• **`302 Found`** / **`307 Temporary Redirect`**: Pindah alamat sementara.<br>• **`304 Not Modified`**: Data belum berubah sejak terakhir kali kamu buka; pakai saja *cache* yang ada di laptopmu! |
| **`4xx`** | **Client Error** (**Salah si Pemanggil / Client!**) | *"Maaf, pesanan Anda salah tulis / Anda belum bayar / menu itu tidak ada."* | • **`400 Bad Request`**: Format JSON/request yang dikirim client rusak/salah sintaks.<br>• **`401 Unauthorized`**: Belum login / token salah.<br>• **`403 Forbidden`**: Sudah login, tapi tidak punya hak akses.<br>• **`404 Not Found`**: URL/path yang diminta tidak ditemukan.<br>• **`429 Too Many Requests`**: Client mengirim terlalu banyak request (terkena *Rate Limit*). |
| **`5xx`** | **Server Error** (**Salah Server / Infrastruktur Kita!**) | *"Maaf, kompor di dapur kami meledak / koki kami pingsan!"* | • **`500 Internal Server Error`**: Kode aplikasi di server *crash* / error tidak tertangani (misal *NullPointerException* / gagal query DB).<br>• **`502 Bad Gateway`**: Reverse Proxy menerima balasan rusak/kosong karena aplikasi di belakangnya mati.<br>• **`503 Service Unavailable`**: Server sedang kelebihan beban (*overload*) atau belum siap menerima trafik.<br>• **`504 Gateway Timeout`**: Aplikasi di belakang proxy terlalu lama berpikir sampai batas waktu tunggu habis. |

> 🚨 **Prinsip Emas Monitoring SRE (`4xx` vs `5xx`):**
> - Jika terjadi lonjakan **`4xx`** (misal banyak `404`), biasanya itu kesalahan pengguna yang mengetik URL salah atau *scanner* bot di internet — **server kita sendiri sebenarnya sehat**.
> - Namun jika terjadi lonjakan **`5xx`**, itu artinya **sistem produksi kita sedang rusak/gagal melayani pengguna!** Alert darurat (*PagerDuty/Alertmanager*) untuk membangunkan SRE di tengah malam hampir selalu dipasang pada kenaikan persentase **`5xx`**!

---

### B. Dua Pasangan Kode yang Paling Sering Tertukar (Wajib Hafal!)

#### 1. Bedanya `401 Unauthorized` vs `403 Forbidden`
Meskipun kata bahasa Inggris *"Unauthorized"* terdengar seperti "tidak punya izin", di standar HTTP artinya berbeda:
- **`401 Unauthorized` = Masalah *Authentication* ("Kamu siapa? Saya belum kenal kamu!")**
  - **Penyebab:** Client belum menyertakan token login, belum login, atau token JWT/password-nya sudah kedaluwarsa.
  - **Solusi:** Silakan login dulu atau perbarui token di header `Authorization`.
- **`403 Forbidden` = Masalah *Authorization* ("Saya tahu kamu Budi, tapi Budi DILARANG masuk ke ruang brankas Direktur!")**
  - **Penyebab:** Identitas/token kamu **valid dan dikenali**, tetapi akun/role kamu tidak memiliki hak akses (*permission*) ke halaman atau aksi tersebut (atau IP kamu diblokir aturan *WAF/ACL*).
  - **Solusi:** Login ulang pun tidak akan menyelesaikan masalah; akun tersebut harus diberi *role/permission* tambahan oleh administrator.

#### 2. Bedanya `500` vs `502` vs `503` vs `504` Saat Insiden On-Call
Saat kamu mendapat laporan *"Website tidak bisa diakses, muncul error 50x!"*, angka terakhir langsung memberitahu kamu **di komponen mana kerusakannya berada**:

```text
  [ Pengguna ] ──► [ Reverse Proxy / Ingress ] ──► [ Aplikasi Backend ] ──► [ Database ]
                            │                              │
     • 500 Internal Error ──┼──────────────────────────────┴─► Aplikasi hidup, tapi kodenya error/exception!
     • 502 Bad Gateway ─────┴─► Proxy hidup, tapi Aplikasi Backend MATI / menolak koneksi (Connection Refused)!
     • 503 Unavailable ─────┴─► Proxy tidak punya Pod backend yang sehat (Readiness Probe gagal / lagi penuh)!
     • 504 Gateway Timeout ─┴─► Proxy sudah kirim request ke Aplikasi, tapi Aplikasi diam saja > 60 detik!
```

| Kode Error | Apa yang Sebenarnya Terjadi di Balik Layar? | Langkah Pertama SRE Saat Troubleshooting |
|:---:|---|---|
| **`500`** | Request **berhasil masuk ke aplikasi**, tetapi kode program mengalami error fatal (misal salah kueri SQL, disk penuh, atau gagal baca variabel). | Cek langsung **log aplikasi backend** (`journalctl -u app` atau `kubectl logs <pod>`). |
| **`502`** (*Bad Gateway*) | Reverse Proxy (Caddy/Nginx/Ingress) mencoba meneruskan request ke aplikasi backend, tapi **aplikasi backend mati total**, port-nya salah, atau tiba-tiba *crash* (`OOMKilled`) di tengah jalan. | Cek apakah aplikasi backend menyala (`systemctl status` / `ss -tlnp` / `kubectl get pods`). Pastikan port upstream di konfigurasi proxy sudah benar! |
| **`503`** (*Service Unavailable*) | Server atau Reverse Proxy menyatakan **"Saat ini tidak ada pekerja yang siap melayani"**. Di Kubernetes, ini muncul jika semua Pod belum lulus *Readiness Probe* atau sedang di-*scale* ke 0. | Cek status kesehatan (*health check*) Pod/service backend atau cek apakah antrean server sedang *overload*. |
| **`504`** (*Gateway Timeout*) | Reverse Proxy sudah berhasil menghubungi aplikasi backend dan mengirim request, lalu menunggu... namun **sampai batas waktu habis (misal 30 atau 60 detik), aplikasi belum juga selesai menjawab!** | Aplikasi sedang *hang*, terjebak *deadlock*, atau sedang menunggu kueri Database / API pihak ketiga yang sangat lambat. |

---

## 5. HTTP Header Penting untuk SRE

**HTTP Header** adalah informasi metadata yang dikirim bersama Request maupun Response. Berikut adalah daftar header yang paling sering diperiksa oleh SRE saat men-debug sistem produksi:

| Nama Header | Arah | Fungsi & Mengapa Penting bagi SRE |
|---|:---:|---|
| **`Host`** | Request | **Penentu alamat tujuan (*Virtual Host*).** Memberitahu Reverse Proxy/Ingress nama domain apa yang sebenarnya sedang diakses oleh client (misal `Host: grafana.lab.local`). |
| **`Content-Type`** | Keduanya | Memberitahu format data di dalam *Body* (misal `application/json`, `text/html`, atau `multipart/form-data`). **Tips:** Sering kali error `400 Bad Request` terjadi hanya karena kita lupa mengirim `-H 'Content-Type: application/json'` saat `curl -X POST`! |
| **`Content-Length`** | Keduanya | Ukuran *Body* dalam satuan byte, agar penerima tahu kapan pembacaan data selesai. |
| **`Authorization`** | Request | Membawa kredensial keamanan, misalnya `Authorization: Bearer <TOKEN_JWT>` atau Basic Auth. |
| **`User-Agent`** | Request | Identitas program pemanggil (misal `Mozilla/5.0...`, `curl/8.5.0`, atau `kube-probe/1.29` milik health check Kubernetes). |
| **`Location`** | Response | Muncul saat server membalas dengan kode redirect (`301`/`302`/`308`). Berisi URL tujuan baru yang harus dikunjungi oleh client. |
| **`X-Forwarded-For`** | Request (dari Proxy) | **Menyimpan alamat IP asli milik pengguna** sebelum melewati Reverse Proxy / Load Balancer. |
| **`X-Forwarded-Proto`** | Request (dari Proxy) | Memberitahu aplikasi backend protokol apa yang dipakai pengguna di depan proxy (`https` atau `http`). |

### Mengapa Header `X-Forwarded-For` dan `X-Forwarded-Proto` Sangat Krusial?

Bayangkan ada pengguna dengan IP `203.0.113.99` mengakses website kamu melalui **Reverse Proxy** (yang memiliki IP internal `10.0.0.1`).
Karena Reverse Proxy membuka koneksi TCP baru ke aplikasi backend, maka di mata aplikasi backend, **semua koneksi terlihat datang dari IP `10.0.0.1` (IP milik Proxy), bukan IP asli pengguna!**

```text
  [ Pengguna Asli ]               [ Reverse Proxy (Caddy) ]               [ Aplikasi Backend ]
  IP: 203.0.113.99                        IP: 10.0.0.1                        IP: 10.0.0.2
         │                                     │                                   │
         │  1. Koneksi HTTPS dari 203.0.113.99 │                                   │
         │ ══════════════════════════════════► │                                   │
         │                                     │  2. Koneksi HTTP dari 10.0.0.1    │
         │                                     │     + Header Tambahan Otomatis:   │
         │                                     │       X-Forwarded-For: 203.0.113.99
         │                                     │       X-Forwarded-Proto: https    │
         │                                     │ ────────────────────────────────► │
```

Apa akibatnya jika tidak ada header `X-Forwarded-*`?
1. **Tanpa `X-Forwarded-For`:** Semua log kunjungan di aplikasi hanya mencatat IP `10.0.0.1`. Lebih parah lagi, jika ada 1 peretas mencoba menebak password berkali-kali dan aplikasi memblokir IP pemanggil, **aplikasi malah memblokir IP Reverse Proxy (`10.0.0.1`) sehingga seluruh pengguna di dunia ikut terblokir!**
2. **Tanpa `X-Forwarded-Proto`:** Karena komunikasi antara Proxy ke Aplikasi menggunakan HTTP biasa (setelah *TLS Termination*), aplikasi mengira pengguna mengakses lewat `http://` tidak aman, lalu aplikasi menyuruh pengguna redirect ke `https://` terus-menerus tanpa henti (**Infinite Redirect Loop `ERR_TOO_MANY_REDIRECTS`**)!
3. Untungnya, Reverse Proxy modern seperti **Caddy** sudah **secara otomatis** menyematkan header `X-Forwarded-For`, `X-Forwarded-Proto`, dan `X-Forwarded-Host` setiap kali meneruskan pesan ke aplikasi backend.

---

## 6. Forward Proxy vs Reverse Proxy — Jangan Sampai Terbalik!

Kata **"Proxy"** secara harfiah berarti **"Perantara / Wakil"**. Di dunia jaringan ada dua jenis proxy yang posisinya saling berkebalikan:

```text
  A. FORWARD PROXY (Perantara di Sisi Client / Pintu Keluar Kantor)
  ┌─────────────────────────┐
  │ Laptop Karyawan A       │──┐
  │ Laptop Karyawan B       │──┼──► [ Forward Proxy ] ──────► ( INTERNET ) ──► Website Luar
  └─────────────────────────┘  │    (Wakil Client)
   "Melindungi & menyembunyikan identitas Client yang mau keluar ke Internet"

  ─────────────────────────────────────────────────────────────────────────────────────────

  B. REVERSE PROXY (Perantara di Sisi Server / Resepsionis Gedung Kita)
                                                              ┌──► [ App Backend :8080 ]
  Pengguna di Internet ──► ( INTERNET ) ──► [ Reverse Proxy ]─┼──► [ Grafana     :3000 ]
                                            (Wakil Server)    └──► [ API Service :9000 ]
   "Melindungi, mengatur rute, & menyembunyikan deretan Server kita dari Internet"
```

| Aspek | **Forward Proxy** (Jarang diurus SRE App) | **Reverse Proxy** (Makanan Sehari-hari SRE!) |
|---|---|---|
| **Siapa yang diwakili?** | Mewakili **Client** (pengguna yang ingin mengakses internet luar). | Mewakili **Server** (menerima tamu dari luar untuk diteruskan ke aplikasi di dalam). |
| **Analogi Dunia Nyata** | **Asisten Pribadi** yang kamu suruh keluar rumah untuk membelikan barang di toko (penjual toko hanya melihat wajah asistenmu, bukan wajahmu). | **Resepsionis Hotel / Lobby Gedung** yang menyambut semua tamu di pintu depan, memeriksa KTP, lalu mengarahkan tamu ke kamar/departemen yang tepat di dalam gedung. |
| **Tujuan Utama** | Menyaring situs yang boleh dibuka karyawan kantor, hemat kuota lewat caching, atau menyembunyikan IP client (mirip VPN). | **Terminasi TLS (HTTPS) satu pintu**, membagi beban (**Load Balancing**), mengarahkan banyak domain/aplikasi lewat 1 IP (**Routing**), dan **menyembunyikan server backend** dari serangan langsung. |
| **Contoh Alat** | Squid Proxy, Corporate Web Filter. | **Caddy**, **Nginx**, **HAProxy**, **Traefik**, **Envoy**. |

> 💡 **Kaitan Langsung dengan Kubernetes (Fase 2):**
> Saat nanti kita masuk ke **Kubernetes (Modul 2.1)**, komponen yang bertugas menerima trafik HTTP/HTTPS dari luar cluster dan membaginya ke berbagai service di dalam cluster disebut **Kubernetes Ingress**.
> Apa isi di dalam *Ingress Controller* (seperti Traefik bawaan k3s atau Nginx Ingress)? **Isinya adalah Reverse Proxy!** Jadi dengan memahami cara kerja Reverse Proxy di modul ini, kamu otomatis sudah menguasai 80% konsep kerja Kubernetes Ingress!

---

## 7. Host-Based & Path-Based Routing (Cara Satu IP Melayani Banyak Aplikasi)

Pertanyaan yang sering ditanyakan pemula:
> *"Di VM atau server kita hanya ada **1 alamat IP** dan pintu standar HTTPS hanya **1 nomor port (Port `443`)**. Tapi di dalam server kita ada 3 aplikasi berbeda: Aplikasi Utama (port `8080`), Dashboard Grafana (port `3000`), dan Service Pembayaran (port `9000`). Bagaimana caranya agar pengguna tidak perlu mengetik nomor port `:8080` atau `:3000` di browser?"*

Jawabannya adalah dengan menaruh **Reverse Proxy** di port `80` dan `443` sebagai resepsionis tunggal! Reverse Proxy menggunakan dua teknik pengaturan rute (*Routing*):

### A. *Host-Based Routing* (Routing Berdasarkan Nama Domain di Header `Host`)
Semua nama domain (`app.lab.local` dan `grafana.lab.local`) diarahkan di DNS ke **alamat IP server yang sama** (misal `192.168.64.5`). Saat paket HTTP tiba di port `80`/`443`, Reverse Proxy membaca **Header `Host`** di dalam amplop HTTP untuk menentukan ke port internal mana pesan itu harus diteruskan:

```text
  Browser ketik: https://app.lab.local        Browser ketik: https://grafana.lab.local
  (Header -> Host: app.lab.local)             (Header -> Host: grafana.lab.local)
                 │                                           │
                 └───────────────────┐   ┌───────────────────┘
                                     ▼   ▼
                       ┌───────────────────────────────┐
                       │     REVERSE PROXY (Caddy)     │
                       │    Mendengarkan di Port 443   │
                       └───────┬───────────────┬───────┘
          Jika Host ==         │               │         Jika Host ==
         app.lab.local         ▼               ▼       grafana.lab.local
                   ┌──────────────────┐ ┌──────────────────┐
                   │   App Backend    │ │     Grafana      │
                   │ (localhost:8080) │ │ (localhost:3000) │
                   └──────────────────┘ └──────────────────┘
```

### B. *Path-Based Routing* (Routing Berdasarkan Akhiran URL / *Path*)
Selain membedakan dari nama domain, Reverse Proxy juga bisa membedakan tujuan berdasarkan **akhiran path URL** pada domain yang sama:
- Jika pengguna mengakses `https://monitor.lab.local/api/...` $\rightarrow$ teruskan ke service API di `localhost:8080`.
- Jika pengguna mengakses `https://monitor.lab.local/` (selain `/api/*`) $\rightarrow$ teruskan ke service tampilan web (UI) di `localhost:3000`.

### Teknik SRE: Menguji Routing dengan "Spoofing Header `Host`"
Bagaimana jika kamu baru selesai mengatur *Host-Based Routing* di Reverse Proxy untuk domain `app.lab.local`, tetapi **DNS untuk `app.lab.local` belum selesai diatur**?
Kamu tidak perlu menunggu DNS selesai! Kamu bisa langsung mengirim request ke `localhost` (atau IP server) sambil **memalsukan/menyisipkan Header `Host` secara manual** menggunakan `curl -H`:

```bash
# Menghubungi IP 127.0.0.1 secara langsung, tapi berpura-pura mengakses domain app.lab.local:
curl -H 'Host: app.lab.local' http://127.0.0.1/
```
Trik sederhana ini adalah senjata andalan SRE setiap hari untuk mengetes apakah konfigurasi Reverse Proxy atau Kubernetes Ingress sudah berjalan benar sebelum mengubah pengaturan DNS publik!

---

## 8. Mengenal Caddy — Reverse Proxy Modern Pilihan Kita

Di dunia industri, ada beberapa Reverse Proxy populer seperti **Nginx**, **HAProxy**, **Traefik**, dan **Caddy**. Mengapa di modul fondasi ini kita memilih **Caddy**?
1. **Konfigurasi Super Ringkas & Manusiawi (`Caddyfile`):** Apa yang membutuhkan 35 baris konfigurasi rumit di Nginx cukup ditulis dalam **3 baris** di Caddy!
2. **HTTPS Otomatis (*Automatic HTTPS*):** Untuk domain publik, Caddy otomatis memintakan dan memperpanjang sertifikat TLS ke *Let's Encrypt* tanpa perlu setup tambahan. Untuk domain lokal/internal (`.local`), cukup tambahkan 1 baris `tls internal` dan Caddy otomatis menjadi *Certificate Authority (CA)* lokal!
3. **Ditulis dalam Bahasa Go & Native ARM64:** Sangat ringan dan berjalan mulus di VM Ubuntu ARM64 (Apple Silicon / OrbStack).

### Perbandingan Sekilas: Konfigurasi Caddy vs Nginx
Untuk membuat reverse proxy HTTPS dari `app.lab.local` ke `localhost:8080`:

```caddyfile
# Di Caddy (/etc/caddy/Caddyfile) — Hanya 4 baris, sudah otomatis HTTPS & HTTP->HTTPS redirect!
app.lab.local {
    tls internal
    reverse_proxy localhost:8080
}
```
*(Bandingkan dengan Nginx di mana kamu harus menulis blok `server` port 80 untuk redirect, blok `server` port 443, menentukan path `ssl_certificate`, `ssl_certificate_key`, `proxy_set_header Host $host`, `proxy_set_header X-Forwarded-For`, dll.)*

### Contoh `Caddyfile` Lengkap (Host-Based + Path-Based Routing)

```caddyfile
# /etc/caddy/Caddyfile

# 1. Situs Pertama: Host-Based Routing ke Port 8080
app.lab.local {
    tls internal
    reverse_proxy localhost:8080
}

# 2. Situs Kedua: Host-Based Routing ke Port 3000
grafana.lab.local {
    tls internal
    reverse_proxy localhost:3000
}

# 3. Situs Ketiga: Gabungan Host-Based & Path-Based Routing
monitor.lab.local {
    tls internal

    # Jika path URL diawali /api/* -> arahkan ke service backend di port 8080
    handle /api/* {
        reverse_proxy localhost:8080
    }

    # Untuk semua path lainnya -> arahkan ke service frontend di port 3000
    handle {
        reverse_proxy localhost:3000
    }
}
```

### Perintah Penting Mengelola Caddy di Server
```bash
# 1. Cek apakah penulisan Caddyfile sudah valid (sebelum di-reload!):
caddy validate --config /etc/caddy/Caddyfile

# 2. Rapikan format penulisan/indentasi Caddyfile secara otomatis:
sudo caddy fmt --overwrite /etc/caddy/Caddyfile

# 3. Terapkan perubahan konfigurasi tanpa memutus koneksi pengguna (Zero-Downtime Reload):
sudo systemctl reload caddy

# 4. Cek status dan log jika terjadi error:
sudo systemctl status caddy
sudo journalctl -u caddy -n 30 --no-pager
```

---

## 9. Pola Kegagalan (Failure Modes) di Production & Cara Diagnosanya

Tabel ini merangkum skenario masalah nyata yang sering terjadi pada lapisan HTTP & Reverse Proxy beserta langkah diagnosa sistematisnya:

| Gejala / Masalah yang Dilaporkan | Penyebab Paling Umum | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| Browser / `curl` menerima **`502 Bad Gateway`** dari Caddy/Nginx | Reverse Proxy sehat, tetapi **aplikasi backend di belakangnya mati** atau menolak koneksi (*Connection Refused*). | 1. Cek apakah proses backend hidup: `ss -tlnp \| grep 8080`.<br>2. Tes tembak langsung ke backend tanpa lewat proxy: `curl -v http://localhost:8080/`.<br>3. Nyalakan ulang service backend yang mati. |
| Browser / `curl` menerima **`504 Gateway Timeout`** | Aplikasi backend terlalu lambat merespons hingga melewati batas waktu tunggu proxy. | 1. Cek beban CPU/RAM server (`htop`) dan log aplikasi backend.<br>2. Cek apakah backend macet saat menghubungi Database atau API lain. |
| Aplikasi terjebak **`ERR_TOO_MANY_REDIRECTS`** (Redirect `301`/`302` berulang tanpa henti) | Aplikasi backend mewajibkan HTTPS, tetapi tidak sadar bahwa TLS sudah dibuka (*terminated*) di Reverse Proxy. | Pastikan Reverse Proxy mengirimkan header `X-Forwarded-Proto: https` dan aktifkan pengaturan *Trust Proxy* di aplikasi backend. |
| `curl https://app.lab.local` gagal dengan **`SSL: certificate verify failed`** | Sertifikat TLS yang dipakai adalah sertifikat internal / *self-signed* yang belum terdaftar di daftar kepercayaan (*Trust Store*) OS pemanggil. | • Untuk uji cepat di terminal: tambahkan flag `-k` (`curl -k https://...`).<br>• Untuk solusi permanen: daftarkan Root CA internal ke Keychain Mac / `/usr/local/share/ca-certificates/` Linux (akan dipraktikkan di [LAB-01](lab/LAB-01-reverse-proxy.md)). |
| Mengakses `app.lab.local` dan `grafana.lab.local` malah menampilkan halaman yang sama | Konfigurasi *Virtual Host* salah atau client mengakses lewat alamat IP langsung tanpa mengirimkan header `Host` yang sesuai. | Uji masing-masing domain dengan `curl -v -H 'Host: nama.domain' http://localhost/` dan periksa blok nama domain di `Caddyfile`. |

---

## 10. Latihan Praktik Mandiri (15 Menit di Terminal VM OrbStack)

Sebelum masuk ke instalasi penuh Caddy di [LAB-01](lab/LAB-01-reverse-proxy.md), mari kita bangun *muscle memory* menginspeksi HTTP Request, Response, Header, dan Status Code langsung di VM `lab01`:

```bash
# 1. Buat folder latihan dan 1 file JSON sederhana untuk disajikan oleh server uji:
mkdir -p ~/latihan-http && cd ~/latihan-http
echo '{"service": "payment-api", "status": "healthy"}' > health.json

# 2. Jalankan HTTP server bawaan Python di port 8080 di background (&):
python3 -m http.server 8080 >/tmp/http_test.log 2>&1 &
SERVER_PID=$!
sleep 1

# 3. Pastikan port 8080 benar-benar sudah LISTEN:
ss -tlnp | grep :8080

# 4. Amati percakapan HTTP lengkap (Request '>' dan Response '<') dengan curl -v:
curl -v http://localhost:8080/health.json

# 5. Ambil HANYA HTTP Response Header saja (tanpa mencampur Body):
curl -s -D - -o /dev/null http://localhost:8080/health.json

# 6. Ambil HANYA angka Status Code 3 digit (teknik favorit untuk skrip monitoring!):
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/health.json
# Output yang diharapkan: 200

# 7. Uji meminta file yang TIDAK ADA untuk melihat kode 404 Not Found:
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/file-gaib.json
# Output yang diharapkan: 404

# 8. Uji mengirim method POST ke server statis (lihat bagaimana server menolak dengan 501/405):
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8080/health.json

# 9. Uji menyisipkan Custom Header (simulasi Host-Based Routing & X-Forwarded-For):
curl -v -H 'Host: api.perusahaan.local' -H 'X-Forwarded-For: 203.0.113.55' \
  http://localhost:8080/health.json

# 10. Lihat catatan log di sisi server (perhatikan status code 200, 404, 501 yang tercatat!):
cat /tmp/http_test.log

# 11. Matikan server uji dan bersihkan folder latihan:
kill $SERVER_PID
cd ~ && rm -rf ~/latihan-http /tmp/http_test.log
```

**Acceptance Criteria:**
- [ ] Mampu menunjukkan baris Request (`> GET /health.json HTTP/1.1`) dan baris Response (`< HTTP/1.0 200 OK`) pada output `curl -v`.
- [ ] Berhasil mengekstrak hanya angka `200` (saat file ada) dan `404` (saat file tidak ada) menggunakan flag `-w "%{http_code}\n"`.
- [ ] Mampu menyisipkan header `Host` dan `X-Forwarded-For` menggunakan flag `-H` dan membuktikannya muncul di bagian `>` pada `curl -v`.
- [ ] Proses `python3 -m http.server` sudah dihentikan dan direktori `~/latihan-http` sudah dibersihkan.

---

## 11. Ringkasan Perintah (Cheat Sheet)

| Kebutuhan Operasional | Perintah Utama |
|---|---|
| Lihat percakapan HTTP lengkap (Request + Response + Info TCP/TLS) | `curl -v http://localhost:8080/` |
| Tampilkan **Header Response saja** (Body dibuang) | `curl -s -D - -o /dev/null http://localhost:8080/` atau `curl -I http://localhost:8080/` |
| Tampilkan **angka Status Code saja** (misal `200`, `404`, `502`) | `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/` |
| Ikuti arah *Redirect* (`301`/`302`/`308`) secara otomatis | `curl -L http://domain.com/` |
| Kirim request `POST` berisi data JSON | `curl -X POST -H 'Content-Type: application/json' -d '{"key":"val"}' http://localhost:8080/api` |
| Uji *Virtual Host* (palsukan header `Host` tanpa ubah DNS) | `curl -H 'Host: app.lab.local' http://127.0.0.1/` |
| Abaikan validasi sertifikat TLS saat mengetes HTTPS *self-signed* | `curl -k https://app.lab.local/` |
| Validasi sintaks `Caddyfile` sebelum reload | `caddy validate --config /etc/caddy/Caddyfile` |
| Terapkan perubahan konfigurasi Caddy tanpa *downtime* | `sudo systemctl reload caddy` |

---

## 12. Cek Pemahaman (Self-Check)

Uji pemahamanmu dengan menjawab pertanyaan berikut sebelum membuka kunci jawaban:

1. **Apa perbedaan mendasar antara kode status `401 Unauthorized` dan `403 Forbidden`? Mengapa tim developer tidak boleh menukar penggunaan kedua kode ini di aplikasi?**
2. **Saat sedang piket *on-call*, kamu mendapati Reverse Proxy mengembalikan `502 Bad Gateway` pada pagi hari, lalu pada sore hari mengembalikan `504 Gateway Timeout`. Apa perbedaan kondisi aplikasi backend pada kedua kejadian tersebut?**
3. **Mengapa aplikasi yang berada di belakang Reverse Proxy tidak bisa melihat alamat IP asli milik pengguna secara langsung dari koneksi TCP, dan bagaimana solusinya?**
4. **Bagaimana caranya satu server dengan 1 alamat IP dan 1 port HTTPS (`443`) bisa melayani 5 website berbeda yang berjalan di port internal berbeda-beda?**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **Perbedaan `401 Unauthorized` vs `403 Forbidden`:**
   - **`401 Unauthorized`** berarti masalah **Otentikasi (*Authentication*)**: server belum tahu siapa pemanggilnya karena belum login atau tokennya salah/kedaluwarsa. Solusinya adalah melakukan login ulang.
   - **`403 Forbidden`** berarti masalah **Otorisasi (*Authorization*)**: server sudah mengenali identitas pemanggil dengan benar, tetapi akun tersebut memang **tidak punya izin/hak akses** ke resource tersebut.
   - Jika tertukar (misalnya mengembalikan `401` padahal seharusnya `403`), aplikasi frontend bisa terjebak meminta user login berulang-ulang padahal user sudah login dengan benar.

2. **Perbedaan `502 Bad Gateway` vs `504 Gateway Timeout` bagi SRE:**
   - Pada **`502 Bad Gateway`**, Reverse Proxy **sama sekali gagal mendapatkan respons yang sah** dari aplikasi backend — biasanya karena proses aplikasi backend mati/crash, port belum terbuka (*Connection Refused*), atau container baru saja terkena *OOMKilled*.
   - Pada **`504 Gateway Timeout`**, aplikasi backend **masih hidup dan menerima koneksi** dari Reverse Proxy, tetapi proses pengerjaannya **terlalu lambat** hingga melewati batas waktu tunggu (*timeout*) — biasanya karena kueri database macet (*locked*) atau menunggu panggilan API eksternal yang lambat.

3. **Mengapa IP asli pengguna tertutup oleh Reverse Proxy & Solusinya:**
   - Karena koneksi TCP dari pengguna berhenti di Reverse Proxy, lalu Reverse Proxy membuat koneksi TCP baru dari dirinya sendiri menuju aplikasi backend. Akibatnya, di level TCP, aplikasi hanya melihat alamat IP milik Reverse Proxy.
   - **Solusinya:** Reverse Proxy menyisipkan alamat IP asli pengguna ke dalam HTTP Header **`X-Forwarded-For`** sebelum meneruskan request ke aplikasi backend.

4. **Cara 1 IP dan Port `443` melayani 5 website berbeda:**
   - Menggunakan mekanisme **Host-Based Routing** pada Reverse Proxy (dan ekstensi **SNI** saat jabat tangan TLS).
   - Setiap kali browser mengakses salah satu dari 5 domain tersebut, browser menyertakan nama domain yang dituju pada **Header `Host`** (misal `Host: web1.contoh.com` atau `Host: web2.contoh.com`). Reverse Proxy di port `443` membaca header `Host` tersebut lalu meneruskan request ke port internal masing-masing aplikasi (`localhost:8001`, `localhost:8002`, dst.).

</details>

---

## 13. Langkah Selanjutnya

Kamu sekarang sudah memahami cara kerja protokol HTTP, arti di balik kode status `2xx`/`4xx`/`5xx`, serta bagaimana Reverse Proxy mengatur lalu lintas masuk ke aplikasi.
- Untuk langsung mempraktikkan pembuatan Reverse Proxy **Caddy** dengan dua service backend dan HTTPS otomatis di VM `lab01`, kamu dapat mengerjakan **[LAB-01: Reverse Proxy Caddy + TLS Otomatis](lab/LAB-01-reverse-proxy.md)**.
- Untuk melengkapi kepingan terakhir fondasi jaringan (bagaimana paket diizinkan/diblokir oleh **Firewall**, diterjemahkan oleh **NAT**, dan bagaimana **ARP** bekerja sebagai fondasi **MetalLB** di Kubernetes), lanjutkan membaca ke **[04-firewall-nat-arp.md](04-firewall-nat-arp.md)**!