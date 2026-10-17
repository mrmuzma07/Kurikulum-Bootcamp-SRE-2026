# 02 — Git Workflow: Conventional Commits, Trunk-Based Development & GitLab

> **Ringkasan:** Membangun disiplin kerja tim SRE berstandar industri: menulis catatan perubahan yang bermakna (*Conventional Commits*), mengadopsi alur percabangan cepat (*Trunk-Based Development*), menjaga kebersihan repositori dari kebocoran berkas rahasia (`.gitignore`), serta mengelola kolaborasi infrastruktur dan peninjauan kode (*Code Review*) melalui GitLab Merge Request dan Milestone.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan menyelesaikan simulasinya di terminal serta GitLab, kamu akan mampu:
- Menulis pesan *commit* yang terstruktur dan mudah dipahami manusia maupun mesin mengikuti spesifikasi **Conventional Commits**.
- Menjelaskan filosofi **Trunk-Based Development** dan mengapa metode ini menjadi standar tim *High-Performing Engineering* dibanding *Git Flow* tradisional.
- Memproteksi repositori dari kebocoran kredensial, token API, dan file sampah sistem operasi menggunakan konfigurasi **`.gitignore`** yang tepat.
- Menjalankan alur kerja kolaboratif **GitLab Merge Request (MR)**: membuat branch fitur pendek, mengajukan usulan, melakukan peninjauan kode (*Code Review*), menerapkan teknik *Squash & Merge*, serta membersihkan branch usang.
- Mengelola tugas operasional SRE secara terstruktur menggunakan fitur **GitLab Issues** dan **Milestones** dengan kata kunci penutup otomatis (*auto-closing keywords*).
- Menata struktur direktori repositori portofolio `sre-bootcamp` yang rapi sebagai fondasi GitOps di modul-modul berikutnya.

---

## 1. Konsep & Arsitektur: Mengapa Workflow & Pesan Commit Begitu Penting?

### Tragedi Jam 2 Pagi: Ketika Buku Sejarah Tak Berguna
Bayangkan skenario nyata berikut:
Jam menunjukkan pukul 02.15 dini hari. Ponselmu berdering kencang karena alarm PagerDuty berbunyi: *Layanan otentikasi login di server production tiba-tiba mati total (HTTP 500).*

Sebagai SRE yang sedang bertugas (*on-call*), kamu membuka terminal dan mengetik `git log` pada repositori konfigurasi untuk mencari tahu perubahan apa yang baru saja diterapkan oleh tim beberapa jam lalu. Di layar terminal, kamu melihat catatan commit seperti ini:

```text
a8f12c3 update
7b4e910 fix bug
1c09d22 revisi lagi
99a712f coba-coba
43f8e01 bismillah jalan
```

Apakah riwayat di atas membantumu menemukan masalah? **Sama sekali tidak!**
Kamu tidak tahu file apa yang diubah, apa tujuan perubahannya, sistem mana yang terdampak, atau siapa yang harus dihubungi. Kamu terpaksa membaca ribuan baris perbedaan kode (*diff*) satu per satu di bawah tekanan waktu sementara kerugian finansial perusahaan terus bertambah.

Bandingkan jika riwayatnya tertulis seperti ini:

```text
7e8a910 fix(caddy): perbaiki directive reverse-proxy timeout pada auth service
3b11f42 feat(ansible): upgrade dependensi modul pam ke versi 2.4
d89a011 docs(runbook): perbarui SOP penanganan node disk full
```

Hanya dalam hitungan detik, kamu langsung tahu bahwa commit `7e8a910` baru saja menyentuh pengaturan proxy autentikasi. Kamu bisa langsung menginspeksi commit tersebut atau melakukan `git revert` untuk memulihkan layanan seketika.

> 🩺 **Analogi Rekam Medis Rumah Sakit:**
> Git commit history adalah **Buku Catatan Rekam Medis Pasien**. Dokter tidak boleh menulis catatan: *"Pasien diobati"* atau *"Kasih obat lagi"*. Dokter wajib mencatat secara presisi: *"Pemberian Paracetamol 500mg via oral untuk meredakan demam 39°C"*. Di dunia SRE, server production adalah pasienmu, dan Git log adalah rekam medis operasinya.

---

