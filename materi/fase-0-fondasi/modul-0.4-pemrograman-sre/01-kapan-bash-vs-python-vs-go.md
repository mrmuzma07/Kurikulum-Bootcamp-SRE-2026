# 01 — Kapan Pakai Bash vs Python vs Go dalam Operasional SRE

> **Ringkasan:** Memahami batas pemakaian **Bash**, **Python**, dan **Go** di lingkungan production on-premises agar skrip operasional tetap dapat dipelihara (*maintainable*), aman dari bug tersembunyi, dan sesuai dengan ekosistem Cloud-Native.

---

## 1. Konsep & Arsitektur: Tiga Alat di Sabuk Perkakas SRE

Di Modul 0.1 ([07-bash-scripting.md](../modul-0.1-linux-shell/07-bash-scripting.md)), kamu sudah belajar menulis skrip Bash dengan `set -euo pipefail`. Pertanyaan yang sering muncul dari pemula adalah: *"Kalau Bash sudah bisa menjalankan semua perintah Linux, kenapa SRE masih perlu belajar Python dan Go?"*

> 🧰 **Analogi Perkakas Bengkel:**
> - **Bash** adalah **lakban & pisau lipat**: selalu ada di saku setiap server Linux, sangat cepat untuk menyambung 3–5 perintah CLI (`grep`, `awk`, `tar`, `systemctl`), tetapi berantakan jika dipakai membangun mesin yang rumit.
> - **Python** adalah **obeng elektrik & kunci pas lengkap**: sangat nyaman untuk logika bercabang, mengolah struktur data bersarang (JSON/YAML), berbicara dengan REST API, dan menulis skrip otomasi ratusan baris dengan cepat.
> - **Go (Golang)** adalah **mesin bubut presisi pabrik**: menghasilkan **satu file biner statis** tanpa ketergantungan (*dependency*) runtime di server target, sangat cepat, hemat RAM, dan menjadi bahasa pembangun Kubernetes, Docker, OpenTofu, Helm, ArgoCD, hingga Prometheus.

```text
Kompleksitas Logika & Struktur Data
  ▲
  │                                     ┌──────────────────────────────┐
  │                                     │          Go (Golang)         │
  │                                     │ - Microservice / Daemon      │
  │                                     │ - Biner statis multi-arch    │
  │                  ┌──────────────────┤ - K8s Operator / Exporter    │
  │                  │    Python 3      └──────────────────────────────┘
  │                  │ - Otomasi API / Webhook
  │                  │ - Olah JSON / YAML kompleks
  │  ┌───────────────┤ - Skrip CLI > 80 baris & Ansible
  │  │  Bash Shell   └──────────────────┐
  │  │ - Glue script (< 50–80 baris)    │
  │  │ - Wrapper perintah OS / cron     │
  └──┴───────────────┴──────────────────┴──────────────────────────────►
                                                Kebutuhan Performa & Distribusi Biner
```

---

## 2. Parameter Keputusan: Matriks Pemilihan Bahasa

Gunakan tabel keputusan ini sebelum kamu mulai menulis baris kode pertama:

| Kriteria | Bash (`#!/usr/bin/env bash`) | Python (`#!/usr/bin/env python3`) | Go (`go build`) |
|---|---|---|---|
| **Panjang kode ideal** | < 50–80 baris | 50 – 2.000+ baris | 100 – puluhan ribu baris |
| **Struktur data (JSON/YAML)** | Buruk (harus bergantung pada `jq`/`yq` eksternal) | **Sangat kuat** (`json`, `PyYAML`, dictionary/list native) | **Sangat kuat** & *type-safe* (`struct` + tag `json:"..."`) |
| **Panggilan HTTP / REST API** | Terbatas (`curl` + parsing teks manual) | **Sangat mudah** (`requests` / `urllib`, mudah *retry* & *timeout*) | **Sangat cepat & konkuren** (`net/http`, `goroutine`) |
| **Error handling** | Kasar (`exit code` `$?` dan `set -e`) | Terstruktur (`try / except / finally`) | Eksplisit di setiap fungsi (`if err != nil`) |
| **Unit testing** | Sulit (butuh framework tambahan seperti `bats`) | Mudah (`pytest` / `unittest`) | **Bawaan langsung** (`go test ./...`) |
| **Distribusi ke server bare-metal** | Langsung jalan (Bash ada di semua Linux) | Butuh `python3` + `venv` / paket `pip` di server | **1 file biner mandiri** (tidak butuh runtime apa pun di server/container) |
| **Peran utama di Bootcamp ini** | Wrapper instalasi, backup sederhana, job CI singkat | Skrip audit infrastruktur, otomasi API, fondasi Ansible | Aplikasi sampel container, Kubernetes probes, Prometheus `/metrics` |

