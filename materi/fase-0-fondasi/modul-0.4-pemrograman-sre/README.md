# Modul 0.4 — Pemrograman untuk SRE (Python & Go Dasar)

> **Tujuan akhir:** mampu menulis skrip otomasi operasional dengan **Python** (mengolah JSON/YAML, CLI argumen, dan memanggil REST API) serta memahami dan membuild service HTTP ringan dengan **Go** (endpoint `/health`, `/metrics`, environment variable, dan kompilasi biner multi-arch) sebagai fondasi containerisasi di Fase 1, Ansible di Fase 4, Helm di Fase 5, dan Observability di Fase 7.

## Capaian Modul (Wajib)

- [ ] Bisa menentukan kapan tetap memakai **Bash**, kapan beralih ke **Python**, dan kapan memakai **Go** di operasional SRE
- [ ] Bisa membuat dan mengelola environment terisolasi Python (`python3 -m venv`) agar tidak merusak Python sistem OS
- [ ] Bisa menulis skrip Python untuk membaca, memvalidasi, dan memodifikasi file **JSON** dan **YAML** (bekal manifest Kubernetes, Ansible, dan Helm)
- [ ] Bisa membuat CLI tool sederhana dengan `argparse`, *exit code* yang jujur, dan *structured logging*
- [ ] Bisa memanggil HTTP REST API dari Python (dengan *timeout*, *retry*, dan *error handling*) untuk *health check* dan audit otomatis
- [ ] Bisa membaca, memodifikasi, dan mengompilasi aplikasi HTTP sederhana menggunakan **Go** (`/healthz`, `/readyz`, `/metrics`, dan *graceful shutdown* `SIGTERM`)
- [ ] Bisa melakukan *cross-compile* biner Go statis (`CGO_ENABLED=0 GOOS=linux GOARCH=arm64/amd64`) yang siap dimasukkan ke container `distroless` di Fase 1

## Rencana 3 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-kapan-bash-vs-python-vs-go](01-kapan-bash-vs-python-vs-go.md) + [02-python-dasar-json-yaml-cli](02-python-dasar-json-yaml-cli.md) | [Latihan Hari 1](evaluasi/latihan.md) |
| 2 | [03-python-http-api-automation](03-python-http-api-automation.md) | [LAB-01: Skrip Audit Endpoint & Konfigurasi](lab/LAB-01-script-audit-api.md) + [Latihan Hari 2](evaluasi/latihan.md) |
| 3 | [04-go-microservice-dasar](04-go-microservice-dasar.md) | [LAB-02: Aplikasi Go HTTP Siap Produksi](lab/LAB-02-http-service-health.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 0.1, 0.2, dan 0.3 selesai (nyaman dengan terminal Linux, konsep HTTP/TCP, dan Git workflow)
- VM OrbStack (`lab01`) atau terminal macOS siap digunakan
- Python 3 dan Go terpasang di VM `lab01` (atau macOS):

```bash
# Di dalam VM Ubuntu OrbStack (orb shell lab01):
sudo apt update && sudo apt install -y python3 python3-venv python3-pip golang-go jq

# Verifikasi instalasi:
python3 --version
go version
```

- Sudah membaca [Fase 0 README](../README.md)

## Deliverables Modul

1. **Direktori `m0.4/` di repo `sre-bootcamp`** yang di-merge ke `main` lewat Merge Request (MR), berisi:
   ```text
   sre-bootcamp/
   └── m0.4/
       ├── audit-tool/          (LAB-01: skrip Python CLI pemeriksa YAML & HTTP endpoint)
       │   ├── audit.py
       │   ├── requirements.txt
       │   └── endpoints.yaml
       ├── app-go/              (LAB-02: service HTTP Go dengan /healthz, /readyz, /metrics)
       │   ├── main.go
       │   ├── main_test.go
       │   └── go.mod
       └── lab/
           └── log-latihan-m0.4.md
   ```
2. **≥ 1 Merge Request** dengan conventional commit (`feat(m0.4): ...`) ke repo GitLab `sre-bootcamp`.
3. **Nilai kuis ≥ 80%** (minimal 16 dari 20 soal benar).

## Cara Memulai

Seorang SRE **bukan** harus menjadi *full-stack software engineer* yang menghafal framework frontend/backend rumit. Namun, SRE modern wajib bisa membaca dan menulis kode untuk:
1. **Mengotomasi *toil*** (pekerjaan manual berulang yang bisa diprogram).
2. **Memvalidasi konfigurasi & API** (mengecek puluhan endpoint, membedah JSON/YAML dari Kubernetes atau Prometheus).
3. **Memahami anatomi aplikasi yang dioperasikan** (bagaimana aplikasi membaca *environment variable*, membuka port, merespons *liveness/readiness probe*, menghasilkan metrik `/metrics`, dan berhenti dengan anggun saat menerima sinyal `SIGTERM` dari Kubernetes).

## Kaitan dengan Modul Berikutnya

Apa yang kamu buat di modul ini langsung dipakai di fase-fase selanjutnya:
- **Fase 1 (Container & OrbStack):** Aplikasi Go dari `LAB-02` akan langsung di-containerize dengan *multi-stage Dockerfile* di Modul 1.1.
- **Fase 2 (Kubernetes):** Endpoint `/healthz` dan `/readyz` yang kamu tulis di Go akan dipakai oleh `livenessProbe` dan `readinessProbe` Pod Kubernetes.
- **Fase 4 (Ansible):** Struktur data Python (*dict*, *list*) dan YAML akan memudahkanmu menguasai Ansible Playbook dan Jinja2 template.
- **Fase 5 (Helm):** Sintaks `{{ .Values.xxx }}` di Helm menggunakan engine `text/template` bawaan bahasa Go.
- **Fase 7 (Observability):** Endpoint `/metrics` yang kamu buat akan di-scrape langsung oleh Prometheus & Grafana Alloy.
