# 02 — Python Dasar untuk SRE: Struktur Data, JSON/YAML, & CLI

> **Ringkasan:** Menguasai fondasi Python yang paling sering dipakai SRE di dunia nyata: mengisolasi environment dengan `venv`, memanipulasi *dictionary* dan *list*, memvalidasi file konfigurasi **JSON/YAML**, membuat CLI tool dengan `argparse`, serta menerapkan *structured logging* dan *exit code* yang benar.

---

## 1. Konsep & Arsitektur: Mengapa SRE Wajib Menguasai `dict`, `list`, & YAML?

Dalam keseharian SRE, hampir semua konfigurasi infrastruktur (Kubernetes Manifest, Ansible Playbook, Helm `values.yaml`, Docker Compose, hingga konfigurasi Prometheus & Grafana Alloy) ditulis dalam format **YAML** atau **JSON**.

Ketika sebuah file YAML atau JSON dibaca ke dalam memori oleh Python (maupun oleh Ansible):
- Objek berpasangan `key: value` (`{...}`) otomatis menjadi **`dict` (Dictionary)** di Python.
- Daftar berurutan `- item` (`[...]`) otomatis menjadi **`list` (List)** di Python.

```text
File YAML / JSON (di Disk)                Struktur Data Python (di Memori)
──────────────────────────                ────────────────────────────────
service: "payment-api"          ───►      data["service"]          (str)
replicas: 3                     ───►      data["replicas"]         (int)
ports:                          ───►      data["ports"]            (list)
  - 8080                                  data["ports"][0]         (int)
  - 9090
resources:                      ───►      data["resources"]        (dict)
  memory_mb: 512                          data["resources"]["memory_mb"]
```

Memahami cara membaca, memfilter, dan memodifikasi `dict` dan `list` di Python berarti kamu otomatis memahami cara kerja mesin internal **Ansible** (Fase 4) dan **Kubernetes API**.

---

## 2. Parameter Kunci & Praktik Terbaik Python untuk SRE

| Komponen / Modul | Fungsi Operasional SRE | Aturan Best Practice |
|---|---|---|
| `python3 -m venv .venv` | Membuat ruang isolasi paket Python per proyek | **Wajib.** Jangan pernah `sudo pip install` ke sistem OS Linux. |
| `dict.get("key", default)` | Mengambil nilai dari dictionary dengan aman | Gunakan `.get()` dibanding `data["key"]` agar skrip tidak crash (`KeyError`) jika field opsional kosong. |
| `pathlib.Path` | Manipulasi path file & direktori lintas OS | Lebih bersih dan aman dibanding merangkai string path dengan `+`. |
| `json` & `yaml` (`PyYAML`) | Membaca & menulis konfigurasi JSON/YAML | Selalu gunakan `yaml.safe_load()` (jangan `yaml.load()` karena celah keamanan eksekusi kode). |
| `argparse` | Menerima argumen CLI (`--file`, `--env`, `--dry-run`) | Setiap skrip operasional wajib punya `--help` yang jelas. |
| `logging` & `sys.exit()` | Mencetak log berlevel (`INFO`, `WARN`, `ERROR`) & kode keluar | Tulis error ke `stderr` dan kembalikan `sys.exit(1)` saat gagal agar pipeline GitLab CI tahu job gagal. |

---

## 3. Hands-On Step-by-Step

Mari kita praktikkan langsung di terminal VM `lab01` (atau macOS).

### 3.1 Membuat Virtual Environment (`venv`) & Menginstal `PyYAML`
Di Ubuntu 22.04/24.04 dan macOS modern, menginstal paket `pip` langsung ke sistem OS akan ditolak demi melindungi stabilitas OS. Kita wajib membuat `.venv`:

```bash
mkdir -p ~/latihan-m0.4/python-dasar && cd ~/latihan-m0.4/python-dasar

# 1. Buat virtual environment bernama .venv
python3 -m venv .venv

# 2. Aktifkan virtual environment
source .venv/bin/activate

# 3. Pastikan pip mengarah ke dalam .venv (bukan /usr/bin/pip)
which python
which pip

# 4. Instal PyYAML dan simpan ke requirements.txt
pip install pyyaml==6.0.2
pip freeze > requirements.txt
cat requirements.txt
```
*Output yang diharapkan:*
```text
/home/ubuntu/latihan-m0.4/python-dasar/.venv/bin/python
/home/ubuntu/latihan-m0.4/python-dasar/.venv/bin/pip
PyYAML==6.0.2
```
> **Penting:** Folder `.venv/` berisi ribuan file library. **Jangan pernah** di-commit ke Git! Selalu masukkan `.venv/` ke dalam `.gitignore`, dan cukup commit file `requirements.txt`.