### Dua Konsumen Riwayat Git: Manusia dan Mesin
Pesan commit yang terstandarisasi memiliki dua manfaat besar:
1. **Untuk Manusia (Tim & Dirimu di Masa Depan):** Menjelaskan *alasan* (konteks) di balik perubahan konfigurasi, bukan sekadar apa yang diubah (karena apa yang diubah sudah terlihat di `git diff`).
2. **Untuk Mesin (Otomasi CI/CD):** Script otomatis dapat membaca pesan commit untuk:
   - Membuat catatan rilis (*Changelog*) otomatis tanpa perlu diketik manual.
   - Menentukan kenaikan versi aplikasi secara otomatis (**Semantic Versioning / SemVer**: *Major*, *Minor*, *Patch*).
   - Menjalankan pipeline pengujian yang hanya relevan dengan modul yang disentuh.

---

## 2. Standar Industri: Conventional Commits

**Conventional Commits** adalah konvensi penulisan pesan commit formal yang paling luas digunakan di industri perangkat lunak dan DevOps dunia.

### Struktur Dasar
```text
<type>(<scope opsional>): <deskripsi singkat imperatif>

[badan pesan / body penjelasan panjang opsional]

[footer catatan kaki opsional / nomor issue / breaking change]
```

---

### 1. Memahami Elemen `<type>` (Jenis Perubahan)
Tipe memberitahu pembaca jenis tindakan apa yang dilakukan pada commit tersebut. Berikut tabel tipe standar beserta contoh nyata dalam konteks SRE:

| Tipe (`type`) | Kapan Digunakan? | Contoh Konkret SRE / DevOps |
|---|---|---|
| **`feat`** | Menambah fitur atau kapabilitas baru pada sistem. | `feat(mimir): aktifkan retention metrik selama 30 hari` |
| **`fix`** | Memperbaiki bug, kerusakan, atau kegagalan konfigurasi. | `fix(ansible): k3s agent gagal join pada mesin ARM64` |
| **`docs`** | Hanya mengubah dokumentasi (README, runbook, arsitektur). | `docs(runbook): tambah panduan mitigasi OOMKilled` |
| **`refactor`**| Mengubah susunan kode/skrip tanpa mengubah fungsi luarnya. | `refactor(backup): pisahkan fungsi kompresi ke file helper terpisah` |
| **`chore`** | Tugas pemeliharaan rutin yang tidak menyentuh kode inti. | `chore: perbarui daftar ekstensi pada .gitignore` |
| **`ci`** | Mengubah file konfigurasi pipeline CI/CD. | `ci(gitlab): tambahkan tahap linting ansible pada MR` |
| **`perf`** | Perubahan konfigurasi untuk mendongkrak performa sistem. | `perf(prometheus): ubah scraping interval dari 15s ke 60s` |
| **`test`** | Menambah skrip pengujian atau memperbaiki skrip tes yang ada. | `test(helm): tambahkan validasi template menggunakan helm lint` |

---

### 2. Memahami Elemen `(<scope>)` (Area yang Terdampak)
*Scope* ditaruh di dalam tanda kurung untuk memperjelas modul atau komponen mana yang disentuh. Scope bersifat opsional, namun sangat dianjurkan agar riwayat mudah difilter.
- Contoh di infrastruktur: `(ansible)`, `(opentofu)`, `(k3s)`, `(metallb)`, `(caddy)`.
- Contoh di aplikasi: `(auth)`, `(database)`, `(api)`.

---

### 3. Menulis `<deskripsi singkat>` yang Baik
Ikuti 3 aturan emas penulisan ringkasan commit:
1. **Gunakan kata kerja imperatif (perintah):** Tulis `"tambah"`, `"perbaiki"`, `"ubah"`, bukan `"menambahkan"` atau `"sudah diperbaiki"`. 
   *(Aturan praktis: Lengkapi kalimat: "Jika commit ini diterapkan, commit ini akan [isi pesanmu]").*
2. **Huruf kecil di awal dan tanpa tanda titik di akhir:** 
   - ✅ `feat(dns): konfigurasi upstream resolver ke Cloudflare`
   - ❌ `feat(dns): Konfigurasi upstream resolver ke Cloudflare.`
3. **Maksimal 50–72 karakter:** Jaga agar judul tetap padat dan tidak terpotong di tampilan antarmuka GitLab/GitHub.

---

### 4. Perubahan Perusak: Breaking Change (`!`)
Jika perubahan yang kamu buat menyebabkan konfigurasi lama tidak lagi kompatibel (misalnya: mengubah port default, menghapus variabel penting, atau mengganti arsitektur OS), kamu **WAJIB** menandainya sebagai **Breaking Change**.