> **Aturan Jempol SRE (Rule of Thumb):**
> Jika skrip Bash-mu mulai membutuhkan **array bersarang**, **parsing JSON lebih dari 2 tingkat**, **logika retry HTTP**, atau **lebih dari 80 baris**, **berhentilah menulis Bash dan pindahkan ke Python**.

---

## 3. Hands-On Step-by-Step: Membandingkan Tugas yang Sama di 3 Bahasa

Mari kita buktikan perbedaannya secara langsung di terminal. Tugas kita adalah: **membaca data daftar node server (JSON), memfilter node yang penggunaan disk-nya ≥ 85%, lalu mencetak peringatan dengan kode keluar (*exit code*) `1` jika ada node kritis.**

### 3.1 Persiapan Workspace & Data Sampel
Jalankan di VM `lab01` (atau terminal lokal):

```bash
mkdir -p ~/latihan-m0.4/perbandingan && cd ~/latihan-m0.4/perbandingan

cat > nodes.json <<'EOF'
{
  "cluster": "onprem-k3s-prod",
  "nodes": [
    {"name": "k3s-srv-01", "ip": "192.168.10.11", "disk_used_pct": 62, "ready": true},
    {"name": "k3s-agt-01", "ip": "192.168.10.21", "disk_used_pct": 88, "ready": true},
    {"name": "k3s-agt-02", "ip": "192.168.10.22", "disk_used_pct": 91, "ready": false}
  ]
}
EOF
```

### 3.2 Pendekatan 1: Bash + `jq` (Mulai Kaku Saat Logika Bertambah)
```bash
cat > cek_disk.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

FILE="${1:-nodes.json}"
THRESHOLD=85

# Mencari node dengan disk >= THRESHOLD menggunakan jq
CRITICAL=$(jq -r --argjson t "$THRESHOLD" '.nodes[] | select(.disk_used_pct >= $t) | "\(.name) (\(.ip)): \(.disk_used_pct)%"' "$FILE")

if [[ -n "$CRITICAL" ]]; then
  echo "[WARN] Node melewati batas disk ${THRESHOLD}%:"
  echo "$CRITICAL"
  exit 1
fi

echo "[OK] Semua node aman."
EOF

chmod +x cek_disk.sh
./cek_disk.sh || echo "Exit code: $?"
```
*Output yang diharapkan:*
```text
[WARN] Node melewati batas disk 85%:
k3s-agt-01 (192.168.10.21): 88%
k3s-agt-02 (192.168.10.22): 91%
Exit code: 1
```
*Evaluasi:* Untuk 1 filter sederhana masih cukup singkat, tetapi jika kita harus menggabungkan data ini dengan panggilan API ke Prometheus dan mengirim laporan terformat, sintaks `jq` di dalam string Bash akan cepat menjadi sulit dibaca.

### 3.3 Pendekatan 2: Python 3 (Jelas, Mudah Dibaca, Mudah Diperluas)
```bash
cat > cek_disk.py <<'EOF'
#!/usr/bin/env python3
import json
import sys
from pathlib import Path

THRESHOLD = 85
file_path = Path(sys.argv[1] if len(sys.argv) > 1 else "nodes.json")

data = json.loads(file_path.read_text(encoding="utf-8"))
cluster = data.get("cluster", "unknown")

critical_nodes = [
    n for n in data.get("nodes", [])
    if n.get("disk_used_pct", 0) >= THRESHOLD or not n.get("ready", False)
]

if critical_nodes:
    print(f"[WARN] Cluster '{cluster}' memiliki {len(critical_nodes)} node bermasalah:")
    for node in critical_nodes:
        status = "READY" if node["ready"] else "NOT_READY"
        print(f"  - {node['name']} ({node['ip']}): disk={node['disk_used_pct']}%, status={status}")
    sys.exit(1)

print(f"[OK] Semua node di cluster '{cluster}' sehat.")
EOF

chmod +x cek_disk.py
./cek_disk.py || echo "Exit code: $?"
```
*Output yang diharapkan:*
```text
[WARN] Cluster 'onprem-k3s-prod' memiliki 2 node bermasalah:
  - k3s-agt-01 (192.168.10.21): disk=88%, status=READY
  - k3s-agt-02 (192.168.10.22): disk=91%, status=NOT_READY
Exit code: 1
```
*Evaluasi:* Perhatikan betapa mudahnya kita menambahkan kondisi `or not n.get("ready", False)` dan memformat output tanpa bergantung pada tool eksternal `jq`.

