# LAB-01 — Membuat CLI Auditor Konfigurasi & Endpoint HTTP dengan Python

> **Target:** Membangun tool CLI Python bernama `sre-audit` di dalam repo `sre-bootcamp` yang membaca daftar layanan dari file `endpoints.yaml`, memvalidasi aturan konfigurasi produksi, memeriksa kesehatan HTTP setiap endpoint beserta batas latensinya (SLO), dan menghasilkan laporan bukti berformat JSON (`audit-report.json`) dengan *exit code* yang akurat.

## Latar Belakang & Skenario Nyata
Di lingkungan produksi on-premises, sebelum tim SRE menyatakan sebuah perubahan atau rilis selesai (*Production Readiness Check*), kita harus mengumpulkan **bukti validasi (Evidence-Driven Operations)**:
1. Apakah konfigurasi YAML layanan sudah mematuhi aturan (tidak memakai tag `:latest`, memiliki minimal 2 replika untuk High Availability)?
2. Apakah endpoint kesehatan (`/healthz`) merespons `HTTP 200` di bawah batas waktu (misalnya `< 300 ms`)?

Daripada mengeceknya satu per satu secara manual dengan `curl`, kamu ditugaskan membuat skrip CLI otomatis `m0.4/audit-tool/audit.py` yang nantinya bisa dipanggil kapan saja oleh manusia maupun oleh GitLab CI.

## Prasyarat
- [ ] Sudah membaca materi [01](../01-kapan-bash-vs-python-vs-go.md), [02](../02-python-dasar-json-yaml-cli.md), dan [03](../03-python-http-api-automation.md)
- [ ] Repo lokal `sre-bootcamp` dari Modul 0.3 siap digunakan
- [ ] Paket `python3`, `python3-venv`, dan `jq` sudah terpasang

## Estimasi Waktu
± 60–90 menit

---

## Langkah Pelaksanaan

### 1. Buat Branch Kerja & Struktur Direktori `m0.4/audit-tool`
Masuk ke repo `sre-bootcamp` milikmu dan buat branch baru:

```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp   # sesuaikan dengan path repo sre-bootcamp kamu
git switch main
git pull
git switch -c feat/m0.4-lab01-python-audit

mkdir -p m0.4/audit-tool
cd m0.4/audit-tool
```

Pastikan `.venv/` sudah terdaftar di `.gitignore` root repo agar virtual environment tidak ikut ter-commit:

```bash
# Cek apakah .venv/ sudah ada di .gitignore root; jika belum, tambahkan:
grep -qxF '.venv/' ../../.gitignore 2>/dev/null || echo '.venv/' >> ../../.gitignore
grep -qxF '__pycache__/' ../../.gitignore 2>/dev/null || echo '__pycache__/' >> ../../.gitignore
```

*Verifikasi:*
```bash
git status --short
```

---

### 2. Siapkan Virtual Environment & `requirements.txt`
```bash
python3 -m venv .venv
source .venv/bin/activate

pip install pyyaml==6.0.2 requests==2.32.3
pip freeze > requirements.txt
```

*Verifikasi:*
```bash
cat requirements.txt
# Harus menampilkan PyYAML dan requests beserta dependensinya
```

---

### 3. Buat File Konfigurasi `endpoints.yaml`
Buat file konfigurasi yang mendefinisikan daftar service yang akan diaudit:

```bash
cat > endpoints.yaml <<'EOF'
cluster: onprem-lab01
max_latency_ms_default: 300
services:
  - name: caddy-local
    image: caddy:2.8-alpine
    replicas: 2
    url: http://127.0.0.1:18081/healthz
    expected_status: 200
    max_latency_ms: 250

  - name: order-api
    image: registry.gitlab.com/sre-bootcamp/order-api:v1.2.0
    replicas: 2
    url: http://127.0.0.1:18081/readyz
    expected_status: 200
    max_latency_ms: 250
EOF
```

---

### 4. Tulis Skrip CLI `audit.py`
Buat skrip `audit.py` yang menggabungkan validasi konfigurasi YAML dan pemeriksaan HTTP live dengan `timeout`:

