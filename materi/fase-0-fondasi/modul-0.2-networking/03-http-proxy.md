# 03 — HTTP/HTTPS, Reverse Proxy & Caddy

> Request-response, status code, header, dan kenapa ada proxy di depan aplikasi.

## Tujuan
- Paham anatomi HTTP request/response & status code
- Bisa membaca header penting saat troubleshooting
- Mengerti peran reverse proxy & bisa mengkonfigurasi Caddy
- Bisa menjelaskan routing/host-based routing (bekal Ingress)

## 1. Anatomi HTTP

HTTP = teks, request-response, stateless. Satu request = satu response.

**Request:**
```
GET /api/health HTTP/1.1
Host: app.example.com
User-Agent: curl/8.4
Accept: application/json
```
Baris 1 = method + path + version. Berikutnya = header. Lalu baris kosong + body (untuk POST/PUT).

**Response:**
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 17

{"status":"ok"}
```

```bash
# Lihat seluruh percakapan:
curl -v http://localhost:8080/         # -v = verbose: header request & response
curl -s -D - -o /dev/null http://localhost:8080/   # header saja, body dibuang
```

## 2. Method

| Method | Sifat | Contoh |
|---|---|---|
| `GET` | baca, aman/idempotent | `GET /users/42` |
| `POST` | buat, tidak idempotent | `POST /users` |
| `PUT` | ganti seluruh, idempotent | `PUT /users/42` |
| `PATCH` | ubah sebagian | `PATCH /users/42` |
| `DELETE` | hapus, idempotent | `DELETE /users/42` |

**Idempotent** = request yang sama diulang memberi hasil sama. Penting saat retry: `POST` dua kali bisa bikin dua data, `PUT` dua kali aman.

## 3. Status Code — Bahasa Error Web

| Range | Arti | Contoh |
|---|---|---|
| `1xx` | informational | `101 Switching Protocols` (WebSocket) |
| `2xx` | sukses | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | redirect | `301 Moved`, `302 Found`, `304 Not Modified` |
| `4xx` | salah client | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests` |
| `5xx` | salah server | `500 Internal Error`, `502 Bad Gateway`, `503 Service Unavailable`, `504 Gateway Timeout` |

**Sering tertukar:**
- `401 Unauthorized` = "siapa kamu?" (belum login / token salah)
- `403 Forbidden` = "kamu siapa tahu, tapi tidak boleh" (login OK, tidak punya izin)

**`502` vs `503` vs `504`** — paling sering muncul di on-call:
- `502 Bad Gateway` = proxy tidak dapat respons valid dari upstream (upstream mati/crash)
- `503 Service Unavailable` = service sengaja menolak (overload/maintenance)
- `504 Gateway Timeout` = upstream tidak menjawab dalam batas waktu

```bash
# Paksa method tertentu:
curl -X POST -d '{"name":"ops"}' -H 'Content-Type: application/json' http://localhost:8080/users
```

## 4. Header Penting untuk SRE

| Header | Fungsi |
|---|---|
| `Host` | nama virtual host (base routing di reverse proxy) |
| `Content-Type` | tipe body (`application/json`, `text/html`) |
| `Content-Length` | ukuran body |
| `Authorization` | kredensial (Bearer token, Basic) |
| `X-Forwarded-For` | IP client asli (setelah lewat proxy) |
| `X-Forwarded-Proto` | protokol asli (`https`) sebelum terminasi TLS |
| `User-Agent` | siapa pemanggil (curl, browser, Prometheus) |
| `Connection: keep-alive` | tetap buka koneksi untuk request berikutnya |

```bash
curl -v -H 'Host: app.lab.local' http://localhost/        # spoof Host (uji virtual host)
curl -H 'X-Forwarded-For: 10.0.0.5' http://localhost/
```

**`X-Forwarded-*`** krusial: setelah reverse proxy terminasi TLS, app melihat koneksi datang dari IP proxy, bukan client. App butuh header ini tahu IP client asli (untuk rate-limiting, audit log).

