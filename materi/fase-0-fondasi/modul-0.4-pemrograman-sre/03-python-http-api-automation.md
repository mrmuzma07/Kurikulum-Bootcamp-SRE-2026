# 03 — Otomasi HTTP & REST API dengan Python untuk SRE

> **Ringkasan:** Membangun skrip otomasi jaringan dan REST API menggunakan Python yang tahan banting di lingkungan production: mengatur **timeout** wajib, menangani kode status HTTP (2xx/4xx/5xx), mengukur latensi respons, menerapkan *retry* dengan *backoff*, dan mengamankan *secret/token* lewat *environment variable*.

---

## 1. Konsep & Arsitektur: Bagaimana SRE Berinteraksi dengan Sistem via HTTP API

Di lingkungan produksi modern, hampir semua komponen yang kamu kelola berbicara melalui **HTTP/HTTPS REST API**:
- **Service Aplikasi:** mengekspos `/healthz` (apakah proses hidup?) dan `/readyz` (apakah siap menerima trafik?).
- **Prometheus & Mimir (Fase 7):** menerima query metrik di `/api/v1/query`.
- **Alertmanager & Webhook:** mengirim payload JSON saat ada insiden/alert yang menyala.
- **GitLab API (Fase 6):** memungkinkan kita memeriksa status pipeline, membuat release, atau mengaudit daftar Merge Request secara otomatis.

```text
┌─────────────────────────┐      HTTP GET / POST       ┌──────────────────────────┐
│   Skrip Otomasi SRE     │ ─────────────────────────► │ Target Endpoint / API    │
│  (Python + Timeout +    │   Header: Authorization    │ - /healthz (Service)     │
│   Retry + Error Check)  │ ◄───────────────────────── │ - /api/v1/query (Prom)   │
└─────────────────────────┘   Status 200 + JSON Body   └──────────────────────────┘
             │
             ▼
  Laporan Bukti (JSON / Exit Code 0 atau 1)
```

> ⚠️ **Aturan Emas SRE untuk Request Jaringan:**
> **DILARANG KERAS melakukan panggilan HTTP tanpa parameter `timeout`!**
> Secara default, banyak library HTTP (termasuk `requests.get(url)` di Python) memiliki `timeout=None` (menunggu selamanya). Jika jaringan ke server target menggantung (*packet drop* di tengah jalan setelah koneksi TCP terbuka), skrip cron atau job CI-mu akan **membeku selamanya (zombie process)**.

---

## 2. Parameter Kunci & Praktik Terbaik HTTP di Python

Kita dapat menggunakan library standar bawaan Python (`urllib.request`) atau library standar industri **`requests`**. Di bawah ini adalah parameter wajib saat menggunakan `requests`:

| Parameter / Fitur | Contoh Kode | Mengapa Wajib bagi SRE? |
|---|---|---|
| **`timeout=(connect, read)`** | `requests.get(url, timeout=(3.0, 5.0))` | Membatasi waktu jabat tangan TCP (3 detik) dan waktu tunggu respons data (5 detik) agar skrip tidak *hang*. |
| **`response.status_code`** | `if resp.status_code == 200:` | Membedakan sukses (`2xx`), salah request/auth (`4xx`), atau server tujuan tumbang (`5xx`). |
| **`response.elapsed`** | `resp.elapsed.total_seconds() * 1000` | Mengukur latensi HTTP dalam milidetik (fondasi SLI Latency di Fase 8). |
| **Token via Env Var** | `os.environ.get("GITLAB_TOKEN")` | Sesuai aturan **Secrets Hygiene**, token API tidak boleh di-hardcode di dalam file `.py`. |
| **Session & Retry** | `requests.Session()` + `HTTPAdapter(max_retries=...)` | Mengurangi *false alarm* akibat gangguan jaringan sesaat (*transient network blip*). |

---

## 3. Hands-On Step-by-Step

Mari kita praktikkan pembuatan *Health & SLO Latency Prober* menggunakan Python.

### 3.1 Instalasi `requests` di Virtual Environment
Pastikan kamu berada di dalam `.venv` yang dibuat sebelumnya:

```bash
cd ~/latihan-m0.4/python-dasar
source .venv/bin/activate

pip install requests==2.32.3
pip freeze > requirements.txt
```

### 3.2 Menjalankan Server Target Lokal untuk Simulasi
Agar latihan kita mandiri dan tidak bergantung pada internet luar, mari kita nyalakan server HTTP kecil di background menggunakan Python yang mensimulasikan 3 endpoint:
- `/healthz` → selalu sehat (`200 OK`, cepat)
- `/slow` → sehat tapi lambat (`200 OK`, tidur 1.5 detik)
- `/broken` → gagal (`503 Service Unavailable`)

```bash
cat > dummy_server.py <<'EOF'
#!/usr/bin/env python3
import json
import time
from http.server import BaseHTTPRequestHandler, HTTPServer


class SimHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/healthz":
            self._respond(200, {"status": "ok", "db": "connected"})
        elif self.path == "/slow":
            time.sleep(1.2)
            self._respond(200, {"status": "ok", "note": "lambat"})
        elif self.path == "/broken":
            self._respond(503, {"status": "error", "reason": "database connection pool exhausted"})
        else:
            self._respond(404, {"status": "not_found"})

    def _respond(self, code: int, payload: dict):
        body = json.dumps(payload).encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, format, *args):
        pass  # Matikan log bawaan agar terminal bersih


if __name__ == "__main__":
    server = HTTPServer(("127.0.0.1", 18080), SimHandler)
    print("Dummy server berjalan di http://127.0.0.1:18080")
    server.serve_forever()
EOF

# Jalankan di background dan simpan PID-nya
python dummy_server.py &
SERVER_PID=$!
sleep 1
curl -s http://127.0.0.1:18080/healthz && echo ""
```
*Output yang diharapkan:*
```text
Dummy server berjalan di http://127.0.0.1:18080
{"status": "ok", "db": "connected"}
```

### 3.3 Membuat Skrip Pemeriksa Endpoint (`http_prober.py`)
Sekarang kita tulis skrip Python yang memeriksa daftar endpoint, memvalidasi:
1. Apakah kode HTTP sesuai ekspektasi (`200`)?
2. Apakah waktu respons di bawah batas latensi maksimum (misal `500 ms`)?
3. Apakah field JSON tertentu bernilai benar?

```bash
cat > http_prober.py <<'EOF'
#!/usr/bin/env python3
"""
Prober HTTP sederhana ala Blackbox Exporter untuk memeriksa kesehatan dan latensi endpoint.
"""
import os
import sys
import time
import requests
from requests.exceptions import RequestException

ENDPOINTS = [
    {"name": "api-health", "url": "http://127.0.0.1:18080/healthz", "max_ms": 500},
    {"name": "api-slow",   "url": "http://127.0.0.1:18080/slow",    "max_ms": 500},
    {"name": "api-broken", "url": "http://127.0.0.1:18080/broken",  "max_ms": 500},
    {"name": "api-mati",   "url": "http://127.0.0.1:19999/healthz", "max_ms": 500},
]


def check_endpoint(ep: dict, token: str | None = None) -> tuple[bool, str]:
    headers = {"User-Agent": "SRE-Bootcamp-Prober/1.0"}
    if token:
        headers["Authorization"] = f"Bearer {token}"

    start = time.monotonic()
    try:
        # WAJIB: selalu sertakan timeout (connect_timeout, read_timeout)
        resp = requests.get(ep["url"], headers=headers, timeout=(2.0, 3.0))
        elapsed_ms = (time.monotonic() - start) * 1000
    except RequestException as exc:
        return False, f"[DOWN] {ep['name']} ({ep['url']}) -> Gagal koneksi/timeout: {exc.__class__.__name__}"

    if resp.status_code != 200:
        return False, f"[FAIL] {ep['name']} -> HTTP {resp.status_code} (ekspektasi 200) dalam {elapsed_ms:.1f}ms"

    if elapsed_ms > ep["max_ms"]:
        return False, f"[SLOW] {ep['name']} -> HTTP 200 tapi latensi {elapsed_ms:.1f}ms > batas {ep['max_ms']}ms"

    return True, f"[OK]   {ep['name']} -> HTTP 200 dalam {elapsed_ms:.1f}ms"


def main() -> int:
    # Mengambil token rahasia dari environment variable (jika ada), bukan hardcode!
    api_token = os.environ.get("PROBING_API_TOKEN")
    failures = 0

    print("=== Laporan Pemeriksaan Endpoint HTTP ===")
    for ep in ENDPOINTS:
        ok, message = check_endpoint(ep, token=api_token)
        print(message)
        if not ok:
            failures += 1

    print(f"=== Ringkasan: {len(ENDPOINTS) - failures}/{len(ENDPOINTS)} sehat ===")
    return 1 if failures > 0 else 0


if __name__ == "__main__":
    sys.exit(main())
EOF

chmod +x http_prober.py
./http_prober.py || echo "Exit code: $?"
```
*Output yang diharapkan:*
```text
=== Laporan Pemeriksaan Endpoint HTTP ===
[OK]   api-health -> HTTP 200 dalam 2.4ms
[SLOW] api-slow -> HTTP 200 tapi latensi 1203.5ms > batas 500ms
[FAIL] api-broken -> HTTP 503 (ekspektasi 200) dalam 1.8ms
[DOWN] api-mati (http://127.0.0.1:19999/healthz) -> Gagal koneksi/timeout: ConnectionError
=== Ringkasan: 1/4 sehat ===
Exit code: 1
```