Ada dua cara menandainya:
1. Menambahkan tanda seru `!` tepat sebelum titik dua:
   ```text
   feat(api)!: ganti format respon metrik dari XML ke JSON
   ```
2. Menyertakan blok `BREAKING CHANGE:` di bagian footer:
   ```text
   feat(api): ganti format respon metrik ke JSON

   Format XML resmi dipensiunkan demi efisiensi bandwidth.

   BREAKING CHANGE: Semua scraper lama yang mengonsumsi XML akan gagal.
   ```

---

### Contoh Pesan Commit Lengkap yang Ideal
```text
fix(ansible): tangani kegagalan instalasi k3s pada arsitektur ARM64

Binary instalasi k3s sebelumnya selalu mengunduh arsitektur amd64 karena 
fakta sistem operasi tidak dipetakan dengan benar pada variabel role. 
Perubahan ini menambahkan pemetaan eksplisit ansible_architecture ke arm64.

Closes #42
```

---

## 3. Strategi Percabangan: Trunk-Based Development vs Git Flow

Bagaimana cara tim berkolaborasi tanpa saling menginjak pekerjaan rekan kerja? Kita membutuhkan **Branching Strategy** (Strategi Percabangan).

```
Trunk-Based Development:
        main (Trunk / Rel Utama — Selalu Stabil & Siap Deploy)
  ───────●──────────●──────────●──────────●──────────────●──────►
          \        /            \        /
           ●──────●              ●──────●   (Branch fitur pendek, 1–2 hari)
         fitur-backup          fitur-caddy
```

### Mengapa SRE Bootcamp Memilih Trunk-Based Development?
Di dunia industri, ada dua mazhab strategi percabangan yang paling terkenal:

| Aspek Pembanding | Trunk-Based Development (Standar Modern) | Git Flow (Mazhab Tradisional) |
|---|---|---|
| **Cabang Utama** | Hanya ada satu cabang utama yang abadi: **`main`**. | Banyak cabang abadi: `main`, `develop`, `release-*`, `hotfix-*`. |
| **Umur Branch Fitur** | **Sangat pendek (1–2 hari)**, langsung di-merge kembali ke `main`. | **Panjang (berminggu-minggu)** hingga satu siklus rilis selesai. |
| **Ukuran Perubahan** | Perubahan kecil-kecil (*small batch size*), mudah di-review. | Perubahan raksasa (*large batch size*), sulit di-review. |
| **Potensi Konflik** | Sangat kecil karena integrasi dilakukan setiap hari. | **Sangat tinggi (*Merge Hell*)** saat menyatukan cabang tua. |
| **Kesesuaian SRE** | **Sangat Ideal.** Selaras dengan prinsip CI/CD dan GitOps. | Tidak cocok untuk tim lincah (*agile*) dan rilis harian. |

> 💡 **Prinsip Utama Trunk-Based:**
> 1. Cabang `main` (disebut juga *Trunk*) harus **selalu dalam keadaan sehat, lulus tes, dan siap diterapkan ke server kapan saja**.
> 2. Jangan pernah membiarkan branch fitur berumur lebih dari 2 hari. Pecah tugas besar menjadi sub-tugas kecil yang dapat digabungkan secara bertahap.

---

## 4. Higienitas Repositori: Menjaga Rahasia dengan `.gitignore`

### Bahaya Fatal: Sekali Bocor, Selamanya di Catatan Sejarah
Salah satu kesalahan paling fatal bagi pemula adalah mengunggah file rahasia (*secret*) ke repositori Git publik. File seperti:
- Kunci privat SSH (`id_rsa`, `*.pem`)
- File environment berisi password database (`.env`)
- Token otentikasi GitLab / AWS Secret Access Key
- File state Terraform / OpenTofu (`*.tfstate`) yang memuat data sensitif dalam bentuk teks polos.

> 🚨 **PERINGATAN KERAS:**
> Menghapus file rahasia dengan perintah `rm .env` lalu melakukan `git commit` **TIDAK MENGHAPUS file tersebut dari riwayat Git!** File tersebut tetap tersimpan di dalam snapshot commit masa lalu dan dapat dilihat oleh siapa pun yang mengunduh repositorimu. Jika kredensial terlanjur ter-push ke GitLab, **kredensial tersebut harus segera diganti/dibatalkan (*revoked/rotated*) saat itu juga**!

---

### Cara Kerja `.gitignore`
`.gitignore` adalah sebuah file teks sederhana yang ditaruh di folder paling atas repositori. Git membaca file ini untuk mengetahui pola nama file atau folder apa saja yang **harus diabaikan secara total** agar tidak pernah masuk ke *Working Directory tracking*, *Staging Area*, maupun riwayat *Commit*.

