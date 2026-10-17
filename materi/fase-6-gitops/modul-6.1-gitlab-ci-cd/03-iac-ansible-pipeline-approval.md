# 03 — Pipeline IaC dan Ansible dengan Approval Gate: Mengotomasi Pengadaan dan Konfigurasi Tanpa Drift

> **Ringkasan:** Menjalankan `tofu apply` atau `ansible-playbook` langsung dari laptop operator adalah bentuk pelanggaran fatal terhadap prinsip *Infrastructure as Code* (IaC) dan tata kelola SRE. Tindakan lokal tersebut menciptakan ketergantungan pada workstation individu (*snowflake operators*), menghilangkan jejak audit (*broken chain of custody*), dan berisiko memicu tabrakan *state lock*. Solusi produksi standar industri adalah memindahkan seluruh eksekusi ke dalam pipa otomatisasi CI/CD terpusat yang dilengkapi **Gerbang Persetujuan Manusia (*Approval Gate*)**. Dokumen ini membedah orkestrasi pipeline OpenTofu dan Ansible di GitLab CI, pengendalian konkurensi via `resource_group`, penguncian rencana eksekusi biner (*deterministic plan execution*), serta mitigasi insiden *state lock*.

---

## 1. Analogi Pemula: Kontrak Pembangunan Jembatan & Sakelar Kunci Ganda

Untuk memahami mengapa eksekusi infrastruktur di CI/CD mewajibkan pemisahan antara tahap *Plan* dan *Apply*, mari gunakan analogi proyek infrastruktur sipil:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI RENCANA ANGGARAN BIAYA vs PENGECORAN BETON          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TAHAP 1: PENYUSUNAN RENCANA ANGGARAN BIAYA (RAB) -> TOFU PLAN ]          │
│  - Mandor dan Arsitek menghitung:                                           │
│    "Kita butuh 100 tiang pancang baru (+), 2 jembatan lama dibongkar (-),   │
│     dan 5 sambungan aspal diperbaiki (~)."                                  │
│  - Dokumen ini dicetak, diberi nomor segel resmi, dan ditandatangani        │
│    (**tfplan.binary**).                                                     │
│  - PENTING: Mandor TIDAK BOLEH langsung menuang semen saat dokumen dibuat!  │
│                                                                             │
│  [ TAHAP 2: SAKELAR DUA KUNCI GANDA -> 4-EYES APPROVAL GATE ]               │
│  - Pejabat Pengawas Keselamatan (Lead SRE) membuka dokumen RAB bersegel:    │
│    "Apakah ada rumah warga yang tidak sengaja terobohkan? Apakah biaya pas?"│
│  - Jika dokumen aman, Pejabat memutar kunci izin resmi (**Klik Tombol Play**│
│    di antarmuka GitLab CI)!                                                 │
│                                                                             │
│  [ TAHAP 3: PENGECORAN FISIK BERDASARKAN SEGEL -> TOFU APPLY ]              │
│  - Truk molen semen datang dan mengecor tanah HANYA BERDASARKAN DOKUMEN     │
│    BERSEGEL RAB TAHAP 1!                                                    │
│  - Truk molen DILARANG berimprovisasi atau menambah semen di luar dokumen!  │
│                                                                             │
│  BAHAYA FATAL JIKA DIAKUKAN DI LAPTOP LOKAL OPERATOR:                       │
│  Jika operator menjalankan apply langsung dari laptopnya tanpa review RAB,  │
│  ia bisa tidak sengaja merobohkan seluruh klaster server produksi dalam     │
│  waktu 5 detik (**CATASTROPHIC ACCIDENTAL DESTRUCTION**)!                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Alur Pipeline OpenTofu (IaC) Produksi

