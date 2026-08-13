# 02 — TCP, UDP & TLS

> Handshake, perbedaan transport, dan enkripsi di atas kabel — secara konsep.

## Tujuan
- Paham TCP three-way handshake & state koneksi
- Bisa membedakan kapan pakai TCP vs UDP
- Paham konsep TLS: sertifikat, handshake, terminasi
- Bisa memeriksa sertifikat dengan `openssl`

## 1. TCP — Connection-Oriented

TCP = andal, berurutan, ada ack. Cocok untuk HTTP, SSH, database.

**Three-way handshake:**
```
Client                         Server
  │ ── SYN ──────────────────► │
  │ ◄── SYN-ACK ────────────── │
  │ ── ACK ──────────────────► │   ← koneksi established
  │                            │
  │ ── data ─────────────────► │
  │ ◄── ACK ────────────────── │
```

```bash
# Lihat handshake secara langsung (di VM lab01):
sudo tcpdump -i any -n port 80 -c 10        # buka terminal lain: curl http://example.com
ss -tn state established                    # koneksi yang sedang aktif
ss -tn state time-wait                      # koneksi yang baru selesai
```

**State koneksi penting** ( troubleshooting "port penuh"):
| State | Arti |
|---|---|
| `LISTEN` | server menunggu koneksi |
| `SYN-SENT` | client kirim SYN, tunggu balasan |
| `ESTABLISHED` | koneksi hidup, data mengalir |
| `TIME-WAIT` | client tutup, tunggu periode aman (~60s) |
| `CLOSE-WAIT` | server tutup dari sisi jauh, lokal belum tutup |

Banyak `TIME-WAIT` = tanda koneksi dibuka-tutup cepat ( HTTP tanpa keep-alive). Di server sibuk, ini bisa habiskan port ephemeral.

## 2. UDP — Connectionless

UDP = kirim dan lupa. Tidak ada handshake, tidak ada ack, tidak ada urutan. Cepat tapi tidak andal.

```bash
sudo tcpdump -i any -n udp -c 10
```

| Aspek | TCP | UDP |
|---|---|---|
| Handshake | ya (3-way) | tidak |
| Andal/berurutan | ya | tidak |
| Overhead | lebih besar | minimal |
| Cocok untuk | HTTP, SSH, DB, file transfer | DNS, video streaming, syslog, game |

**Catatan SRE:** DNS, syslog, dan metrik tertentu (StatsD) pakai UDP. Saat troubleshooting "DNS kadang lambat", ingat UDP bisa **drop tanpa retransmisi** — beda dengan TCP yang pasti coba lagi.

## 3. Port — Pintu di dalam Host

Satu IP punya 65 535 port TCP + 65 535 port UDP. Port < 1024 = privileged (butuh root untuk bind).

```bash
ss -tlnp                  # TCP, listening, numeric, proses
ss -ulnp                  # UDP listening
ss -tnp state established # koneksi aktif + proses pemilik
```

| Port terkenal | Service |
|---|---|
| 22 | SSH |
| 53 | DNS (TCP & UDP) |
| 80 / 443 | HTTP / HTTPS |
| 3306 / 5432 | MySQL / PostgreSQL |
| 6443 | Kubernetes API server |
| 10250 | kubelet |

**`ss` vs `netstat`:** `ss` lebih cepat & default di distro modern. `netstat` sudah deprecated.

## 4. Kenapa TCP/UDP Penting untuk Kubernetes

Service Kubernetes = IP virtual + port. Pod berkomunikasi via TCP (umumnya). Saat debugging "Pod tidak bisa konek Pod lain", urutannya:
1. `ss -tlnp` di target — apakah port benar-benar listening?
2. `0.0.0.0:PORT` vs `127.0.0.1:PORT` — `0.0.0.0` dengar di semua interface, `127.0.0.1` hanya localhost (tidak bisa diakses Pod lain).
3. NetworkPolicy / firewall blokir? (topik 04)

## 5. TLS — Enkripsi di Atas TCP

TLS (Transport Layer Security) = lapisan enkripsi di atas TCP. HTTPS = HTTP di atas TLS.