### Aturan Sintaks `.gitignore`
- Baris yang diawali `#` adalah komentar.
- Tanda bintang `*` adalah *wildcard* (mencocokkan karakter apa pun). Misal: `*.log` mengabaikan semua file berakhiran `.log`.
- Garis miring di akhir `/` mengindikasikan sebuah folder. Misal: `node_modules/`.
- Garis miring di awal `/` mencocokkan file hanya di tingkat folder saat ini.
- Tanda seru `!` di awal digunakan untuk pengecualian (jangan abaikan file ini).

### Template Standar `.gitignore` untuk Bootcamp SRE
Berikut adalah konfigurasi `.gitignore` komprehensif yang wajib kamu pasang di repositori latihanmu:

```gitignore
# ==============================================================================
# 1. KREDENSIAL & RAHASIA (JANGAN PERNAH DI-COMMIT KE GIT!)
# ==============================================================================
*.pem
*.key
*.pub
*.crt
.env
.env.*
!*.env.example
ansible-vault-password*
id_rsa*
id_ed25519*

# ==============================================================================
# 2. FILE ARTIFAK SISTEM OPERASI & EDITOR
# ==============================================================================
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Editor & IDE
.idea/
.vscode/
*.swp
*.swo
*~

# ==============================================================================
# 3. RUNTIME & ARTIFAK BAHASA PEMROGRAMAN
# ==============================================================================
# Python (sering dipakai Ansible & scripting)
__pycache__/
*.py[cod]
*$py.class
.venv/
venv/
ENV/

# Node.js
node_modules/

# ==============================================================================
# 4. INFRASTRUCTURE AS CODE (OPENTOFU / TERRAFORM)
# ==============================================================================
# State lokal mengandung password plaintext!
*.tfstate
*.tfstate.*
*.tfvars
!example.tfvars
.terraform/
.terraform.lock.hcl.backup

# ==============================================================================
# 5. LOG & BERKAS SEMENTARA
# ==============================================================================
*.log
/tmp/
```

---

## 5. Kolaborasi Tim: GitLab Merge Request (MR) & Code Review

Di lingkungan profesional, seorang engineer tidak langsung melakukan `git push` ke branch `main`. Semua perubahan diusulkan terlebih dahulu melalui antarmuka **Merge Request (MR)** (istilah ini setara dengan *Pull Request* di GitHub).

```text
 ┌───────────────┐     git push      ┌─────────────────────────┐
 │ Branch Fitur  │ ────────────────► │ GitLab Remote           │
 │ (Lokal)       │                   │ (Buka Merge Request UI) │
 └───────────────┘                   └─────────────────────────┘
                                                  │
                                                  ▼
                                     ┌─────────────────────────┐
                                     │ Code Review & Otomasi   │
                                     │ - Diskusi baris kode    │
                                     │ - Pipeline CI Lulus     │
                                     └─────────────────────────┘
                                                  │
                                                  ▼ (Squash & Merge)
                                     ┌─────────────────────────┐
                                     │ Branch main Terupdate!  │
                                     └─────────────────────────┘
```

---

### Alur 8 Langkah Siklus Hidup Merge Request

1. **Sinkronkan branch lokal:** Pastikan branch `main` lokalmu selalu mengambil versi terbaru:
   ```bash
   git switch main
   git pull origin main
   ```
2. **Buat branch fitur pendek:**
   ```bash
   git switch -c feat-m0.3-reverse-proxy
   ```
3. **Kerjakan tugas & buat commit konvensional:**
   ```bash
   # Lakukan pengeditan file...
   git add Caddyfile
   git commit -m "feat(caddy): pasang konfigurasi reverse proxy auth"
   ```
4. **Unggah branch ke GitLab remote:**
   ```bash
   git push -u origin feat-m0.3-reverse-proxy
   ```
5. **Buka Merge Request di GitLab Web UI:**
   - Kunjungi repositorimu di browser. GitLab akan menampilkan tombol banner biru: **`Create merge request`**.
   - **Source branch:** `feat-m0.3-reverse-proxy` ➔ **Target branch:** `main`.
   - Isi judul sesuai Conventional Commit: `feat(caddy): pasang konfigurasi reverse proxy auth`.
   - Di kolom deskripsi, jelaskan latar belakang perubahan dan kaitkan dengan tiket issue (misal: `Closes #12`).