Di GitLab CI, pipeline OpenTofu dirancang untuk memisahkan secara ketat antara tahap inspeksi (*read-only*) dan tahap mutasi (*write*):

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ALUR PIPELINE OPENTOFU PRODUKSI                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ STAGE: VALIDATE ] (Otomatis pada setiap commit)                          │
│  - `tofu fmt -check` : Memastikan indentasi HCL rapi.                       │
│  - `tofu validate`   : Memeriksa validitas skema provider & variabel.       │
│            │                                                                │
│            ▼ LULUS                                                          │
│                                                                             │
│  [ STAGE: PLAN ] (Menghasilkan Rencana Eksekusi Bersegel)                   │
│  - Inisialisasi backend MinIO S3 terisolasi.                                │
│  - Eksekusi: `tofu plan -out=tfplan.binary`                                 │
│  - Ekstrak ringkasan perubahan yang disanitasi:                             │
│    `tofu show -json tfplan.binary | jq ... > plan-summary.json`             │
│            │                                                                │
│            ▼ Disimpan sebagai Artifact Internal                             │
│                                                                             │
│  [ STAGE: APPROVAL GATE ] (Pemeriksaan Manusia)                             │
│  - Lead SRE membaca selisih perubahan di Merge Request.                     │
│  - Menyetujui eksekusi (Klik Manual Trigger).                               │
│            │                                                                │
│            ▼ DISETUJUI                                                      │
│                                                                             │
│  [ STAGE: APPLY ] (Eksekusi Mutasi Fisik ke Hypervisor)                     │
│  - KUNCI MUTLAK: `tofu apply tfplan.binary`                                 │
│    HANYA mengeksekusi file biner yang telah disetujui di Tahap Plan!        │
│  - Ekstrak metadata inventaris non-secret: `tofu output -json > hosts.json` │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Pengendalian Konkurensi & State Locking (`resource_group`)

Salah satu malapetaka terbesar pada otomatisasi IaC adalah ketika dua pipeline berjalan bersamaan pada repositori yang sama (misal dua engineer me-merge kode di detik yang sama).

Jika kedua runner mengeksekusi `tofu apply` secara paralel, akan terjadi **State Lock Conflict** pada MinIO S3 backend, atau lebih buruk lagi, korupsi berkas state!

GitLab CI menyediakan fitur penyelamat: **`resource_group`**.

```yaml
apply_infrastructure:
  stage: apply
  # Memastikan hanya ada 1 job apply yang berjalan pada satu waktu di seluruh dunia!
  resource_group: "production-opentofu-state"
  script:
    - tofu apply -auto-approve tfplan.binary
```

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 CARA KERJA RESOURCE_GROUP GITLAB CI                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Pipeline #101 (Engineer A) ──► Apply Berjalan (Memegang Kunci Resource)    │
│                                       │                                     │
│  Pipeline #102 (Engineer B) ──► OTOMATIS MENGANTRE (STATUS: WAITING)        │
│                                 Menunggu Pipeline #101 selesai dengan tertib│
│                                 tanpa memicu bentrokan state lock MinIO!    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Implementasi Pipa Lengkap: OpenTofu -> Handoff -> Ansible

Berikut adalah konfigurasi utuh berkas `.gitlab-ci.yml` yang menghubungkan siklus hidup OpenTofu dan Ansible:

```yaml
stages:
  - iac_validate
  - iac_plan
  - iac_apply
  - config_preflight
  - config_apply

variables:
  TF_ROOT: "terraform/environments/production"
  TF_IN_AUTOMATION: "true"
  ANSIBLE_CONFIG: "ansible/ansible.cfg"

# ==============================================================================
# TAHAP 1: OPENTOFU VALIDATE & PLAN
# ==============================================================================
tofu_validate:
  stage: iac_validate
  image: ghcr.io/opentofu/opentofu:1.8.2
  script:
    - cd ${TF_ROOT}
    - tofu init -backend=false
    - tofu fmt -check
    - tofu validate

tofu_plan:
  stage: iac_plan
  image: ghcr.io/opentofu/opentofu:1.8.2
  needs: ["tofu_validate"]
  before_script:
    - cd ${TF_ROOT}
    - tofu init
  script:
    # Simpan file plan biner yang tersegel
    - tofu plan -out=tfplan.binary
    # Buat ringkasan perubahan yang aman dari rahasia untuk dibaca di MR
    - tofu show -json tfplan.binary | jq -r '{actions: [.resource_changes[].change.actions[]]}' > plan-summary.json
  artifacts:
    name: "tofu-plan-${CI_COMMIT_SHORT_SHA}"
    paths:
      - "${TF_ROOT}/tfplan.binary"
      - "${TF_ROOT}/plan-summary.json"
    expire_in: 2 days

# ==============================================================================
# TAHAP 2: OPENTOFU APPLY (MANUAL APPROVAL GATE)
# ==============================================================================
tofu_apply:
  stage: iac_apply
  image: ghcr.io/opentofu/opentofu:1.8.2
  resource_group: "prod-infrastructure"
  needs:
    - job: tofu_plan
      artifacts: true
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual # GERBANG PERSETUJUAN DUA ORANG (4-EYES PRINCIPLE)!
  before_script:
    - cd ${TF_ROOT}
    - tofu init
  script:
    # Wajib mengeksekusi file biner hasil plan yang sudah disetujui!
    - tofu apply tfplan.binary
    # Ekstrak metadata inventaris non-secret untuk diserahkan ke Ansible
    - tofu output -json ansible_inventory_metadata > ${CI_PROJECT_DIR}/ansible/inventories/production/hosts.json
  artifacts:
    paths:
      - "ansible/inventories/production/hosts.json"
    expire_in: 1 day

# ==============================================================================
# TAHAP 3: ANSIBLE READINESS INSPECTION (8 GATES)
# ==============================================================================
ansible_preflight:
  stage: config_preflight
  image: python:3.11-slim
  needs:
    - job: tofu_apply
      artifacts: true
  before_script:
    - pip install ansible-core
    - echo "$ANSIBLE_SSH_KEY" > /tmp/id_ed25519 && chmod 600 /tmp/id_ed25519
  script:
    - cd ansible
    # Konversi hosts.json menjadi inventaris dan jalankan 8 Gerbang Kesiapan
    - ansible-playbook -i inventories/production/hosts.json playbooks/preflight-readiness.yml --private-key=/tmp/id_ed25519
  after_script:
    - rm -f /tmp/id_ed25519

# ==============================================================================
# TAHAP 4: ANSIBLE CONFIGURATION & K3S DEPLOYMENT
# ==============================================================================
ansible_deploy_k3s:
  stage: config_apply
  image: python:3.11-slim
  resource_group: "prod-os-configuration"
  needs:
    - job: ansible_preflight
    - job: tofu_apply
      artifacts: true
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual # Konfirmasi manual sebelum bootstrap k3s dijalankan!
  before_script:
    - pip install ansible-core
    - echo "$ANSIBLE_SSH_KEY" > /tmp/id_ed25519 && chmod 600 /tmp/id_ed25519
    - echo "$ANSIBLE_VAULT_PASSWORD" > /tmp/vault_pass
  script:
    - cd ansible
    - >
      ansible-playbook -i inventories/production/hosts.json playbooks/deploy-k3s-ha.yml
      --private-key=/tmp/id_ed25519
      --vault-password-file=/tmp/vault_pass
  after_script:
    - rm -f /tmp/id_ed25519 /tmp/vault_pass
```

---

## 5. Manajemen Kredensial & Higienitas Rahasia di Pipeline