Setelah selesai mencoba, matikan server simulasi background:
```bash
kill "$SERVER_PID" 2>/dev/null || true
```

---

## 4. Pola Kegagalan (Failure Mode) & Diagnosa

1. **Request Menggantung Tanpa Batas Waktu (`Missing Timeout`):**
   - **Gejala:** Skrip pengecekan yang dijalankan lewat systemd timer atau cron setiap 5 menit menumpuk puluhan proses `python` di `ps aux` karena server tujuan mengalami *firewall drop*.
   - **Diagnosa & Perbaikan:** Cari semua panggilan `requests.get/post` dan pastikan memiliki argumen `timeout=(connect_sec, read_sec)`.
2. **Retry Storm (Badai Percobaan Ulang Tanpa Jeda):**
   - **Gejala:** Ketika database atau API internal sedang kelebihan beban (*overload*), skrip otomasi langsung melakukan retry di dalam `while True:` tanpa `time.sleep()`. Ini justru membuat layanan yang sedang sekarat mati total (*Self-Inflicted DDoS*).
   - **Perbaikan:** Batasi jumlah retry maksimal (misal `3 kali`) dan gunakan *exponential backoff* (tunggu 1 detik, lalu 2 detik, lalu 4 detik).
3. **Menampilkan Token/Password di Log Saat Error:**
   - **Gejala:** Mencetak seluruh isi `headers` ke log terminal saat terjadi error HTTP, sehingga `Authorization: Bearer <TOKEN>` bocor ke sistem log atau output CI/CD.
   - **Perbaikan:** Jangan pernah mencetak nilai header `Authorization` atau variabel secret ke stdout/stderr.

---

## 5. Ringkasan & Langkah Selanjutnya

- Selalu pasang `timeout` pada setiap operasi jaringan di Python.
- Tangkap `requests.exceptions.RequestException` agar kegagalan jaringan (DNS gagal, koneksi ditolak, timeout) berubah menjadi pesan diagnostik yang rapi, bukan *stack trace* mentah.
- Ambil kredensial/token dari `os.environ.get()`, bukan menulisnya di dalam kode.
- Selanjutnya di [04-go-microservice-dasar.md](04-go-microservice-dasar.md), kita akan beralih ke **Go (Golang)** untuk melihat sisi sebaliknya: bagaimana membangun service HTTP yang ringan, memiliki `/healthz`, `/readyz`, `/metrics`, dan siap di-containerize di Fase 1.