6. **Peninjauan Kode (*Code Review*):**
   - Rekan kerja (atau mentor) membuka tab **Changes** di GitLab untuk membaca baris demi baris kode yang diubah.
   - Komentar dapat disematkan langsung pada baris tertentu untuk meminta perbaikan atau penjelasan.
7. **Persetujuan & Penggabungan (*Squash and Merge*):**
   - Setelah semua komentar terselesaikan dan pengujian CI/CD hijau, tombol **Merge** akan aktif.
8. **Pembersihan Lokal:**
   - Kembali ke laptopmu, tarik perubahan terbaru ke `main`, lalu hapus branch fitur yang sudah selesai:
   ```bash
   git switch main
   git pull origin main
   git branch -d feat-m0.3-reverse-proxy
   ```

---

### Memahami Pilihan Opsi Penggabungan di GitLab UI

Saat kamu hendak menekan tombol hijau Merge di GitLab, terdapat opsi kotak centang penting:

```
[✔] Squash commits
[✔] Delete source branch
```

1. **Squash Commits (Sangat Direkomendasikan):**
   - Selama bekerja di branch fitur, kamu mungkin membuat banyak commit kecil yang berantakan, misalnya: `wip`, `typo`, `coba lagi`.
   - Jika fitur di-merge biasa, semua commit sampah tersebut akan mengotori riwayat branch `main`.
   - Dengan mencentang **Squash**, GitLab akan memadatkan (*melipat*) 10 commit kecil tersebut menjadi **tepat 1 buah commit bersih** di branch `main` dengan pesan Conventional Commit yang rapi!
2. **Delete Source Branch:**
   - Menghapus branch fitur di remote GitLab secara otomatis setelah berhasil digabungkan. Ini menjaga agar repositori tidak dipenuhi ratusan branch usang (*stale branches*).

---

### Etika & Seni Melakukan Code Review (Bahkan untuk Diri Sendiri)
Meskipun saat ini kamu sedang belajar mandiri di bootcamp dan me-review Merge Request milikmu sendiri, biasakan mempraktikkan standar kerja profesional:
- **Tinjau Tab *Changes* Sebelum Menekan Merge:** Bacalah seluruh perbedaan baris kode (*diff*). Sering kali kita baru menyadari ada baris komentar debug yang terlupa atau spasi berantakan saat membacanya di antarmuka web.
- **Ajukan Pertanyaan, Bukan Menghakimi:** Saat me-review kode rekan kerja, gunakan pertanyaan terbuka: *"Apakah port 8080 ini sudah dialokasikan di firewall?"* alih-alih *"Konfigurasi ini salah"*.
- **Prinsip Blameless:** Fokuslah pada ketahanan sistem dan otomatisasi, bukan menyalahkan kelalaian individu.

---

## 6. Manajemen Pelacakan Tugas: Issue & Milestone di GitLab

Bagaimana tim SRE melacak ratusan tugas, perbaikan insiden, dan target proyek tanpa ada yang tercecer? Jawabannya adalah melalui **Issues** dan **Milestones**.

```text
Milestone: "Fase 0 — Fondasi SRE" (Target: Minggu ke-2 Selesai)
 ├── Issue #1: LAB-01 VM OrbStack Linux ───► Terselesaikan oleh MR !1
 ├── Issue #2: LAB-02 Skrip Otomasi Backup ─► Terselesaikan oleh MR !2
 ├── Issue #3: LAB-03 Investigasi Proses ──► Terselesaikan oleh MR !3
 └── Issue #4: LAB-01 Repositori Git ──────► Terselesaikan oleh MR !4
```

---

### 1. GitLab Issue: Satuan Tunggal Pekerjaan
Setiap tugas, lab, atau laporan bug harus memiliki **1 tiket Issue**.

**Anatomi Tiket Issue yang Baik:**
- **Judul:** Jelas dan terukur, misal: `[M0.2 - LAB-01] Konfigurasi Reverse Proxy Caddy`.
- **Deskripsi:** Berisi konteks dan daftar kriteria penerimaan (*Acceptance Criteria* / checklist kotak centang):
  ```markdown
  ## Deskripsi Tugas
  Mengonfigurasi web server Caddy sebagai reverse proxy untuk dua domain internal.

  ## Acceptance Criteria (Kriteria Keberhasilan)
  - [ ] Layanan Caddy berjalan aktif di bawah supervisi systemd
  - [ ] Domain `app1.internal.lab` dapat diakses via curl
  - [ ] File Caddyfile tersimpan rapi di direktori `m0.2/lab/`
  ```

