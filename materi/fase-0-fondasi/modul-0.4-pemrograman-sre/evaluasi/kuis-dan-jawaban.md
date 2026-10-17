# Kuis & Kunci Jawaban — Modul 0.4 Pemrograman untuk SRE (Python & Go)

> **Cara kerja:** jawab sendiri tanpa membuka catatan. Cek jawabanmu di bagian bawah. Target kelulusan ≥ 80% (minimal 16 dari 20 soal benar).

---

## Bagian A — Pilihan Ganda (10 soal)

**1.** Kapan seorang SRE sebaiknya berhenti menulis skrip dalam **Bash** dan memindahkannya ke **Python**?
- A. Saat skrip hanya menjalankan 3 perintah `mkdir`, `cp`, dan `tar`
- B. Saat skrip membutuhkan manipulasi struktur data JSON/YAML bersarang, logika HTTP retry, atau panjangnya sudah melewati ~80 baris
- C. Saat ingin mengubah permission file dengan `chmod`
- D. Bash tidak boleh dipakai sama sekali oleh SRE

**2.** Mengapa di Ubuntu 22.04/24.04 kita wajib menggunakan `python3 -m venv .venv` dibanding menjalankan `sudo pip install` langsung?
- A. Karena `sudo pip install` hanya berjalan di Windows
- B. Untuk mengisolasi dependensi paket proyek dan mencegah rusaknya paket Python bawaan sistem operasi Linux
- C. Karena `.venv` otomatis mengompilasi Python menjadi biner Go
- D. Agar file `.venv` bisa di-commit ke GitLab

**3.** Saat membaca dictionary Python hasil parsing konfigurasi YAML, mengapa `svc.get("replicas", 1)` lebih disarankan dibanding `svc["replicas"]`?
- A. Karena `svc["replicas"]` akan memicu `KeyError` dan menghentikan skrip jika key `"replicas"` tidak ditulis di file YAML
- B. Karena `.get()` mengubah tipe integer menjadi string
- C. Karena `svc["replicas"]` hanya bekerja untuk file JSON
- D. Keduanya persis sama, tidak ada perbedaan saat key hilang

**4.** Fungsi dari library `PyYAML` yang **aman** untuk mem-parsing file YAML agar terhindar dari celah keamanan eksekusi kode arbitrer adalah...
- A. `yaml.load_all_unsafe()`
- B. `yaml.safe_load()`
- C. `yaml.eval()`
- D. `yaml.exec()`

**5.** Apa risiko operasional jika skrip Python memanggil `requests.get("https://api.internal/healthz")` **tanpa** menyertakan parameter `timeout`?
- A. Python otomatis membatalkan request dalam 1 detik
- B. Request ditolak oleh sintaks Python
- C. Skrip dapat menggantung (*hang*) tanpa batas waktu jika server atau jaringan mengalami *packet drop* setelah koneksi terbuka
- D. Server tujuan akan otomatis restart

**6.** Agar job GitLab CI atau skrip pemanggil mengetahui bahwa skrip validasi Python menemukan kegagalan konfigurasi, apa yang wajib dilakukan skrip sebelum berhenti?
- A. Mencetak kata `"GAGAL"` lalu keluar dengan `sys.exit(0)`
- B. Keluar dengan kode keluar non-zero, misalnya `sys.exit(1)`
- C. Menghapus file konfigurasi YAML
- D. Menunggu input `Enter` dari pengguna

**7.** Di dalam aplikasi Go, apa tujuan utama mengatur variabel `CGO_ENABLED=0` saat menjalankan `go build` untuk container produksi?
- A. Memperbesar ukuran file biner agar terlihat lengkap
- B. Menghasilkan biner statis mandiri yang tidak bergantung pada library C (`glibc`) sistem operasi host, sehingga bisa jalan di image `distroless` atau `scratch`
- C. Mengaktifkan fitur debugging GUI
- D. Memaksa aplikasi hanya berjalan di macOS

**8.** Perintah untuk melakukan *cross-compile* aplikasi Go dari laptop MacBook Apple Silicon (`arm64`) agar menghasilkan biner untuk server produksi Linux `x86_64` (`amd64`) adalah...
- A. `GOOS=darwin GOARCH=arm64 go build`
- B. `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build`
- C. `go build --target=windows`
- D. `python3 setup.py build_amd64`

**9.** Dalam aplikasi HTTP yang akan dijalankan di Kubernetes, apa perbedaan fungsi `/healthz` (*Liveness*) dan `/readyz` (*Readiness*)?
- A. `/healthz` untuk cek koneksi internet luar, `/readyz` untuk cek kapasitas harddisk
- B. `/healthz` memberi tahu apakah proses aplikasi masih hidup (jika gagal, Pod di-restart); `/readyz` memberi tahu apakah aplikasi siap menerima trafik (jika gagal, Pod dilepas sementara dari Service/LoadBalancer tanpa di-restart)
- C. Keduanya hanya dipakai untuk menampilkan halaman login pengguna
- D. `/readyz` dipakai oleh Prometheus untuk mengambil metrik

