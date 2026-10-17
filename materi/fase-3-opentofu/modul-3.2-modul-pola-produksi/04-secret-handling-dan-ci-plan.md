# 04 — Pengelolaan Rahasia (Secret Handling) & Pipeline CI/CD Plan

> **Ringkasan:** Salah satu celah keamanan terbesar dalam implementasi Infrastructure as Code (IaC) adalah kebocoran kredensial rahasia (*secret leakage*) ke dalam repositori Git publik, log pipeline CI/CD, atau file state yang tidak terenkripsi. Dokumen ini membedah peta 6 titik paparan rahasia (*The 6 Secret Exposure Points*), merancang integrasi variabel lingkungan aman, memahami arsitektur Vault dan SOPS, mendesain batas penyerahan (*handoff*) metadata non-rahasia dari OpenTofu ke Ansible, hingga membangun alur pipeline CI/CD bertingkat yang memiliki gerbang persetujuan (*Approval Gate*) ketat.

---

## 1. Analogi Pemula: Kurir Brankas Khusus vs Menempelkan Kunci di Papan Pengumuman

Bagi pemula, menyisipkan password langsung di dalam file konfigurasi terasa sangat praktis:
> *"Kenapa saya tidak menuliskan `password = 'SuperSecret123'` langsung di file `main.tf`? Kan repositori Git saya bersifat privat, jadi aman-aman saja!"*

Mari kita gunakan analogi keamanan fisik:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI KUNCI BRANKAS VS PAPAN PENGUMUMAN LOBI              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ ANTI-PATTERN: MENEMPELKAN KUNCI DI PAPAN PENGUMUMAN LOBI (HARDCODED) ]   │
│  Anda menulis nomor kombinasi brankas uang kantor di atas kertas memo, lalu │
│  menempelkannya di papan pengumuman lobi kantor (Meng-commit ke Git).       │
│  - Masalah: Meskipun satpam gedung berkata "Gedung ini privat hanya untuk   │
│    karyawan", siapa pun yang berjalan lewat lobi (staf magang, vendor AC,   │
│    tamu luar) bisa memotret nomor kombinasi tersebut.                       │
│  - Sekali kertas memo itu terfoto (masuk ke Git Commit History), nomor      │
│    tersebut akan abadi di internet selamanya meskipun memonya Anda copot!   │
│                                                                             │
│  [ PENDEKATAN SRE PROFESIONAL: PETUGAS KURIR BRANKAS DENGAN KUNCI SEMENTARA]│
│  Nomor kombinasi brankas disimpan di brankas pusat terenkripsi (**Vault**). │
│  - Ketika pintu brankas harus dibuka, kurir khusus beridentitas resmi       │
│    (**CI/CD Runner IAM Role**) meminta kunci sementara yang hanya berlaku   │
│    selama 10 menit (**Short-Lived Ephemeral Token**).                       │
│  - Kurir membuka pintu brankas di ruang tertutup tanpa ada yang melihat,   │
│    lalu kunci tersebut otomatis kedaluwarsa dan musnah seketika!            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Peta 6 Titik Paparan Rahasia (The 6 Secret Exposure Points)

Banyak engineer mengira jika kode HCL mereka bersih dari teks password, maka rahasia mereka sudah 100% aman. Faktanya, di dalam siklus operasional OpenTofu, ada **enam titik kebocoran rahasia** yang wajib diawasi secara ketat oleh tim SRE:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ENAM TITIK RAWAN KEBOCORAN RAHASIA                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. KODE HCL DI REPOSITORI GIT ──► Password tertulis polos di *.tf / *.tfvars│
│  2. RIWAYAT TERMINAL SHELL     ──► Flag -var="pass=xxx" terekam di history   │
│  3. STATE FILE (*.tfstate)     ──► TERSIMPAN TELANJANG (PLAINTEXT DECODED!)  │
│  4. BINARY PLAN ARTIFACT       ──► File tfplan menyimpan isi selisih secret  │
│  5. OUTPUT LOG CI/CD RUNNER    ──► Nilai variabel tercetak di layar console  │
│  6. BACKUP & CRASH LOGS        ──► File dump memori saat OpenTofu crash      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!CAUTION]
> **Hukum Mutlak Keamanan State:**
> **State file (`terraform.tfstate`) MENYIMPAN SELURUH NILAI SENSITIF DALAM BENTUK MENTAH (PLAINTEXT)!**
> Meskipun Anda menyetel `sensitive = true` pada variabel atau output, itu **HANYA MENYEMBUNYIKAN TAMPILAN DI TERMINAL CLI**, bukan mengenkripsi isi file state. Siapa pun yang memiliki hak baca ke bucket S3/MinIO state Anda dapat membaca seluruh password database dan private key perusahaan Anda!