Otomatisasi IaC dan Ansible membutuhkan hak akses tinggi (hak kelola hypervisor dan akses root SSH). Kebocoran kredensial di pipeline adalah risiko keamanan nomor 1.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ATURAN HIGIENITAS RAHASIA SRE DI PIPELINE                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. DILARANG KERAS MENGUNGGAH `terraform.tfstate` SEBAGAI ARTIFAK PIPELINE! │
│     File state memuat seluruh atribut sistem dalam format plaintext.        │
│     Biarkan state tersimpan aman di dalam bucket MinIO S3 yang terenkripsi! │
│                                                                             │
│  2. HAPUS KUNCI PRIVAT DI `after_script`:                                   │
│     Kunci privat SSH yang disuntikkan ke `/tmp/id_ed25519` WAJIB DIHAPUS   │
│     pada blok `after_script:` untuk menjamin file terhapus meskipun task    │
│     mengalami kegagalan (*fail-safe cleanup*).                              │
│                                                                             │
│  3. GUNAKAN SELALU `tfplan.binary`:                                         │
│     Jangan pernah menjalankan `tofu apply -auto-approve` tanpa rujukan file │
│     plan biner. Menjalankan apply mentah dapat mengeksekusi perubahan liar   │
│     yang terjadi antara waktu peninjauan code dan waktu klik eksekusi!      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA PIPELINE IAC DAN ANSIBLE                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "Error: Error acquiring the state lock" ]                       │
│  Gejala: Job `tofu_plan` atau `tofu_apply` gagal seketika dengan pesan lock.│
│  Akar Masalah:                                                              │
│  - Pipeline sebelumnya dibatalkan paksa saat `tofu apply` sedang menulis    │
│    state ke MinIO S3, meninggalkan file kunci lock yang menggantung.        │
│  Solusi SRE:                                                                │
│  1. Verifikasi bahwa tidak ada operator lain yang sedang menjalankan apply. │
│  2. Jalankan perintah force-unlock dengan Lock ID yang tertera di log:      │
│     tofu force-unlock <LOCK-ID>                                             │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "Plan file was created by a different version of OpenTofu" ]    │
│  Gejala: Job `tofu_apply` menolak mengeksekusi `tfplan.binary`.             │
│  Akar Masalah:                                                              │
│  - Gambar kontainer pada job `tofu_plan` menggunakan versi OpenTofu yang    │
│    berbeda dengan gambar kontainer pada job `tofu_apply`.                   │
│  Solusi SRE:                                                                │
│  Selalu kunci versi image OpenTofu secara presisi pada variabel global:     │
│  image: ghcr.io/opentofu/opentofu:1.8.2 (JANGAN GUNAKAN :latest!).         │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: ANSIBLE PREFLIGHT GAGAL KARENA VM BELUM SELESAI BOOT ]          │
│  Gejala: `tofu_apply` sukses membuat VM, tapi `ansible_preflight` gagal     │
│          dengan error `UNREACHABLE! Connection refused (Port 22)`.          │
│  Akar Masalah:                                                              │
│  - VM baru saja selesai dibuat hypervisor, namun proses booting OS belum    │
│    sempat menyalakan layanan `sshd.service`.                                 │
│  Solusi SRE:                                                                │
│  Tambahkan task penunggu (wait_for) di awal playbook Ansible:               │
│  - wait_for:                                                                │
│      port: 22                                                               │
│      host: "{{ ansible_host }}"                                             │
│      delay: 15                                                              │
│      timeout: 120                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menghasilkan file rencana eksekusi biner terisolasi
tofu plan -out=tfplan.binary

# 2. Menginspeksi isi file plan biner dalam format teks tanpa mengeksekusi
tofu show tfplan.binary

# 3. Menerapkan perubahan HANYA berdasarkan file rencana biner yang disetujui
tofu apply tfplan.binary

# 4. Membuka paksa state lock yang tersangkut pasca kegagalan pipeline
tofu force-unlock <LOCK-ID>

# 5. Mengekstrak metadata inventaris Ansible dalam format JSON
tofu output -json ansible_inventory_metadata > hosts.json
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa dalam pipeline CI/CD produksi, perintah `tofu apply` **WAJIB** mengeksekusi berkas rencana biner (`tofu apply tfplan.binary`) alih-alih menjalankan `tofu apply -auto-approve` secara langsung?
- A. Agar proses apply berjalan tanpa memerlukan koneksi jaringan ke hypervisor.
- B. Menjamin sifat deterministik mutlak (*Plan Immutability*): memastikan bahwa perubahan yang dieksekusi ke infrastruktur fisik identik 100% dengan apa yang telah ditinjau dan disetujui pada tahap review, mencegah eksekusi perubahan tak terduga (*drift*) yang mungkin terjadi di antara waktu review dan waktu apply.
- C. Karena OpenTofu versi 1.8 menolak eksekusi tanpa file biner.
- D. Untuk menghapus seluruh file log di server runner.

*Kunci Jawaban:* **B**. Mengeksekusi file plan biner yang tersegel adalah pilar kepatuhan utama untuk mencegah eksekusi liar di luar rencana yang disetujui.

### Soal 2 (Pilihan Ganda)
Apa fungsi utama dari menyertakan parameter `resource_group: "prod-infrastructure"` pada job deployment OpenTofu di berkas `.gitlab-ci.yml`?
- A. Mengelompokkan seluruh server ke dalam satu grup VLAN.
- B. Berfungsi sebagai pengendali konkurensi terpusat yang menjamin hanya ada tepat 1 job apply yang berjalan dalam satu waktu di seluruh proyek, mencegah terjadinya bentrokan *state lock* atau korupsi basis data state MinIO S3 akibat eksekusi paralel.
- C. Mempercepat kompresi file biner.
- D. Mengubah GitLab Runner menjadi load balancer MetalLB.