**10.** Sinyal sistem operasi yang dikirim oleh Kubernetes (atau `systemctl stop`) untuk meminta aplikasi berhenti secara anggun (*graceful shutdown*) sebelum dipaksa mati dengan `SIGKILL` adalah...
- A. `SIGHUP`
- B. `SIGTERM`
- C. `SIGSEGV`
- D. `SIGALRM`

---

## Bagian B — Perintah & Potongan Kode (4 soal)

**11.** Tulis urutan perintah terminal untuk membuat virtual environment Python di folder `.venv`, mengaktifkannya, menginstal `pyyaml` dan `requests`, lalu menyimpan daftar versinya ke `requirements.txt`.

**12.** Tulis potongan kode Python singkat (3–5 baris) yang membaca *environment variable* bernama `GITLAB_TOKEN`, dan jika variabel tersebut kosong/belum di-set, mencetak pesan error ke `stderr` lalu keluar dengan *exit code* `2`.

**13.** Tulis perintah `go build` lengkap yang mengompilasi kode Go di direktori saat ini menjadi biner statis Linux ARM64 bernama `dist/app-arm64` dengan ukuran terkompresi (simbol debug dibuang).

**14.** Tulis perintah untuk menjalankan seluruh unit test Go di dalam sebuah proyek beserta subdirektorinya dengan output detail (*verbose*).

---

## Bagian C — Skenario Operasional (2 soal)

**15.** Anda sedang membangun pipeline CI (Fase 6) yang menjalankan skrip Python `audit.py --file deployments.yaml`. Di dalam `deployments.yaml`, ada seorang developer yang menulis `image: nginx:latest` dan `replicas: 1` untuk environment `production`. Jelaskan:
1. Mengapa penggunaan tag `:latest` berbahaya di lingkungan produksi SRE?
2. Apa output dan *exit code* yang harus diberikan oleh `audit.py` agar pipeline CI otomatis menggagalkan Merge Request tersebut?

**16.** Saat tim melakukan *rolling update* aplikasi Go di cluster Kubernetes (Fase 2), beberapa pengguna melaporkan error `502 Bad Gateway` selama beberapa detik setiap kali Pod lama dimatikan. Setelah Anda periksa `main.go`, aplikasi hanya memanggil `http.ListenAndServe(":8080", mux)` tanpa menangkap sinyal `SIGTERM`. Jelaskan apa yang sebenarnya terjadi pada koneksi HTTP yang sedang berjalan di Pod lama, dan bagaimana mekanisme *graceful shutdown* (`signal.NotifyContext` + `srv.Shutdown` + mengubah status `/readyz` menjadi `503`) menyelesaikan masalah ini.

---

## Bagian D — Troubleshooting (2 soal)

**17.** Seorang rekan SRE mengompilasi aplikasi Go di laptop Mac M5 miliknya hanya dengan perintah `go build -o my-app .`, lalu menyalin file `my-app` ke VM Ubuntu Linux di OrbStack dan menjalankannya. Muncul error:
`cannot execute binary file: Exec format error`
Jelaskan penyebab pasti error tersebut, perintah untuk memverifikasi jenis file biner itu, dan cara memperbaikinya!

**18.** Anda meng-clone repo `sre-bootcamp` milik rekan tim ke VM baru, lalu langsung menjalankan `./m0.4/audit-tool/audit.py --file endpoints.yaml` dan mendapat error:
`ModuleNotFoundError: No module named 'yaml'`
Padahal di dalam folder tersebut sudah ada file `requirements.txt`. Tulis langkah-langkah tepat untuk memperbaikinya tanpa mengotori sistem Python OS!

---

## Bagian E — Esai Singkat (2 soal)

**19.** Bandingkan peran **Bash**, **Python**, dan **Go** di dalam ekosistem kerja SRE. Berikan masing-masing **1 contoh kasus nyata** di mana bahasa tersebut adalah pilihan paling tepat dibanding dua bahasa lainnya!

**20.** Mengapa menanamkan endpoint `/metrics` (format teks Prometheus) langsung di dalam kode aplikasi (seperti yang kita lakukan pada `LAB-02`) jauh lebih kuat untuk observabilitas dibanding hanya mengandalkan pembacaan log teks biasa?

---

## Kunci Jawaban

