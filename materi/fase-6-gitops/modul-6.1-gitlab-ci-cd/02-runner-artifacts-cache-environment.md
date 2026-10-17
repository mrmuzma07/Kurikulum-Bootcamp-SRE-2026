# 02 — GitLab Runner, Artifacts, Cache, dan Environment: Mengelola Agen Eksekusi dan Isolasi Lingkungan

> **Ringkasan:** Pipeline CI/CD tidak berjalan di atas ruang hampa udara; ia membutuhkan mesin pelaksana nyata yang disebut **GitLab Runner**. Dalam konteks *on-premises* dan *private data center*, ketergantungan pada runner publik (*shared SaaS runners*) sering kali tidak memungkinkan karena keterbatasan akses jaringan ke server internal dan kepatuhan data (*data sovereignty*). Mengoperasikan *Self-Hosted Runner* yang aman menuntut pemahaman arsitektur *agent polling*, pemilihan *executor* yang terisolasi, tata kelola *environment*, serta pemanfaatan *protected variables*. Dokumen ini membedah operasional runner kelas produksi, optimalisasi strategi *cache*, penanganan variabel tersamar (*masked variables*), dan pemeliharaan disk agen secara otomatis.

---

## 1. Analogi Pemula: Buruh Harian Umum vs Teknisi Brankas Internal

Untuk memahami perbedaan antara *GitLab SaaS Shared Runner* dan *Self-Hosted On-Premises Runner*, mari gunakan analogi pengelolaan kantor perbankan:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI SHARED RUNNER vs SELF-HOSTED RUNNER                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ SHARED RUNNER PUBLIK (BURUH HARIAN DI PINGGIR JALAN) ]                   │
│  - Masalah: Anda menyewa pekerja umum dari luar kantor untuk merakit meja.  │
│  - Kelemahan:                                                               │
│    1. Pekerja luar TIDAK MEMILIKI KARTU AKSES masuk ke ruang brankas        │
│       rahasia bank Anda di lantai bawah tanah (**Private Data Center**)!    │
│    2. Anda tidak boleh memberikan kunci pintu brankas kepada orang asing!   │
│                                                                             │
│  [ SELF-HOSTED RUNNER ON-PREMISES (TEKNISI INTERNAL RESMI PERUSAHAAN) ]     │
│  - Teknisi internal (Runner yang berjalan di Mac M5 / VM Ubuntu Data Center)│
│    memiliki tanda pengenal resmi dan sudah berada di DALAM gedung bank!     │
│  - Teknisi ini bisa langsung menyambungkan kabel ke server k3s fisik,       │
│    mengakses router MetalLB, dan mengontak database internal tanpa hambatan!│
│                                                                             │
│  [ KEAMANAN POLA LONG-POLLING (TIDAK MEMBUKA PINTU MASUK FIREWALL): ]       │
│  Banyak orang takut: "Apakah memasang Runner di server lokal berarti saya   │
│  harus membuka port router kantor ke internet?"                             │
│  **JAWABANNYA: TIDAK SAMA SEKALI!**                                         │
│  Runner internal bekerja seperti satpam yang setiap 3 detik menelepon       │
│  kantor pusat GitLab via telepon keluar (HTTPS Port 443 Outbound):          │
│  "Halo kantor pusat, apakah ada tugas baru untuk saya hari ini?"            │
│  Tidak ada satu pun port masuk (Inbound) yang dibuka di firewall data center!│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Arsitektur Polling & Pilihan Executor GitLab Runner

GitLab Runner dirancang dengan model *pull-based polling*. Komunikasi selalu dimulai dari Runner lokal menuju GitLab Server melalui koneksi TLS terenkripsi:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ARSITEKTUR KOMUNIKASI OUTBOUND RUNNER ON-PREMISES           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ GITLAB.COM / GITLAB SELF-HOSTED ] (Control Plane SaaS)                   │
│  - Menyimpan Repositori Git, Issue, Merge Request, dan Job Queue.           │
│                                     ▲                                       │
│                                     │ HTTPS Port 443 (Outbound Only!)       │
│                                     │ Long-Polling: "Apakah ada job baru?"  │
│                                     ▼                                       │
│  [ SELF-HOSTED RUNNER ON-PREMISES ] (MacBook M5 / Server Ubuntu Data Center)│
│  - Menerima payload instruksi job dan kredensial sementara (`CI_JOB_TOKEN`).│
│  - Menjalankan lingkungan eksekusi (*Executor*):                            │
│                                                                             │
│    PILIHAN EXECUTOR:                                                        │
│    1. Docker Executor (STANDAR EMAS SRE):                                   │
│       Job dijalankan di dalam kontainer steril sekali pakai yang otomatis   │
│       dihancurkan (*destroyed*) setelah job selesai. Bersih & terisolasi!   │
│                                                                             │
│    2. Shell Executor (TIDAK DISARANKAN UNTUK MULTI-TENANT):                 │
│       Job berjalan langsung di sistem operasi host runner. File sampah dan │
│       environment variable dapat bocor antar-proyek (*Dirty Workspace*)!    │
│                                                                             │
│    3. Kubernetes Executor (UNTUK SKALA BESAR):                              │
│       Setiap job di-spawn sebagai Pod baru di dalam klaster k3s lokal.       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Penargetan Job Menggunakan Tags & Proteksi Lingkungan