```bash
cat > audit.py <<'EOF'
#!/usr/bin/env python3
"""
LAB-01 Modul 0.4 — SRE Configuration & HTTP Endpoint Auditor.
"""
import argparse
import json
import logging
import sys
import time
from datetime import datetime, timezone
from pathlib import Path
import requests
from requests.exceptions import RequestException
import yaml

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)
log = logging.getLogger("sre-audit")


def validate_config(svc: dict) -> list[str]:
    """Memvalidasi aturan statis konfigurasi service."""
    issues = []
    name = svc.get("name", "unknown")
    image = str(svc.get("image", ""))
    replicas = svc.get("replicas", 0)

    if not image or ":" not in image or image.endswith(":latest"):
        issues.append(f"{name}: image '{image}' tidak boleh memakai tag ':latest' atau tanpa versi")

    if not isinstance(replicas, int) or replicas < 2:
        issues.append(f"{name}: replicas={replicas} kurang dari standar HA (minimal 2)")

    if not svc.get("url"):
        issues.append(f"{name}: field 'url' wajib diisi")

    return issues


def probe_http(svc: dict, default_max_ms: int) -> dict:
    """Melakukan pemeriksaan HTTP GET ke endpoint dengan timeout."""
    url = svc["url"]
    expected = int(svc.get("expected_status", 200))
    max_ms = int(svc.get("max_latency_ms", default_max_ms))

    start = time.monotonic()
    try:
        resp = requests.get(url, timeout=(2.0, 3.0))
        latency_ms = round((time.monotonic() - start) * 1000, 2)
    except RequestException as exc:
        return {
            "reachable": False,
            "status_code": None,
            "latency_ms": None,
            "healthy": False,
            "reason": f"connection_error: {exc.__class__.__name__}",
        }

    status_ok = resp.status_code == expected
    latency_ok = latency_ms <= max_ms
    healthy = status_ok and latency_ok

    reason = "ok"
    if not status_ok:
        reason = f"expected HTTP {expected}, got {resp.status_code}"
    elif not latency_ok:
        reason = f"latency {latency_ms}ms exceeds limit {max_ms}ms"

    return {
        "reachable": True,
        "status_code": resp.status_code,
        "latency_ms": latency_ms,
        "healthy": healthy,
        "reason": reason,
    }


def main() -> int:
    parser = argparse.ArgumentParser(description="SRE Config & HTTP Endpoint Auditor")
    parser.add_argument("-f", "--file", required=True, help="Path ke file YAML inventaris endpoint")
    parser.add_argument("-o", "--output", default="audit-report.json", help="Path file output JSON")
    parser.add_argument("--skip-http", action="store_true", help="Hanya validasi konfigurasi YAML tanpa request HTTP")
    args = parser.parse_args()

    cfg_path = Path(args.file)
    if not cfg_path.is_file():
        log.error("File konfigurasi tidak ditemukan: %s", cfg_path)
        return 2

    data = yaml.safe_load(cfg_path.read_text(encoding="utf-8"))
    cluster = data.get("cluster", "unknown")
    default_max_ms = int(data.get("max_latency_ms_default", 300))
    services = data.get("services", [])

    results = []
    overall_passed = True

    for svc in services:
        name = svc.get("name", "unknown")
        cfg_issues = validate_config(svc)
        http_result = None

        if not args.skip_http and svc.get("url"):
            http_result = probe_http(svc, default_max_ms)

        svc_passed = len(cfg_issues) == 0 and (args.skip_http or (http_result is not None and http_result["healthy"]))
        if not svc_passed:
            overall_passed = False

        log.info(
            "Service=%s | config_ok=%s | http_ok=%s",
            name,
            len(cfg_issues) == 0,
            "SKIPPED" if args.skip_http else (http_result["healthy"] if http_result else False),
        )

        results.append({
            "name": name,
            "passed": svc_passed,
            "config_issues": cfg_issues,
            "http_probe": http_result,
        })

    report = {
        "timestamp_utc": datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"),
        "cluster": cluster,
        "overall_passed": overall_passed,
        "results": results,
    }

    out_path = Path(args.output)
    out_path.write_text(json.dumps(report, indent=2) + "\n", encoding="utf-8")
    log.info("Laporan disimpan ke %s (overall_passed=%s)", out_path, overall_passed)

    return 0 if overall_passed else 1


if __name__ == "__main__":
    sys.exit(main())
EOF

chmod +x audit.py
```

---

### 5. Uji Jalankan Skrip pada Kondisi Gagal vs Kondisi Sehat

1. **Uji pertama (Kondisi Server Tujuan Belum Nyala → Harus Gagal dengan Exit Code 1):**
   ```bash
   ./audit.py --file endpoints.yaml --output audit-report.json || echo "Exit code: $?"
   jq . audit-report.json
   ```
   Pastikan `overall_passed` bernilai `false` dan `Exit code: 1`.

2. **Uji kedua (Nyalakan Web Server Lokal di Port `18081` → Harus Lulus dengan Exit Code 0):**
   ```bash
   # Jalankan server HTTP mini di background pada port 18081
   python3 -c '
   from http.server import BaseHTTPRequestHandler, HTTPServer
   class H(BaseHTTPRequestHandler):
       def do_GET(self):
           self.send_response(200); self.end_headers(); self.wfile.write(b"ok\n")
       def log_message(self, *a): pass
   HTTPServer(("127.0.0.1", 18081), H).serve_forever()
   ' &
   TEST_PID=$!
   sleep 1

   # Jalankan kembali auditor
   ./audit.py --file endpoints.yaml --output audit-report.json
   echo "Exit code: $?"
   jq . audit-report.json

   # Matikan server uji setelah selesai
   kill "$TEST_PID" 2>/dev/null || true
   ```

*Verifikasi:*
```bash
jq '.overall_passed' audit-report.json
# Output wajib: true
```

---

### 6. Commit & Push ke GitLab
```bash
git add endpoints.yaml requirements.txt audit.py audit-report.json ../../.gitignore
git status
git commit -m "feat(m0.4): tambah python cli auditor konfigurasi dan http endpoint"
git push -u origin feat/m0.4-lab01-python-audit
```

---

## Acceptance Criteria (Kriteria Keberhasilan)
- [ ] Direktori `.venv/` tidak masuk ke dalam `git status` (sudah di-ignore di `.gitignore`).
- [ ] `./audit.py --help` menampilkan dokumentasi argumen `--file`, `--output`, dan `--skip-http`.
- [ ] Saat endpoint mati atau image menggunakan tag `:latest`, `./audit.py` mengembalikan *exit code* `1` dan mencatat alasannya di `audit-report.json`.
- [ ] Saat semua aturan konfigurasi terpenuhi dan endpoint merespons `HTTP 200` di bawah batas latensi, `./audit.py` mengembalikan *exit code* `0` dan `"overall_passed": true`.

## Troubleshooting & Tips
- **`ModuleNotFoundError: No module named 'yaml'`:** Kamu belum mengaktifkan virtual environment (`source .venv/bin/activate`) atau menjalankan skrip menggunakan `/usr/bin/python3` alih-alih `python` dari dalam `.venv`.
- **Port `18081` sudah terpakai (`Address already in use`):** Cari proses yang memakai port tersebut dengan `ss -tulpn | grep 18081` atau `lsof -i :18081` lalu hentikan dengan `kill`.