---

## 3. Tiga Arsitektur Pengelolaan Rahasia Standar Industri

Bagaimana cara memasukkan nilai rahasia (seperti password root database atau token API) ke dalam OpenTofu tanpa pernah menuliskannya di kode HCL?

### Metode 1: Environment Variables CI/CD (Protected & Masked)
Di pipeline GitLab CI/CD atau GitHub Actions, rahasia disimpan di menu *Settings ──► CI/CD ──► Variables* sebagai variabel yang ditandai **Protected** dan **Masked**.

Runner CI/CD menyuntikkan nilai tersebut saat runtime menggunakan prefix khusus **`TF_VAR_<nama_variabel>`**:

```bash
# Di dalam skrip runner CI/CD (Bukan di laptop lokal!):
export TF_VAR_database_password="$DB_PROD_SECRET_PASSWORD"

# OpenTofu otomatis membaca TF_VAR_database_password ke variable "database_password"
tofu plan -out=prod.tfplan

# Segera bersihkan variabel dari lingkungan shell setelah selesai
unset TF_VAR_database_password
```

---

### Metode 2: HashiCorp Vault / Secret Manager (Dynamic Ephemeral Secrets)
Untuk lingkungan enterprise berskala besar, password tidak dibuat statis oleh manusia. OpenTofu dapat berintegrasi dengan **HashiCorp Vault** untuk meminta kredensial sementara (*just-in-time dynamic credentials*):

```hcl
# Membaca password dinamis dari Vault yang hanya berlaku selama 1 jam (TTL = 1h)
data "vault_generic_secret" "db_credentials" {
  path = "secret/data/production/database"
}

resource "docker_container" "db" {
  name  = "production-mysql"
  image = "mysql:8.0"

  env = [
    # Nilai diambil langsung dari memori Vault, tanpa pernah ada di file Git!
    "MYSQL_ROOT_PASSWORD=${data.vault_generic_secret.db_credentials.data["root_password"]}"
  ]
}
```

---

### Metode 3: SOPS (Secrets OPerationS) + age / PGP
Jika perusahaan Anda belum memiliki server Vault terpusat, standar industri menggunakan tool **SOPS** buatan Mozilla yang dipadukan dengan enkripsi kunci **age**:

```text
  File Asli (secrets.dec.yaml) ──► Di-encrypt oleh SOPS ──► File Git (secrets.enc.yaml)
  (Password: Rahasia123)                                     (Password: ENC[AES256_GCM,data:...])
```

- File `secrets.enc.yaml` **sangat aman di-commit ke repositori Git publik** karena seluruh isinya terenkripsi kriptografis tingkat militer.
- Di dalam pipeline CI/CD, runner yang memegang *Private Key age* mendekripsi file tersebut ke direktori memori sementara (*ramdisk `/dev/shm`*) untuk dikonsumsi oleh OpenTofu saat eksekusi.

---

## 4. Pemisahan Peran: Handoff Metadata OpenTofu ke Ansible

Di dalam arsitektur operasional Bootcamp SRE 2026, kita menerapkan pembagian batas wewenang yang tegas antara alat *Provisioning* (OpenTofu) dengan alat *Configuration Management* (Ansible):

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                  BATAS WEWENANG: OPENTOFU VS ANSIBLE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TUGAS OPENTOFU (INFRASTRUCTURE PROVISIONING) ]                           │
│  ├── 1. Membuat VM di Proxmox atau kontainer di Docker.                     │
│  ├── 2. Mengalokasikan IP Address, Network Bridge, dan Virtual Disk.        │
│  └── 3. MENGEKSPOS METADATA NON-SECRET KE ANSIBLE (Hostname, IP, Role).    │
│                                                                             │
│       │ Handoff Metadata Non-Secret (inventory.json)                        │
│       ▼ OpenTofu DILARANG KERAS mengirim password SSH / Private Key!        │
│                                                                             │
│  [ TUGAS ANSIBLE (HOST CONFIGURATION & HARDENING) ]                         │
│  ├── 1. Mengambil kredensial SSH dari Ansible Vault / Runner Key terpisah.  │
│  ├── 2. Melakukan OS Hardening, konfigurasi UFW firewall, dan sysctl tuning.│
│  └── 3. Menginstal k3s cluster, etcd initialization, dan systemd service.   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Format Output yang Layak Diserahkan ke Ansible (`outputs.tf`):
Hanya keluarkan informasi topologi jaringan dan peran node, **tanpa menyertakan rahasia apa pun**:

```hcl
output "ansible_inventory_metadata" {
  description = "Metadata topologi node untuk dikonsumsi oleh dynamic inventory Ansible."
  value = {
    for name, instance in module.k3s_nodes : name => {
      ansible_host = instance.ip_address
      node_role    = instance.role          # control-plane atau worker
      environment  = var.environment
    }
  }
}
```

Ansible akan membaca metadata IP di atas, lalu melakukan autentikasi SSH menggunakan kunci privat yang tersimpan aman di dalam *Ansible Vault* atau *SSH Agent* lokal runner, bukan dari OpenTofu!

---

## 5. Perancangan Pipeline CI/CD yang Aman (The Secure IaC Pipeline)

Di lingkungan produksi enterprise, seorang engineer **dilarang menjalankan `tofu apply` langsung dari laptop pribadinya**. Seluruh perubahan infrastruktur wajib dieksekusi melalui pipeline CI/CD terpusat yang memiliki tahapan pengawasan ketat:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR PIPELINE GITOPS CI/CD ENTERPRISE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ FASE 1: PULL / MERGE REQUEST (READ-ONLY GATE) ]                          │
│  Pipeline berjalan otomatis pada setiap commit PR baru:                     │
│  1. $ tofu fmt -check       (Pastikan format rapi)                          │
│  2. $ tofu validate         (Pastikan sintaks benar)                        │
│  3. $ tofu plan -out=tfplan (Hitung selisih perubahan)                      │
│  4. Hasil plan di-render ke komentar PR untuk ditinjau bersama rekan tim!   │
│  *KUNCI KEAMANAN:* Runner di Fase 1 HANYA MEMILIKI AKSES READ-ONLY ke cloud!│
│                                                                             │
│       │ PR di-review & disetujui oleh Senior SRE / Lead Engineer            │
│       ▼ Merge ke branch 'main'                                              │
│                                                                             │
│  [ FASE 2: MANUAL APPROVAL GATE (REMBOK PENGAMAN AKHIR) ]                   │
│  Pipeline berhenti sejenak dan menunggu tombol konfirmasi fisik manusia.   │
│  Hanya engineer berwenang (*Maintainer*) yang memiliki hak klik "Approve".  │
│                                                                             │
│       │ Klik Tombol "Approve Production Apply"                              │
│       ▼                                                                     │
│                                                                             │
│  [ FASE 3: PRODUCTION APPLY (RESTRICTED RUNNER) ]                            │
│  Dieksekusi oleh Runner khusus yang memiliki hak tulis (*Write IAM Role*):  │
│  1. $ tofu apply tfplan     (Menerapkan TEPAT plan yang sudah di-review!)   │
│  2. Validasi status kesehatan layanan (*Smoke Test*)                        │
│  3. File binary tfplan langsung dimusnahkan dari cache (*Artifact Purge*).  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Contoh Skrip Pipeline Guardrail Non-Secret (`deploy-guardrail.sh`)

Untuk mencegah terjadinya salah tembak lingkungan di dalam pipeline otomatisasi, kita memasang skrip pelindung (*guardrail*) yang akan membatalkan proses seketika jika parameter lingkungan tidak cocok:

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Validasi Keberadaan Variabel Lingkungan Wajib
TARGET_ENV="${ENVIRONMENT:?ERROR: Variabel ENVIRONMENT wajib didefinisikan!}"
ALLOWED_ENVS=("dev" "staging" "prod")

# 2. Periksa apakah target environment terdaftar di whitelist resmi
if [[ ! " ${ALLOWED_ENVS[*]} " =~ " ${TARGET_ENV} " ]]; then
  echo "[FATAL] Lingkungan '${TARGET_ENV}' tidak dikenal! Eksekusi dibatalkan demi keamanan."
  exit 1
fi

echo "[INFO] Memulai validasi dan perencanaan untuk lingkungan: ${TARGET_ENV}"
cd "environments/${TARGET_ENV}"

# 3. Validasi Statis Kode HCL
tofu fmt -check
tofu init -input=false
tofu validate

# 4. Generate Rencana Eksekusi Binary Terkunci
tofu plan -input=false -out="${TARGET_ENV}.tfplan"