### A — Pilihan Ganda
1. **B** — Saat membutuhkan struktur data JSON/YAML bersarang, HTTP retry, atau kode > 80 baris, Python jauh lebih aman dan mudah dipelihara dibanding Bash.
2. **B** — `venv` mengisolasi paket per proyek dan mencegah konflik dengan paket Python bawaan OS (`externally-managed-environment`).
3. **A** — `.get("key", default)` mencegah skrip crash akibat `KeyError` ketika field opsional tidak ada di YAML/JSON.
4. **B** — `yaml.safe_load()` hanya mem-parsing tipe data standar YAML yang aman tanpa mengeksekusi objek Python berbahaya.
5. **C** — Tanpa `timeout`, `requests.get()` dapat menunggu respons selamanya (*hang*) jika terjadi *blackhole/packet drop* di jaringan.
6. **B** — Sistem CI/CD dan shell Linux memeriksa *exit code*: `0` berarti sukses, sedangkan `1–255` (`sys.exit(1)`) berarti gagal.
7. **B** — `CGO_ENABLED=0` mematikan ketergantungan dinamis ke `libc` OS sehingga menghasilkan biner statis murni.
8. **B** — `CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build` menghasilkan biner statis untuk Linux x86_64 (`amd64`).
9. **B** — `/healthz` (Liveness) menentukan apakah Pod perlu di-restart; `/readyz` (Readiness) menentukan apakah Pod boleh menerima trafik dari Service.
10. **B** — `SIGTERM` adalah sinyal terminasi standar yang bisa ditangkap aplikasi untuk membereskan transaksi sebelum mati.

### B — Perintah & Potongan Kode
**11.**
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install pyyaml requests
pip freeze > requirements.txt
```

**12.**
```python
import os, sys
token = os.environ.get("GITLAB_TOKEN")
if not token:
    print("[ERROR] GITLAB_TOKEN belum di-set", file=sys.stderr)
    sys.exit(2)
```

**13.**
```bash
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -ldflags="-s -w" -o dist/app-arm64 .
```

**14.**
```bash
go test -v ./...
```

### C — Skenario Operasional
**15.**
1. Tag `:latest` bersifat *mutable* (isinya bisa berubah sewaktu-waktu tanpa jejak versi yang jelas), sehingga melanggar prinsip *reproducibility* dan menyulitkan *rollback* saat terjadi insiden. Sementara `replicas: 1` menciptakan *Single Point of Failure (SPOF)*.
2. `audit.py` harus mencatat kedua pelanggaran tersebut di laporan JSON/log `stderr` dan keluar dengan **`sys.exit(1)`** (exit code `1`), sehingga job GitLab CI otomatis berstatus **Failed (merah)** dan memblokir Merge Request.

**16.**
- **Penyebab:** Tanpa menangkap `SIGTERM`, proses Go langsung mati seketika saat menerima sinyal berhenti, sehingga request HTTP yang sedang diproses di tengah jalan langsung terputus (*connection reset* → `502 Bad Gateway` di sisi reverse proxy/ingress).
- **Solusi Graceful Shutdown:** Ketika `SIGTERM` diterima via `signal.NotifyContext`, aplikasi langsung mengubah state `/readyz` menjadi `503` (agar Service berhenti mengirim trafik baru), lalu memanggil `srv.Shutdown(ctx)` yang menunggu request yang sedang berjalan hingga selesai sebelum menutup proses dengan bersih.

### D — Troubleshooting
**17.**
- **Penyebab:** `go build` tanpa variabel `GOOS` di macOS menghasilkan biner berformat **Mach-O (macOS / darwin)**, bukan format **ELF (Linux)**, sehingga kernel Linux menolak mengeksekusinya (`Exec format error`).
- **Verifikasi:** Jalankan `file my-app` (akan terlihat `Mach-O 64-bit arm64 executable`).
- **Perbaikan:** Kompilasi ulang dengan menyertakan target OS Linux: `CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o my-app .`.

**18.**
Buat virtual environment lokal di folder tersebut, aktifkan, lalu instal dependensi dari `requirements.txt`:
```bash
cd m0.4/audit-tool
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
./audit.py --file endpoints.yaml
```

### E — Esai Singkat
**19.**
- **Bash:** Paling tepat untuk *glue script* singkat (< 50 baris) yang memanggil utilitas Linux secara berurutan, misalnya skrip cron backup `tar -czf` direktori `/etc` lalu menyalinnya dengan `rsync`.
- **Python:** Paling tepat untuk skrip otomasi operasional menengah yang mem-parsing konfigurasi YAML/JSON dan berinteraksi dengan REST API, misalnya skrip auditor kepatuhan manifest atau skrip pengambil laporan alert dari Prometheus/GitLab API.
- **Go:** Paling tepat untuk membangun layanan jaringan/microservice berperforma tinggi, custom Prometheus Exporter, atau Kubernetes Operator yang perlu didistribusikan sebagai satu biner statis kecil di dalam container `distroless`.

**20.**
Metrik di `/metrics` berbentuk angka terstruktur (*time-series* seperti `counter`, `gauge`, `histogram`) yang sangat ringan untuk di-scrape, disimpan, dan dihitung laju perubahannya (`rate()`) oleh Prometheus/Mimir guna mendeteksi gejala awal penurunan SLO (seperti lonjakan error rate atau latensi) secara real-time, tanpa harus mem-parsing jutaan baris teks log yang mahal CPU dan disk.