### 3.4 Pendekatan 3: Go (Type-Safe & Dikompilasi Jadi Biner)
```bash
cat > main.go <<'EOF'
package main

import (
	"encoding/json"
	"fmt"
	"os"
)

type Node struct {
	Name        string `json:"name"`
	IP          string `json:"ip"`
	DiskUsedPct int    `json:"disk_used_pct"`
	Ready       bool   `json:"ready"`
}

type ClusterInventory struct {
	Cluster string `json:"cluster"`
	Nodes   []Node `json:"nodes"`
}

func main() {
	raw, err := os.ReadFile("nodes.json")
	if err != nil {
		fmt.Fprintf(os.Stderr, "[ERROR] gagal baca file: %v\n", err)
		os.Exit(2)
	}

	var inv ClusterInventory
	if err := json.Unmarshal(raw, &inv); err != nil {
		fmt.Fprintf(os.Stderr, "[ERROR] JSON tidak valid: %v\n", err)
		os.Exit(2)
	}

	hasIssue := false
	for _, n := range inv.Nodes {
		if n.DiskUsedPct >= 85 || !n.Ready {
			hasIssue = true
			fmt.Printf("[WARN] %s (%s): disk=%d%%, ready=%v\n", n.Name, n.IP, n.DiskUsedPct, n.Ready)
		}
	}
	if hasIssue {
		os.Exit(1)
	}
	fmt.Println("[OK] Semua node sehat.")
}
EOF

go run main.go || echo "Exit code: $?"
```
*Evaluasi:* Kodenya sedikit lebih panjang karena kita mendeklarasikan `struct` (`Node` dan `ClusterInventory`), tetapi kompilator Go menjamin keamanan tipe data (*type-safety*) dan dapat dikompilasi menjadi satu file biner `cek-node` yang bisa dijalankan di server mana saja tanpa perlu menginstal Python atau `jq`.

---

## 4. Pola Kegagalan (Failure Mode) & Diagnosa

1. **Jebakan "Bash Monster" (Skrip Bash 500 Baris):**
   - **Gejala:** Skrip otomasi deployment/backup di Bash gagal diam-diam ketika ada spasi pada nama variabel atau nilai JSON berisi karakter tanda kutip `"`.
   - **Solusi SRE:** Refactor ke Python menggunakan modul `subprocess`, `json`, dan `pathlib`.
2. **Ketergantungan Paket Python yang Merusak OS (*Broken System Python*):**
   - **Gejala:** Menjalankan `sudo pip install ...` langsung di Ubuntu 22.04/24.04 menyebabkan error `externally-managed-environment` atau merusak tool sistem OS (seperti `apt` / `cloud-init`).
   - **Solusi SRE:** **Selalu** gunakan Virtual Environment (`python3 -m venv .venv`) untuk proyek Python.
3. **Biner Go Salah Arsitektur (`exec format error`):**
   - **Gejala:** Mengompilasi biner Go di MacBook Apple Silicon (`arm64`) lalu menyalinnya ke VM/Server produksi `amd64` tanpa mengatur `GOOS` dan `GOARCH`.
   - **Solusi SRE:** Selalu nyatakan target OS dan arsitektur saat build: `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build`.

---

## 5. Ringkasan & Langkah Selanjutnya

- Gunakan **Bash** untuk merangkai perintah CLI singkat (< 50–80 baris).
- Gunakan **Python** untuk skrip operasional yang mengolah JSON/YAML, CLI parameter, dan otomasi REST API.
- Gunakan **Go** untuk membangun service HTTP, custom exporter, dan aplikasi cloud-native yang dikemas ke dalam container kecil.
- Selanjutnya di [02-python-dasar-json-yaml-cli.md](02-python-dasar-json-yaml-cli.md), kita akan mempraktikkan Python dari nol khusus untuk kebutuhan SRE sehari-hari.