Di repositori enterprise, satu runner tidak boleh mengeksekusi semua hal secara sembarangan. Kita menggunakan sistem penandaan (**Tags**) untuk membatasi eksekusi:

### 3.1 Konfigurasi Tags pada `.gitlab-ci.yml`
```yaml
build_arm64:
  stage: build
  tags:
    - macos-m5-arm64  # HANYA boleh dieksekusi oleh Runner Apple Silicon lokal!
  script:
    - uname -m # Menghasilkan: aarch64

deploy_datacenter:
  stage: publish
  tags:
    - baremetal-datacenter # HANYA dieksekusi oleh Runner di dalam VLAN fisik!
  script:
    - ./scripts/deploy-to-local-k3s.sh
```

### 3.2 Tata Kelola Lingkungan (*GitLab Environments*)
Fitur `environment:` memungkinkan SRE memantau riwayat deployment, mengunci akses ke server produksi, dan mengaktifkan tombol rollback langsung dari UI GitLab:

```yaml
deploy_production:
  stage: publish
  environment:
    name: production
    url: https://payment.internal.corp
    on_stop: decommission_production
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual # Wajib persetujuan Lead SRE!
  script:
    - helm upgrade payment ./charts/internal-app -f values-prod.yaml
```

---

## 4. Keamanan Variabel: Protected vs Masked Variables

Kredensial rahasia (seperti Token API, Kunci SSH, atau Password MinIO) disuntikkan ke dalam pipeline melalui antarmuka *Settings -> CI/CD -> Variables*.

SRE wajib menerapkan dua proteksi keamanan mutlak:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DUA PERISAI VARIABEL RAHASIA GITLAB CI                      │
├─────────────────┬───────────────────────────────────────────────────────────┤
│ Fitur Proteksi  │ Mekanisme Keamanan                                        │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ 1. Protected    │ Variabel HANYA AKAN DISUNTIKKAN ke dalam pipeline jika    │
│    Variables    │ job berjalan pada cabang terproteksi (*Protected Branch*  │
│                 │ seperti `main`) atau tag terproteksi (`v*`).              │
│                 │ *Keuntungan:* Developer junior yang membuat branch baru   │
│                 │ tidak bisa mencuri token produksi via Merge Request nakal!│
├─────────────────┼───────────────────────────────────────────────────────────┤
│ 2. Masked       │ Nilai variabel OTOMATIS DISAMARKAN di layar konsol output │
│    Variables    │ menjadi string `[MASKED]`.                                │
│                 │ *Syarat:* Panjang minimal 8 karakter, tidak boleh memuat  │
│                 │ spasi atau karakter baris baru (*newline*).               │
└─────────────────┴───────────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **Mengapa Kunci Privat SSH (`id_ed25519`) Gagal Di-mask?**
> Kunci privat SSH memuat banyak karakter baris baru (*newlines*), sehingga GitLab menolak opsi *Masked*.
> **Solusi SRE:** Simpan kunci privat SSH dalam bentuk **File Variable** (*Type: File*). GitLab akan menuliskan kunci tersebut ke berkas sementara di dalam runner (misal `/tmp/CI_SSH_KEY`) yang hanya dapat dibaca oleh job tersebut dan langsung dihapus setelah job selesai!

---

## 5. Strategi Optimalisasi Cache vs Artifacts

Salah satu faktor penentu keandalan pipeline adalah kecepatan eksekusi. Konfigurasi cache yang cerdas memangkas waktu unduh dependensi dari 10 menit menjadi 10 detik:

### 5.1 Strategi Kunci Cache Dinamis Berbasis Checksum Berkas
Jangan gunakan kunci statis seperti `key: "my-cache"`. Gunakan hashing isi berkas lock agar cache otomatis diperbarui hanya saat dependensi benar-benar berubah:

```yaml
# Cache dependensi Go modules
cache:
  key:
    files:
      - go.sum # Kunci cache HANYA berubah jika go.sum berubah!
  paths:
    - .go/pkg/mod/
  policy: pull-push # Download di awal, update di akhir

test_app:
  stage: test
  image: golang:1.22-alpine
  variables:
    GOPATH: "$CI_PROJECT_DIR/.go"
  cache:
    policy: pull # Job testing HANYA membaca cache, tidak perlu upload ulang!
  script:
    - go test ./...
```

### 5.2 Pengelolaan Masa Berlaku Artefak (*Artifact Expiration*)
Selalu tentukan masa berlaku artefak rilis agar harddisk server runner tidak penuh:
```yaml
artifacts:
  paths:
    - dist/
    - coverage.html
  expire_in: 7 days # Hapus otomatis setelah 7 hari untuk menghemat storage!
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA MASALAH RUNNER & LINGKUNGAN                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: JOB STATUS "STUCK / PENDING: THIS JOB IS WAITING FOR A RUNNER"] │
│  Gejala: Pipeline tidak pernah mulai berjalan, hanya berputar tanpa batas.  │
│  Akar Masalah:                                                              │
│  - Tag pada job (misal: `tags: [macos-m5]`) tidak cocok dengan tag apa pun  │
│    yang didaftarkan pada Runner aktif.                                      │
│  - Atau Runner sedang offline / daemon `gitlab-runner` mati di server.      │
│  Solusi SRE:                                                                │
│  1. Cek status runner: `sudo gitlab-runner status`                          │
│  2. Cek kesesuaian tags di menu *Settings -> CI/CD -> Runners*.             │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "NO SPACE LEFT ON DEVICE" PADA RUNNER ON-PREM ]                 │
│  Gejala: Job build gagal saat mengekstrak Docker image.                     │
│  Akar Masalah:                                                              │
│  - Runner lokal menumpuk ratusan image kontainer lama dan volume gantung    │
│    (*dangling volumes*) yang tidak pernah dibersihkan.                      │
│  Solusi SRE:                                                                │
│  1. Jalankan pembersihan darurat di host runner:                            │
│     docker system prune -af --volumes                                       │
│  2. Pasang cron job harian pada host runner untuk pembersihan rutin:        │
│     0 2 * * * docker system prune -f --filter "until=72h"                   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: DOCKER-IN-DOCKER MTU MISMATCH (NETWORK TIMEOUT) ]               │
│  Gejala: `docker build` di dalam runner on-prem hang saat `apt-get update`. │
│  Akar Masalah:                                                              │
│  - Jaringan bridge Docker di dalam runner memiliki MTU 1500, tetapi jaringan│
│    VLAN fisik data center atau VPN memiliki MTU 1450 (Terjadi fragmentasi!).│
│  Solusi SRE:                                                                │
│  Paksa MTU jaringan dind di `.gitlab-ci.yml`:                               │
│  services:                                                                  │
│    - name: docker:24.0.5-dind                                               │
│      command: ["--mtu=1450"]                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Mendaftarkan Runner baru ke GitLab Server secara interaktif
sudo gitlab-runner register \
  --url "https://gitlab.com/" \
  --registration-token "<REGISTRATION_TOKEN>" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "onprem-baremetal-runner-01" \
  --tag-list "baremetal,arm64,production"

# 2. Memeriksa status dan konektivitas daemon runner ke GitLab
sudo gitlab-runner verify
sudo gitlab-runner status

# 3. Menjalankan runner dalam mode debugging interaktif di terminal
sudo gitlab-runner --debug run

# 4. Memverifikasi pemakaian disk ruang kerja pada host runner
sudo du -sh /var/lib/gitlab-runner/builds/*
docker system df

# 5. Membersihkan kontainer dan cache lama yang menggantung di host runner
docker container prune -f
docker volume prune -f
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa arsitektur komunikasi *Outbound Long-Polling* pada GitLab Runner dianggap sangat menguntungkan dari perspektif keamanan jaringan data center privat (*Private Network Security*)?
- A. Karena Runner otomatis mengubah alamat IP publik setiap menit.
- B. Karena tim SRE tidak perlu membuka satu pun port masuk (*Inbound Port*) pada firewall perimeter data center; seluruh koneksi dimulai dari Runner lokal menuju GitLab SaaS melalui port keluar standar HTTPS (Port 443).
- C. Karena Runner berjalan tanpa memerlukan koneksi internet.
- D. Agar kecepatan transfer data meningkat menjadi 10 Gbps.

*Kunci Jawaban:* **B**. Pola outbound-only menjaga keamanan perimeter jaringan lokal dan mencegah server internal terpapar serangan dari internet luar.

### Soal 2 (Pilihan Ganda)
Apa yang membedakan *Docker Executor* dengan *Shell Executor* saat menjalankan pekerjaan di GitLab Runner?
- A. Docker executor hanya bisa digunakan untuk bahasa pemrograman C++.
- B. Docker executor menjalankan setiap job di dalam kontainer terisolasi yang bersih dan otomatis dihancurkan setelah job selesai, sedangkan Shell executor menjalankan instruksi langsung di sistem operasi host yang berisiko meninggalkan file sampah dan konflik antar-proyek.
- C. Shell executor otomatis mematikan server jika terjadi error.
- D. Tidak ada perbedaan sama sekali.

*Kunci Jawaban:* **B**. Docker executor menjamin lingkungan eksekusi yang bersih (*clean-room environment*) dan mencegah kontaminasi antar-pekerjaan.

### Soal 3 (Pilihan Ganda)
Di pengaturan proyek GitLab, sebuah variabel rahasia ditandai sebagai **Protected Variable**. Pada kondisi manakah variabel tersebut akan disuntikkan ke dalam lingkungan job pipeline?
- A. Pada semua pipeline yang dijalankan oleh developer mana pun.
- B. HANYA pada pipeline yang dieksekusi pada cabang terproteksi (*Protected Branch* seperti `main`) atau tag terproteksi (*Protected Tag*), mencegah penyalahgunaan rahasia dari cabang eksperimental.
- C. Hanya jika job memiliki nama `deploy`.
- D. Hanya jika ukuran filenya di bawah 10 KB.

*Kunci Jawaban:* **B**. Protected variables adalah instrumen tata kelola akses untuk mengisolasi kredensial produksi dari cabang pengembangan biasa.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Di sebuah private data center, pipeline GitLab CI sering kali gagal di job build Docker dengan pesan kesalahan `ERROR: failed to solve: failed to do request: Head https://registry...: dial tcp: i/o timeout`. Setelah diinvestigasi, ternyata koneksi jaringan fisik data center menggunakan MTU 1420 karena melewati enkripsi tunnel IPsec GRE, sedangkan service Docker-in-Docker (dind) pada runner default menggunakan MTU 1500. Jelaskan apa yang terjadi pada paket jaringan tersebut dan bagaimana cara mengatasinya secara permanen pada konfigurasi `.gitlab-ci.yml`!