---

### 2. Keajaiban Penutupan Otomatis (Auto-Closing Pattern)
Kamu tidak perlu repot-repot membuka halaman Issue untuk mengeklik tombol *"Close issue"* secara manual setelah pekerjaanmu selesai.

GitLab memiliki fitur otomatis: jika kamu menuliskan salah satu kata kunci khusus diikuti nomor issue di deskripsi Merge Request milikmu, **GitLab akan otomatis menutup issue tersebut saat Merge Request berhasil di-merge**!

Daftar kata kunci ajaib (*magic keywords*):
- `Closes #12`
- `Fixes #5`
- `Resolves #8`

*Contoh penulisan di deskripsi Merge Request:*
```markdown
## Ringkasan Perubahan
Menambahkan konfigurasi reverse proxy Caddy sesuai spesifikasi modul 0.2.

Closes #12
```

---

### 3. GitLab Milestone: Mengelompokkan Target Bersama
Jika Issue adalah tugas harian, maka **Milestone** adalah tujuan besar (seperti fase belajar atau siklus rilis sprint).

- Di bootcamp ini, setiap fase pembelajaran memiliki satu Milestone, contoh: **`Fase 0 — Fondasi SRE`**.
- Masukkan semua Issue modul 0.1, 0.2, dan 0.3 ke dalam Milestone tersebut.
- GitLab akan menampilkan bilah persentase kemajuan (*Progress Bar*) yang bergerak naik setiap kali sebuah Issue selesai dikerjakan. Milestone ditutup saat seluruh lab fase tersebut tuntas 100%.

---

## 7. Cetak Biru: Struktur Repositori Portofolio `sre-bootcamp`

Di akhir Modul 0.3 ini, kamu akan membangun sebuah repositori resmi bernama **`sre-bootcamp`** di akun GitLab pribadimu. Repositori ini akan menjadi bukti portofolio nyata yang dinilai oleh mentor dan calon pemberi kerja.

Berikut adalah struktur direktori standar yang harus dibangun:

```text
sre-bootcamp/
├── .gitignore                     # Aturan proteksi file rahasia & sampah sistem
├── README.md                      # Dokumentasi profil, daftar isi, & status modul
│
├── m0.1/                          # Modul 0.1: Linux & Shell Scripting
│   ├── README.md                  # Rangkuman capaian modul 0.1
│   └── lab/
│       ├── backup.sh              # Skrip backup idempoten
│       ├── backup.service         # Systemd unit service
│       ├── backup.timer           # Systemd timer terjadwal
│       └── lab03-report.md        # Laporan investigasi proses zombie/CPU leak
│
├── m0.2/                          # Modul 0.2: Jaringan Komputer & Troubleshooting
│   ├── README.md                  # Rangkuman capaian modul 0.2
│   └── lab/
│       ├── Caddyfile              # Konfigurasi reverse proxy & TLS lokal
│       ├── dnsmasq-lab.conf       # Konfigurasi DNS server lokal
│       ├── hosts-lab.txt          # Pemetaan static domain
│       └── lab03-trace-report.md  # Laporan diagnosa latensi TCP/DNS
│
└── m0.3/                          # Modul 0.3: Git & Kolaborasi (Modul Ini)
    ├── README.md                  # Rangkuman capaian modul 0.3
    └── lab/
        └── gitlab-setup-report.md # Bukti konfigurasi SSH key & tautan repo remote
```

> 🎯 **Pola Pengerjaan Berbasis Merge Request:**
> Jangan memasukkan semua file di atas sekaligus ke dalam branch `main`!
> Praktikkan alur profesional:
> - Pindahkan hasil lab modul 0.1 melalui **Merge Request #1** (dengan squash commit).
> - Pindahkan hasil lab modul 0.2 melalui **Merge Request #2**.
> - Pindahkan hasil lab modul 0.3 melalui **Merge Request #3**.
> Dengan cara ini, buku riwayat Git di repositorimu akan mencerminkan proses belajar yang terstruktur dan membanggakan.

---

## 8. Hands-On Mandiri Terpandu (20 Menit)

Mari simulasikan alur kerja Conventional Commit, branching, dan teknik Squash secara lokal di terminal komputermu sebelum mencobanya di GitLab.