*Kunci Jawaban:* **B**. `resource_group` menerapkan antrean serial otomatis pada pekerjaan infrastruktur misi kritis guna mencegah bencana konkurensi state.

### Soal 3 (Pilihan Ganda)
Mengapa berkas state mentah `terraform.tfstate` dilarang keras untuk disimpan sebagai *CI Artifacts* publik yang dapat diunduh di GitLab?
- A. Karena ukuran berkas state melebihi kuota 5 GB.
- B. Karena berkas state memuat seluruh konfigurasi infrastruktur dalam format teks biasa (*plaintext*), termasuk kata sandi awal sistem operasi, kunci privat, dan token akses yang sangat berisiko bocor jika diunduh oleh pihak yang tidak berwenang.
- C. Karena berkas state otomatis rusak jika diunggah ke GitLab.
- D. Agar runner tidak kehabisan memori RAM.

*Kunci Jawaban:* **B**. Berkas state adalah dokumen sangat sensitif yang wajib disimpan secara terisolasi di penyimpanan objek dingin (*MinIO S3 backend*) dengan kontrol hak akses ketat.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Di sebuah private data center, job `tofu_apply` di pipeline GitLab CI tiba-tiba gagal di tengah jalan karena server runner mengalami mati lampu mendadak. Ketika listrik menyala dan tim mencoba menjalankan kembali pipeline, seluruh job OpenTofu melempar error:
`Error: Error acquiring the state lock: ConditionalCheckFailedException: Lock already held`
Jelaskan mengapa mekanisme penguncian (*State Locking*) ini terjadi, dan susun prosedur SRE yang aman untuk menyelesaikan insiden ini tanpa merusak integritas infrastruktur!

*Panduan Solusi:*
1. **Analisis Mekanisme State Locking:**
   Saat `tofu apply` mulai berjalan, ia menuliskan catatan kunci (*distributed lock*) pada storage backend MinIO S3 untuk mencegah proses lain menulis ke state yang sama. Karena proses mati mendadak sebelum sempat menyelesaikan siklus normal, berkas kunci tersebut tidak sempat dihapus (*stale lock*). Sistem secara sengaja menolak eksekusi lanjutan sebagai bentuk perlindungan diri (*Fail-Closed*) agar state tidak korup.
2. **SOP Resolusi Aman SRE:**
   - **Verifikasi Ketiadaan Proses Aktif:**
     Pastikan tidak ada runner lain atau engineer lain yang sedang mencoba menjalankan apply.
   - **Dapatkan Lock ID:**
     Buka log job yang gagal di GitLab CI dan salin nilai unik `Lock Info: ID: 8f9b2c17-43a1-9e24-f706-e25776d6ec14`.
   - **Lepaskan Kunci Secara Sadar (Force Unlock):**
     Dari terminal terproteksi dengan hak akses backend, jalankan:
     ```bash
     tofu force-unlock 8f9b2c17-43a1-9e24-f706-e25776d6ec14
     ```
   - **Sinkronisasi Ulang (Refresh-Only):**
     Sebelum melakukan apply baru, jalankan `tofu plan -refresh-only` untuk mendeteksi apakah ada VM yang sempat terbuat sebelum runner mati lampu, sehingga rekaman state kembali selaras dengan keadaan fisik di hypervisor.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Fase 3: Modul 3.3: 03 — OpenTofu, Ansible, dan k3s Handoff](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/03-opentofu-ansible-k3s-handoff.md) — Kontrak alih tugas infrastruktur ke konfigurasi.
  - [Modul 6.1: 02 — GitLab Runner, Artifacts, Cache, dan Environment](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.1-gitlab-ci-cd/02-runner-artifacts-cache-environment.md) — Arsitektur agen pelaksana runner.
- **Tahap Berikutnya:**
  - [Modul 6.2 — GitOps dengan ArgoCD](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/) — Mengotomasi rekonsiliasi manifes aplikasi Kubernetes secara pull-based.
  - [LAB-02 — Pipeline IaC dan Ansible Approval](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.1-gitlab-ci-cd/lab/LAB-02-iac-plan-approval-lab-run.md) — Praktik simulasi eksekusi plan, approval, dan apply di terminal.