*Panduan Solusi:*
1. **Analisis Akar Masalah (Packet Fragmentation & MTU Blackhole):**
   Ketika service Docker-in-Docker mengirimkan paket TCP berukuran 1500 bytes (MTU standar) melewati antarmuka jaringan data center yang memiliki batas MTU 1420 bytes, router jaringan menuntut fragmentasi paket. Jika flag *Do Not Fragment* (DF) aktif atau router membuang paket ICMP *Destination Unreachable (Fragmentation Needed)*, paket akan lenyap tanpa jejak (*MTU Black Hole*), menyebabkan koneksi HTTP/TLS menggantung (*hang*) hingga mencapai waktu *timeout*.
2. **Solusi Konfigurasi Permanen pada `.gitlab-ci.yml`:**
   Paksa daemon Docker-in-Docker pada runner untuk menggunakan MTU yang selaras dengan jalur jaringan fisik data center (1420 bytes):
   ```yaml
   services:
     - name: docker:24.0.5-dind
       command:
         - "--mtu=1420" # Menyelaraskan MTU kontainer dengan tunnel IPsec lokal!
   ```
   Langkah ini menjamin seluruh paket jaringan yang dikirim oleh proses build kontainer muat melewati terowongan jaringan tanpa mengalami pemotongan paket atau *timeout*.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Pipeline Stages, Jobs, Rules, dan Gate](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.1-gitlab-ci-cd/01-pipeline-stages-jobs-rules.md) — Memahami penyusunan alur dan aturan graf job.
- **Topik Berikutnya:**
  - [03 — Pipeline IaC dan Ansible dengan Approval Gate](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.1-gitlab-ci-cd/03-iac-ansible-pipeline-approval.md) — Menerapkan runner ini untuk mengotomasi OpenTofu dan Ansible.
  - [LAB-01 — CI Lint, Test, Build, dan Push](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.1-gitlab-ci-cd/lab/LAB-01-ci-lint-test-build-push.md) — Praktik hands-on setup runner dan pipeline di terminal.