```bash
# --- LANGKAH 1: Inisialisasi Proyek Simulasi ---
cd ~
mkdir simulasi-workflow && cd simulasi-workflow
git init
echo "# Repositori Simulasi Workflow" > README.md

# Pasang .gitignore standar sederhana
cat << 'EOF' > .gitignore
*.env
*.log
.DS_Store
EOF

git add .
git commit -m "chore: inisialisasi repositori dan konfigurasi .gitignore"

# --- LANGKAH 2: Latihan Menulis Conventional Commits ---
# Commit dokumentasi
echo "Panduan alur kerja tim SRE berbasis Trunk-Based." >> README.md
git add README.md
git commit -m "docs(readme): tambahkan penjelasan alur kerja"

# Commit penambahan fitur konfigurasi
echo "LOG_LEVEL=info" > app.conf
git add app.conf
git commit -m "feat(config): tambahkan konfigurasi log level aplikasi"

# Commit perbaikan bug
echo "LOG_LEVEL=debug" > app.conf
git add app.conf
git commit -m "fix(config): ubah log level ke debug untuk kebutuhan lab"

# Periksa catatan sejarah
git log --oneline

# --- LANGKAH 3: Simulasi Branch Fitur & Banyak Commit Kecil (WIP) ---
git switch -c feat-monitoring

echo "probe_interval: 5s" >> app.conf
git add app.conf && git commit -m "wip: coba interval probe 5 detik"

echo "probe_timeout: 1s" >> app.conf
git add app.conf && git commit -m "wip: atur timeout probe"

echo "probe_retries: 3" >> app.conf
git add app.conf && git commit -m "wip: perbaiki nilai retries"

# Lihat betapa berantakannya riwayat di branch fitur
git log --oneline -n 3

# --- LANGKAH 4: Menyatukan dengan Teknik Squash (Simulasi GitLab Squash) ---
# Kembali ke cabang utama
git switch main

# Lakukan penggabungan dengan metode squash (mengumpulkan semua perubahan tanpa commit aslinya)
git merge --squash feat-monitoring

# Periksa status: semua perubahan kini terkumpul rapi di Staging Area!
git status

# Ambil satu snapshot bersih dengan format Conventional Commit:
git commit -m "feat(monitoring): konfigurasi parameter health check probe lengkap"

# Hapus branch fitur yang sudah selesai
git branch -D feat-monitoring

# --- LANGKAH 5: Verifikasi Hasil Akhir ---
# Perhatikan riwayat main yang sangat rapi dan bebas dari commit sampah 'wip'
git log --oneline --graph

# --- LANGKAH 6: Bersihkan Folder Simulasi ---
cd ..
rm -rf simulasi-workflow
```

---

## 9. Pola Kegagalan (Failure Modes) & Diagnosa SRE

Berikut adalah 4 masalah yang paling sering dihadapi oleh pemula saat berkolaborasi menggunakan Git dan GitLab:

### Kasus 1: Terlanjur Melakukan Perubahan Langsung di Branch `main`
- **Gejala:** Kamu sudah mengetik kode selama 2 jam, lalu saat mengecek `git status`, kamu sadar kamu sedang berdiri di branch `main`, padahal seharusnya membuat branch fitur baru!
- **Solusi Tanpa Panik (Jika Belum Di-Commit):**
  Cukup buat branch baru saat ini juga dan berpindahlah ke sana. Semua ketikan yang belum di-commit akan otomatis ikut terbawa ke branch baru:
  ```bash
  git switch -c feat-tugas-baru
  # Sekarang kamu sudah berada di branch fitur, silakan git add dan git commit!
  ```
- **Solusi (Jika Sudah Terlanjur Di-Commit di `main` Lokal tapi Belum Di-Push):**
  Pindahkan commit tersebut ke branch baru, lalu mundurkan branch `main`:
  ```bash
  # 1. Buat branch baru di posisi commit saat ini:
  git branch feat-tugas-baru

  # 2. Mundurkan branch main 1 langkah ke belakang:
  git reset --hard HEAD~1

  # 3. Pindah ke branch baru untuk melanjutkan kerja:
  git switch feat-tugas-baru
  ```

---