### 3.2 Mengolah Variabel, `list`, `dict`, dan Exception Handling
Buat skrip `01_struktur_data.py` untuk melatih cara SRE memeriksa inventaris layanan:

```bash
cat > 01_struktur_data.py <<'EOF'
#!/usr/bin/env python3
"""Latihan manipulasi list dan dictionary untuk inventaris service SRE."""

services = [
    {"name": "auth-svc", "replicas": 2, "resources": {"cpu_m": 250, "mem_mi": 256}},
    {"name": "order-svc", "replicas": 1, "resources": {"cpu_m": 500, "mem_mi": 512}},
    {"name": "legacy-worker", "replicas": 3},  # Sengaja tidak punya key 'resources'
]

print("=== Audit Konfigurasi Service ===")
for svc in services:
    name = svc.get("name", "unknown")
    replicas = svc.get("replicas", 1)

    # Aman mengambil nested dict menggunakan .get() berantai
    resources = svc.get("resources", {})
    mem_mi = resources.get("mem_mi")

    if mem_mi is None:
        print(f"[WARN] {name}: belum memiliki batas memory (resources.mem_mi)!")
    elif replicas < 2:
        print(f"[WARN] {name}: replicas={replicas} (SPOF - Single Point of Failure), mem={mem_mi}Mi")
    else:
        print(f"[OK]   {name}: replicas={replicas}, mem={mem_mi}Mi")
EOF

python 01_struktur_data.py
```
*Output yang diharapkan:*
```text
=== Audit Konfigurasi Service ===
[OK]   auth-svc: replicas=2, mem=256Mi
[WARN] order-svc: replicas=1 (SPOF - Single Point of Failure), mem=512Mi
[WARN] legacy-worker: belum memiliki batas memory (resources.mem_mi)!
```

### 3.3 Membaca, Memvalidasi, & Menulis File YAML / JSON
Salah satu tugas nyata SRE di CI/CD pipeline (Fase 6) adalah memvalidasi apakah file konfigurasi deployment sudah memenuhi standar produksi (misalnya: wajib punya `livenessProbe`, `resources.limits`, dan dilarang memakai tag image `:latest`).

Mari kita buat file `deployments.yaml` dan skrip validatornya:

```bash
cat > deployments.yaml <<'EOF'
environment: production
services:
  - name: frontend-web
    image: registry.gitlab.com/sre/frontend:v1.4.2
    replicas: 2
    healthcheck_path: /healthz
    memory_limit_mi: 256
  - name: payment-worker
    image: registry.gitlab.com/sre/payment:latest
    replicas: 1
    memory_limit_mi: 512
  - name: reporting-api
    image: registry.gitlab.com/sre/reporting:v2.0.1
    replicas: 2
    healthcheck_path: /healthz
EOF
```

Sekarang buat skrip CLI `audit_yaml.py` lengkap dengan `argparse`, `logging`, dan output laporan JSON:

```bash
cat > audit_yaml.py <<'EOF'
#!/usr/bin/env python3
"""
CLI Auditor untuk memeriksa kepatuhan file konfigurasi YAML service produksi.
Contoh pemakaian:
  python audit_yaml.py --file deployments.yaml --output report.json
"""
import argparse
import json
import logging
import sys
from pathlib import Path
import yaml

# Konfigurasi logging standar SRE (menulis ke stderr dengan timestamp)
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)
log = logging.getLogger("sre-auditor")


def audit_service(svc: dict) -> list[str]:
    """Memeriksa satu service dan mengembalikan daftar pelanggaran (violations)."""
    violations = []
    name = svc.get("name", "<tanpa-nama>")
    image = svc.get("image", "")
    replicas = svc.get("replicas", 0)
    health_path = svc.get("healthcheck_path")
    mem_limit = svc.get("memory_limit_mi")

    if not image or image.endswith(":latest") or ":" not in image:
        violations.append(f"{name}: dilarang menggunakan image tag ':latest' atau tanpa tag ({image})")

    if replicas < 2:
        violations.append(f"{name}: jumlah replicas ({replicas}) kurang dari standar HA (minimal 2)")

    if not health_path:
        violations.append(f"{name}: wajib mendefinisikan 'healthcheck_path'")

    if not isinstance(mem_limit, int) or mem_limit <= 0:
        violations.append(f"{name}: wajib mendefinisikan 'memory_limit_mi' (integer > 0)")

    return violations


def main() -> int:
    parser = argparse.ArgumentParser(description="Audit kepatuhan konfigurasi YAML produksi SRE.")
    parser.add_argument("-f", "--file", required=True, help="Path ke file YAML yang akan diaudit")
    parser.add_argument("-o", "--output", help="Path opsional untuk menyimpan hasil laporan JSON")
    args = parser.parse_args()

    yaml_path = Path(args.file)
    if not yaml_path.is_file():
        log.error("File tidak ditemukan: %s", yaml_path)
        return 2

    try:
        config = yaml.safe_load(yaml_path.read_text(encoding="utf-8"))
    except yaml.YAMLError as exc:
        log.error("Gagal mem-parsing YAML %s: %s", yaml_path, exc)
        return 2

    services = config.get("services", [])
    all_violations = []

    log.info("Memulai audit untuk environment='%s' (%d services)...", config.get("environment"), len(services))
    for svc in services:
        v = audit_service(svc)
        all_violations.extend(v)

    report = {
        "source_file": str(yaml_path),
        "environment": config.get("environment", "unknown"),
        "total_services": len(services),
        "passed": len(all_violations) == 0,
        "violations": all_violations,
    }

    if args.output:
        out_path = Path(args.output)
        out_path.write_text(json.dumps(report, indent=2) + "\n", encoding="utf-8")
        log.info("Laporan JSON disimpan ke %s", out_path)

    if all_violations:
        for item in all_violations:
            log.warning("PELANGGARAN: %s", item)
        log.error("Audit GAGAL dengan %d pelanggaran.", len(all_violations))
        return 1

    log.info("Audit LULUS. Semua service memenuhi standar produksi.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
EOF

chmod +x audit_yaml.py
```