**Komponen:**
- **Sertifikat** — identitas server (nama + public key), ditandatangani CA.
- **CA (Certificate Authority)** — pihak yang dipercaya untuk menandatangani sertifikat.
- **Private key** — rahasia server; tidak pernah dikirim.
- **Handshake TLS** — negosiasi cipher, verifikasi sertifikat, tukar key.

```
Client                         Server
  │ ── ClientHello ──────────► │
  │ ◄── ServerHello + Cert ──── │
  │    (verifikasi cert & CA)   │
  │ ── KeyExchange ──────────► │
  │ ◄── Finished ────────────── │
  │ ═══ encrypted data ══════► │   ← kini aman
```

```bash
# Periksa sertifikat situs nyata:
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
# subject=CN=example.com
# issuer=C=... (CA)
# notBefore=... notAfter=...   ← tanggal kadaluarsa

# Cek rantai CA:
echo | openssl s_client -connect example.com:443 -showcerts 2>/dev/null | grep -E "^(s|i):"
```

## 6. Terminasi TLS — Di Mana Dekripsi Terjadi?

**TLS terminasi** = di mana HTTPS diubah jadi HTTP. Tiga pola:

```
(a) Server sendiri          (b) Reverse proxy terminasi   (c) End-to-end
Client ═TLS══════════ Server  Client ═TLS═══ Proxy ─HTTP─ Server  Client ═TLS═════════ Server
                             (proxy punya cert)              (proxy tidak baca isi)
```

| Pola | Kelebihan | Kekurangan |
|---|---|---|
| Server sendiri | sederhana | server repot dekripsi |
| Reverse proxy terminasi | server fokus app; satu titinjau sertifikat | traffic internal HTTP (butuh jaringan tepercaya) |
| End-to-end | aman sampai ujung | kompleks; semua node butuh cert |

**Di production on-prem**, reverse proxy (Caddy/Nginx/Traefik Ingress) biasanya terminasi TLS, lalu HTTP ke Pod. Ini akan dipraktikkan di [LAB-01](lab/LAB-01-reverse-proxy.md) dan jadi pola Ingress Kubernetes (Modul 2.1).

## 7. Sertifikat Internal (mTLS) — Pengantar

Di cluster, komponen bicara satu sama lain lewat **mTLS** (mutual TLS — kedua pihak tunjukkan sertifikat). Contoh: kubelet ↔ API server. Ini kenapa Kubernetes punya banyak sertifikat & ada insiden klasik "certificate expired → cluster mati" (akan dilatih di Game Day, Fase 9).

```bash
# Lihat sertifikat di file:
openssl x509 -in cert.pem -noout -text | grep -E "Subject:|Not After|DNS:"
```

## Latihan Cepat (10 menit)

```bash
# 1. Tangkap handshake TCP (butuh 2 terminal)
# Terminal 1:
sudo tcpdump -i any -n -c 15 'tcp port 80' &
# Terminal 2:
curl http://example.com >/dev/null
wait

# 2. Lihat state koneksi
ss -tn state established | head
ss -tn state time-wait | wc -l        # hitung TIME-WAIT

# 3. Bandingkan TCP vs UDP untuk DNS
dig @8.8.8.8 example.com              # default: UDP
dig +tcp @8.8.8.8 example.com         # paksa TCP

# 4. Periksa sertifikat
echo | openssl s_client -connect gitlab.com:443 -servername gitlab.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# 5. Cek port yang listening
ss -tlnp | head
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Lihat handshake | `tcpdump -n tcp port N` |
| Lihat koneksi aktif | `ss -tn state established` |
| Lihat port listening | `ss -tlnp` / `ss -ulnp` |
| Paksa DNS lewat TCP | `dig +tcp` |
| Periksa sertifikat | `openssl s_client` + `openssl x509` |
| Cek kadaluarsa cert | `openssl x509 -noout -dates` |

## Cek Pemahaman

1. Kenapa `0.0.0.0:8080` bisa diakses Pod lain tapi `127.0.0.1:8080` tidak?
2. Apa dampak banyak koneksi `TIME-WAIT` di server sibuk?
3. Kenapa DNS bisa "kadang lambat" padahal TCP biasanya andal?
4. Beda terminasi TLS di reverse proxy vs end-to-end — kapan masing-masing dipilih?