### Kasus 2: Konflik pada Merge Request di Antarmuka GitLab
- **Gejala:** Di halaman GitLab MR muncul kotak merah bertuliskan: *"Merge blocked: Merge conflicts must be resolved"*.
- **Penyebab:** Selama kamu mengerjakan branch fiturnmu, ada rekan tim lain yang sudah terlebih dahulu me-merge perubahannya ke branch `main`, dan perubahannya menyentuh baris file yang sama dengan kodemu.
- **Solusi SRE:** Selesaikan konflik di laptop lokalmu, bukan di web browser:
  ```bash
  # 1. Pindah ke branch fiturnmu
  git switch feat-tugas-baru

  # 2. Tarik kode main terbaru ke dalam branch fiturnmu
  git pull origin main

  # 3. Buka editor, selesaikan penanda konflik (<<<<<<<, =======, >>>>>>>)
  # 4. Tandai selesai dan commit
  git add <file-konflik>
  git commit -m "fix(merge): selesaikan konflik sinkronisasi dengan main"

  # 5. Push kembali ke branch fiturnmu
  git push origin feat-tugas-baru
  # GitLab akan mendeteksi konflik sudah hilang dan tombol Merge akan kembali aktif!
  ```

---

### Kasus 3: Pesan Commit Ditolak oleh Sistem Validasi Otomatis
- **Gejala:** Saat mengetik `git commit`, muncul pesan error dari tools penegak aturan (*pre-commit hook* atau *commitlint*) bahwa pesan commit-mu tidak sah.
- **Penyebab:** Kamu menulis pesan di luar spesifikasi Conventional Commits (misal: `Update file`, huruf kapital di awal, atau lupa menyertakan tanda titik dua `:`).
- **Solusi:** Perbaiki pesan commit menggunakan format yang benar:
  ```bash
  git commit -m "fix(auth): perbaiki kesalahan parsing token jwt"
  ```

---

### Kasus 4: Branch Fitur Tertinggal Jauh dari `main` (*Stale Branch*)
- **Gejala:** Branch fiturmu sudah tidak disentuh selama 1 minggu, sementara branch `main` di server sudah maju puluhan commit.
- **Solusi:** Lakukan sinkronisasi rutin setiap pagi sebelum mulai mengetik kode:
  ```bash
  git switch main
  git pull origin main
  git switch feat-tugas-lama
  git merge main
  ```

---

## 10. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Konsep & Perintah Workflow

| Aktivitas | Perintah / Solusi Praktis |
|---|---|
| **Format pesan commit standar** | `<type>(<scope>): <deskripsi imperatif>` |
| **Menandai perubahan besar non-kompatibel** | Tambahkan `!` setelah scope, misal: `feat(api)!: ganti port` |
| **Mencegah file rahasia ter-commit** | Daftarkan polanya di `.gitignore` sebelum `git add` |
| **Mengunggah branch fitur baru** | `git push -u origin <nama-branch-fitur>` |
| **Membersihkan commit sampah di GitLab** | Centang opsi **Squash commits** saat merge MR |
| **Menghapus branch remote otomatis** | Centang opsi **Delete source branch** saat merge MR |
| **Menutup tiket tugas otomatis** | Tulis `Closes #<nomor-issue>` pada deskripsi Merge Request |
| **Mengelompokkan issue dalam target besar** | Buat **Milestone** dan kaitkan tiket-tiket Issue ke dalamnya |

---

### Uji Pemahaman Mandiri
Ujilah pemahamanmu dengan menjawab pertanyaan evaluasi berikut sebelum melanjutkan ke laboratorium praktik:

1. **Ubahlah pesan commit informal berikut menjadi format Conventional Commit yang baku:**
   - *"udah benerin script backup yang error gara-gara direktori tujuan belum dibuat"*
   - *"menambahkan dashboard grafana baru untuk memantau penggunaan memory pod"*
2. **Mengapa tim SRE berkinerja tinggi lebih memilih Trunk-Based Development dengan branch berumur pendek dibanding Git Flow yang membiarkan branch terpisah selama berminggu-minggu?**
3. **Jika kamu tidak sengaja meng-commit file `.env` yang berisi password root database dan sudah terlanjur kamu push ke remote GitLab publik, jelaskan mengapa sekadar menghapus file tersebut di commit berikutnya tidak menyelesaikan masalah keamanan, dan tindakan darurat apa yang wajib kamu lakukan?**
4. **Apa keuntungan teknis dari mengaktifkan fitur *Squash commits* saat melakukan merge pada Merge Request di GitLab? Bagaimana pengaruhnya terhadap riwayat buku sejarah pada branch `main`?**
5. **Bagaimana mekanisme kata kunci `Closes #15` pada deskripsi Merge Request dapat menghemat waktu operasional seorang SRE?**

---
*Lanjutkan ke laboratorium praktik: [lab/LAB-01-gitlab-repo.md](lab/LAB-01-gitlab-repo.md) untuk mulai membangun repositori portofolio resmi `sre-bootcamp` di GitLab!*