echo "[SUCCESS] Plan untuk ${TARGET_ENV} berhasil dibuat dan dikunci di ${TARGET_ENV}.tfplan."
```

---

## 7. Prosedur Insiden: Apa yang Wajib Dilakukan Jika Secret Bocor?

Jika secara tidak sengaja Anda melihat ada password, API key, atau private key yang ter-commit ke dalam repositori Git atau tercetak di log terminal:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 SOP DARURAT KEBOCORAN RAHASIA (SECRET LEAK)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. ANGGAP KUNCI TELAH DIRETAS SEKETIKA ITU JUGA!                            │
│    Jangan buang waktu dengan berpikir "Mungkin belum ada yang membaca".     │
│                                                                             │
│ 2. REVOKE & ROTATE LANGSUNG PADA SUMBERNYA:                                 │
│    Segera buka dashboard database atau cloud provider, matikan password     │
│    yang bocor, dan terbitkan password baru yang berbeda!                    │
│                                                                             │
│ 3. PERBAIKI STATE & PIPELINE:                                               │
│    Perbarui nilai variabel rahasia di Secret Manager / Protected Variables. │
│                                                                             │
│ 4. BERSIHKAN RIWAYAT GIT (GIT PURGE):                                       │
│    Gunakan tool 'git-filter-repo' atau 'BFG Repo-Cleaner' untuk menghapus   │
│    commit yang mengandung rahasia dari riwayat Git secara permanen.         │
│                                                                             │
│ 5. TULIS BLAMELESS POSTMORTEM:                                              │
│    Analisis mengapa linter pra-commit (*gitleaks*) gagal menangkap insiden. │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Gejala Masalah | Titik Investigasi | Penyebab Akar (*Root Cause*) | Solusi Pemulihan SRE |
|---|---|---|---|
| `Kredensial tercetak polos di log CI/CD console` | Output job terminal CI/CD | Variabel input tidak ditandai `sensitive = true`, atau skrip shell menggunakan `set -x` (*xtrace*) yang mencetak seluruh baris perintah. | Tambahkan `sensitive = true` pada variabel HCL dan matikan opsi `set -x` pada skrip runner shell. |
| `Error: Failed to read tfplan file` | `tofu apply tfplan` | File plan binary dibuat dari versi OpenTofu atau arsitektur OS yang berbeda dari runner eksekutor. | Pastikan container image runner CI pada tahap *plan* dan tahap *apply* identik 100%. |
| `Pipeline apply gagal dengan 403 Forbidden` | `tofu apply` | Runner CI/CD tahap apply tidak memiliki hak akses IAM / token yang cukup ke target API cloud. | Periksa izin Service Account / IAM Role yang dikaitkan ke runner khusus deployment. |
| `Sensitive output bernilai <sensitive>` | `tofu output` | Nilai output memang ditandai `sensitive = true` untuk melindungi dari rekaman mata. | Ini adalah perilaku normal. Jika butuh membacanya secara terisolasi di mesin aman, gunakan `tofu output -json`. |

---

## 9. Ringkasan Praktis & Cheatsheet Keamanan Rahasia

```bash
# 1. Memeriksa keberadaan secret yang tidak sengaja ter-commit sebelum push Git
git diff --staged | grep -iE 'password|secret|token|private_key'

# 2. Mengeksekusi plan tanpa interaktivitas manusia di pipeline CI/CD
tofu plan -input=false -out=prod.tfplan

# 3. Menerapkan plan yang telah disetujui tanpa konfirmasi ulang prompt
tofu apply -input=false prod.tfplan

# 4. Membaca metadata inventory non-secret untuk diserahkan ke Ansible
tofu output -json ansible_inventory_metadata > inventory.json
```

---

## 10. Cek Pemahaman Mandiri

1. Mengapa menambahkan atribut `sensitive = true` pada sebuah variabel HCL tidak otomatis melindungi kerahasiaan password tersebut jika file `terraform.tfstate` berhasil diunduh oleh peretas?
2. Sebutkan minimal 4 dari 6 titik paparan rahasia (*Secret Exposure Points*) di mana sebuah kredensial dapat bocor di luar kode HCL Git!
3. Jelaskan batasan pembagian tugas (*handoff boundary*) antara OpenTofu dengan Ansible! Mengapa OpenTofu dilarang keras menjadi perantara pengiriman password SSH root ke Ansible?
4. Apa fungsi dari pemisahan tahap *Pull Request Plan (Read-Only)* dengan tahap *Main Branch Apply (Restricted Identity)* di dalam perancangan pipeline CI/CD enterprise?
5. Jika sebuah token API produksi secara tidak sengaja ter-commit ke dalam repositori Git publik, sebutkan 3 langkah darurat pertama yang wajib dieksekusi oleh seorang SRE!

---

*Selamat! Anda telah menguasai seluruh kurikulum pola produksi OpenTofu.* Lanjutkan ke [LAB-01 — Modul Reusable Web Server](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.2-modul-pola-produksi/lab/LAB-01-modul-reusable-web-server.md) dan [LAB-02 — Environment Promotion dan Secret Safety](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.2-modul-pola-produksi/lab/LAB-02-environment-promotion-dan-secret-safety.md) untuk mempraktikkan seluruh teknik arsitektur modular ini secara langsung di lingkungan lab Anda!