Mari kita uji jalankan skrip tersebut dan periksa kode keluar (*exit code*) serta file `report.json`:

```bash
# 1. Lihat panduan --help otomatis dari argparse
./audit_yaml.py --help

# 2. Jalankan audit terhadap deployments.yaml
./audit_yaml.py --file deployments.yaml --output report.json || echo "Exit code: $?"

# 3. Lihat isi laporan JSON yang dihasilkan
cat report.json
```
*Output yang diharapkan:*
```text
2026-10-17T10:00:00 [INFO] Memulai audit untuk environment='production' (3 services)...
2026-10-17T10:00:00 [WARNING] PELANGGARAN: payment-worker: dilarang menggunakan image tag ':latest' atau tanpa tag (registry.gitlab.com/sre/payment:latest)
2026-10-17T10:00:00 [WARNING] PELANGGARAN: payment-worker: jumlah replicas (1) kurang dari standar HA (minimal 2)
2026-10-17T10:00:00 [WARNING] PELANGGARAN: payment-worker: wajib mendefinisikan 'healthcheck_path'
2026-10-17T10:00:00 [WARNING] PELANGGARAN: reporting-api: wajib mendefinisikan 'memory_limit_mi' (integer > 0)
2026-10-17T10:00:00 [ERROR] Audit GAGAL dengan 4 pelanggaran.
Exit code: 1
```

---

## 4. Pola Kegagalan (Failure Mode) & Diagnosa

1. **Menggunakan `yaml.load()` alih-alih `yaml.safe_load()`:**
   - **Risiko:** `yaml.load()` tanpa Loader aman mengizinkan eksekusi objek Python arbitrer di dalam file YAML (celah keamanan fatal jika memproses YAML dari luar).
   - **Solusi:** Selalu gunakan `yaml.safe_load()`.
2. **Lupa Mengembalikan Exit Code Non-Zero Saat Gagal:**
   - **Gejala:** Skrip mencetak tulisan `[ERROR] Audit gagal`, tetapi selesai dengan exit code `0`. Akibatnya, GitLab CI menganggap pipeline **hijau (sukses)** dan tetap melanjutkan deployment yang bermasalah.
   - **Solusi:** Pastikan `sys.exit(1)` dipanggil saat ditemukan kondisi gagal.
3. **Menulis Log ke `stdout` Saat Output Skrip Berupa JSON:**
   - **Gejala:** Jika skrip dipanggil di pipeline `python script.py | jq .`, pencampuran pesan `print("Memulai proses...")` dengan output JSON membuat `jq` gagal mem-parsing.
   - **Solusi:** Gunakan modul `logging` (yang secara default menulis ke `stderr`) atau pisahkan output data ke file `--output`.

---

## 5. Ringkasan & Langkah Selanjutnya

- Gunakan `python3 -m venv .venv` dan `requirements.txt` di setiap proyek Python.
- Manfaatkan `.get()` saat mengakses `dict` hasil parsing JSON/YAML agar tahan terhadap field yang hilang.
- Gunakan `argparse`, `logging`, dan `sys.exit()` agar skrip Python siap dijalankan di dalam CI/CD maupun cron job server.
- Selanjutnya di [03-python-http-api-automation.md](03-python-http-api-automation.md), kita akan menghubungkan skrip Python dengan jaringan untuk memeriksa kesehatan layanan HTTP dan berinteraksi dengan REST API secara tangguh.