## 5. Reverse Proxy vs Forward Proxy

```
Forward Proxy (keluar)              Reverse Proxy (masuk)
Client ──► [Proxy] ──► Internet    Internet ──► [Proxy] ──► Server
(sembunyikan identitas client)      (sembunyikan identitas/arsitektur server)
```

SRE hampir selalu bicara **reverse proxy**. Tugasnya:
- Terminasi TLS (satu titinjau sertifikat)
- Routing berdasarkan `Host`/path → banyak app di port 80/443
- Load balancing (distribusi ke beberapa upstream)
- Rate limiting, caching, kompresi
- Menyembunyikan app langsung dari internet

**Di Kubernetes** peran ini dipegang **Ingress** (Traefik bawaan k3s / Nginx Ingress). Jadi belajar reverse proxy sekarang = belajar mental model Ingress nanti.

## 6. Caddy — Reverse Proxy Termudah

Caddy dipilih karena: konfigurasi singkat, HTTPS otomatis (Let's Encrypt), native ARM64.

```bash
# Install di VM lab01 (Ubuntu, ARM64):
sudo apt install -y caddy
# atau manual:
curl -fsSL 'https://caddyserver.com/api/download?os=linux&arch=arm64' -o /usr/local/bin/caddy
chmod +x /usr/local/bin/caddy
```

**Caddyfile minimal** — serve statis + reverse proxy:
```caddyfile
# /etc/caddy/Caddyfile

app.lab.local {
    reverse_proxy localhost:8080
}

grafana.lab.local {
    reverse_proxy localhost:3000
}
```

```bash
sudo systemctl reload caddy
sudo systemctl status caddy
```

Caddy otomatis minta sertifikat untuk nama domain publik. Untuk nama `.local` / internal, pakai TLS internal (lihat [LAB-01](lab/LAB-01-reverse-proxy.md)).

## 7. Host-Based Routing (Bekal Ingress)

Satu reverse proxy, banyak situs, berdasarkan `Host`:

```caddyfile
monitor.lab.local {
    # /api → service API
    handle /api/* {
        reverse_proxy localhost:8080
    }
    # sisanya → UI
    handle {
        reverse_proxy localhost:3000
    }
}
```

Inilah persis cara Ingress Kubernetes bekerja: satu IP, satu port 443, banyak service dibedakan oleh `Host` header & path.

## Latihan Cepat (15 menit)

```bash
# 1. Jalankan server uji sederhana
python3 -m http.server 8080 &      # di VM lab01
curl -v http://localhost:8080/

# 2. Lihat semua header
curl -s -D - -o /dev/null http://localhost:8080/

# 3. Spoof Host untuk simulasi virtual host
curl -H 'Host: app.lab.local' http://localhost:8080/

# 4. Inspeksi status code saja
curl -o /dev/null -s -w "%{http_code}\n" http://localhost:8080/

# 5. (lanjutan) Konfigurasi Caddyfile sederhana — lihat LAB-01

# 6. Matikan server uji
kill %1
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Lihat request+response | `curl -v` |
| Header saja | `curl -s -D - -o /dev/null` |
| Status code saja | `curl -o /dev/null -s -w '%{http_code}\n'` |
| Kirim body JSON | `curl -X POST -d '{...}' -H 'Content-Type: application/json'` |
| Reverse proxy termudah | Caddy + Caddyfile |
| Routing multi-situs | bedakan oleh `Host` header |

## Cek Pemahaman

1. Beda `401` vs `403` — dan kenapa tidak boleh tertukar di log?
2. `502` vs `504` — apa yang berbeda dari sisi SRE saat troubleshooting?
3. Kenapa app di belakang reverse proxy tidak melihat IP client asli tanpa `X-Forwarded-For`?
4. Bagaimana satu reverse proxy melayani 5 situs berbeda hanya di port 443?