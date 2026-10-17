# Latihan — Modul 0.4 Pemrograman untuk SRE (Python & Go Dasar)

Kerjakan seluruh latihan di terminal VM `lab01` (atau macOS). Target: membangun *muscle memory* menulis skrip Python dan memodifikasi kode Go tanpa copy-paste buta.

> **Aturan:** Catat perintah penting dan hasil pengamatanmu di `m0.4/lab/log-latihan-m0.4.md` di dalam repo `sre-bootcamp`.

---

## Hari 1 — Keputusan Bahasa, Python `venv`, Struktur Data, & YAML

### 1.1 Isolasi Environment & Dependency Pinning
```bash
mkdir -p ~/latihan-m0.4/hari1 && cd ~/latihan-m0.4/hari1

# 1. Buat dan aktifkan virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 2. Cek lokasi biner python & pip
which python
python --version

# 3. Instal PyYAML dan kunci versinya ke requirements.txt
pip install pyyaml==6.0.2
pip freeze > requirements.txt
```
Catat di log latihan: apa perbedaan path `which python` sebelum dan sesudah menjalankan `source .venv/bin/activate`?

### 1.2 Filter & Transformasi Inventory JSON
```bash
cat > pods.json <<'EOF'
{
  "namespace": "production",
  "items": [
    {"name": "web-01", "restarts": 0, "status": "Running", "memory_mi": 180},
    {"name": "web-02", "restarts": 7, "status": "CrashLoopBackOff", "memory_mi": 512},
    {"name": "worker-01", "restarts": 2, "status": "Running", "memory_mi": 240},
    {"name": "db-proxy-01", "restarts": 12, "status": "Running", "memory_mi": 490}
  ]
}
EOF

# Tulis skrip filter_pods.py yang mencetak pod dengan restarts >= 5 ATAU status != "Running"
cat > filter_pods.py <<'EOF'
#!/usr/bin/env python3
import json
import sys
from pathlib import Path

data = json.loads(Path("pods.json").read_text(encoding="utf-8"))
problem_pods = [
    p for p in data.get("items", [])
    if p.get("restarts", 0) >= 5 or p.get("status") != "Running"
]

for pod in problem_pods:
    print(f"[ALERT] Pod={pod['name']} status={pod['status']} restarts={pod['restarts']}")

sys.exit(1 if problem_pods else 0)
EOF

python filter_pods.py || echo "Exit code: $?"
```
Modifikasi `filter_pods.py` agar juga menerima argumen `--max-restarts` menggunakan `argparse` (default: `5`).

### 1.3 Konversi YAML ke JSON & Sebaliknya
```bash
python -c '
import json, yaml
from pathlib import Path

data = json.loads(Path("pods.json").read_text())
Path("pods.yaml").write_text(yaml.safe_dump(data, sort_keys=False))
print(Path("pods.yaml").read_text())
'
```

---

## Hari 2 — Otomasi HTTP API, Timeout, & Error Handling

### 2.1 Eksperimen Timeout vs Tanpa Timeout
```bash
cd ~/latihan-m0.4/hari1
source .venv/bin/activate
pip install requests==2.32.3

# Buat server kecil yang sengaja tidur 4 detik sebelum menjawab
python -c '
import time
from http.server import BaseHTTPRequestHandler, HTTPServer
class Slow(BaseHTTPRequestHandler):
    def do_GET(self):
        time.sleep(4)
        self.send_response(200); self.end_headers(); self.wfile.write(b"lambat\n")
    def log_message(self, *a): pass
HTTPServer(("127.0.0.1", 18090), Slow).serve_forever()
' &
SLOW_PID=$!
sleep 1

# Uji request dengan timeout=1.5 detik (harus gagal ReadTimeout dalam 1.5 detik)
python -c '
import requests
from requests.exceptions import Timeout
try:
    requests.get("http://127.0.0.1:18090", timeout=(1.0, 1.5))
except Timeout as e:
    print("[TERTANGKAP] Request dibatalkan tepat waktu karena melewati batas timeout:", type(e).__name__)
'

kill "$SLOW_PID" 2>/dev/null || true
```

### 2.2 Membaca Secret dari Environment Variable (Secrets Hygiene)
```bash
# Jalankan skrip yang wajib memeriksa keberadaan env var API_TOKEN
API_TOKEN="simulasi-token-rahasia" python -c '
import os, sys
token = os.environ.get("API_TOKEN")
if not token:
    print("[ERROR] Environment variable API_TOKEN belum di-set!", file=sys.stderr)
    sys.exit(2)
print(f"[OK] Token terdeteksi (panjang={len(token)} karakter, nilai disembunyikan)")
'
```

---

## Hari 3 — Microservice Go, Unit Test, & Cross-Compilation

### 3.1 Menambahkan Endpoint `/version` pada Aplikasi Go
Di direktori `m0.4/app-go` (dari `LAB-02`):
1. Tambahkan handler baru `/version` pada fungsi `setupRouter` di `main.go` yang mengembalikan JSON `{"version": version, "go_arch": runtime.GOARCH}` (gunakan paket `"runtime"`).
2. Tambahkan test case untuk `/version` di dalam `main_test.go`.
3. Jalankan `go test -v ./...` sampai lulus.

### 3.2 Membandingkan Ukuran Biner dengan dan Tanpa `-ldflags="-s -w"`
```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp/m0.4/app-go

CGO_ENABLED=0 go build -o dist/app-normal .
CGO_ENABLED=0 go build -ldflags="-s -w" -o dist/app-stripped .

ls -lh dist/app-normal dist/app-stripped
```
Catat berapa MB selisih ukuran antara `app-normal` dan `app-stripped`.

---

## Soal Refleksi (Tulis di `m0.4/lab/log-latihan-m0.4.md`)

1. Mengapa SRE sangat disarankan beralih dari Bash ke Python ketika skrip sudah melibatkan parsing JSON/YAML bercabang dan panggilan HTTP API dengan logika *retry*?
2. Apa bahaya nyata di server produksi jika sebuah skrip Python yang dijalankan via cron setiap menit memanggil `requests.get(url)` **tanpa** argumen `timeout`?
3. Apa perbedaan fungsi antara endpoint `/healthz` (*liveness*) dan `/readyz` (*readiness*) saat aplikasi Go kita nanti dijalankan di dalam Pod Kubernetes (Fase 2)?
4. Mengapa kita menyertakan `CGO_ENABLED=0` saat mengompilasi aplikasi Go yang akan dimasukkan ke dalam image container `distroless` atau `alpine`?

---

## Catatan Performa

- [ ] Semua latihan Hari 1–3 dikerjakan langsung di terminal
- [ ] Log latihan dan jawaban refleksi tersimpan di `m0.4/lab/log-latihan-m0.4.md`
- [ ] Bisa membuat `.venv`, menginstal library, dan memvalidasi YAML tanpa melihat contekan
- [ ] Bisa menjelaskan arti setiap baris pada handler `/healthz`, `/readyz`, `/metrics`, dan `SIGTERM` di `main.go`
