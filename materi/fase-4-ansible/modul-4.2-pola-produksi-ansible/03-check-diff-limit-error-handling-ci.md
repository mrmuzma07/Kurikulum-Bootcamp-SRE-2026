# 03 — Check, Diff, Limit, Error Handling, dan CI: Membangun Guardrail Otomatisasi Kelas Produksi

> **Ringkasan:** Dalam skala data center dan klaster Kubernetes bare-metal, satu kesalahan kecil pada skrip otomatisasi yang dijalankan dengan hak akses `root` dapat melumpuhkan seluruh infrastruktur perusahaan dalam hitungan detik. Di sinilah prinsip keandalan *Site Reliability Engineering* (SRE) diuji: kita tidak mengandalkan harapan bahwa "kode tidak akan pernah salah", melainkan membangun **Guardrail** (sistem pengaman berlapis). Dokumen ini membedah teknik pengendalian radius dampak (*blast radius control*), arsitektur penanganan pengecualian (*block/rescue/always*), validasi pertahanan (*defensive assertion*), serta integrasi pipa otomatisasi CI/CD berstandar industri dengan gerbang persetujuan manusia (*human approval gate*).

---

## 1. Analogi Pemula: Rem Darurat F1 & Sekering Listrik Otomatis (Circuit Breaker)

Mengapa SRE sangat terobsesi dengan *guardrails* saat menjalankan playbook?

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI OTOMASI TANPA PENGAMAN vs GUARDRAIL SRE             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ OTOMASI AMATIR: MOBIL BALAP TANPA SISTEM PENGEREMAN ]                    │
│  - Seorang engineer menulis playbook dan menjalankannya ke 100 server:      │
│    `ansible-playbook -i production site.yml`                                │
│  - Masalah: Terdapat satu kesalahan ketik pada task penghapusan direktori    │
│    temporary (`rm -rf / tmp/*` yang terpisah spasi sehingga menghapus `/`). │
│  - Akibat: Dalam 5 detik, 100 server produksi terhapus bersih secara        │
│    serentak! Perusahaan mengalami *catastrophic outage*.                    │
│                                                                             │
│  [ OTOMASI SRE: SISTEM KELISTRIKAN BERSEKERING & SEKAT PINTU KAPAL ]       │
│  - SRE memasang sekering otomatis (*Circuit Breaker*):                     │
│    1. `--limit canary-01`: Uji coba perubahan pada 1 server kecil dahulu.   │
│    2. `serial: [1, 5, 20%]`: Terapkan bertahap gelombang demi gelombang.   │
│    3. `any_errors_fatal: true`: Jika ada 1 server yang konslet, sekering   │
│       langsung jeglek seketika! 99 server lainnya DISELAMATKAN dari bahaya! │
│    4. `block / rescue / always`: Tim pemadam kebakaran otomatis menyerap    │
│       dampak, mencatat bukti log forensik, dan membunyikan alarm ke Slack.  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Pengendalian Radius Dampak (Blast Radius Control)

Di lingkungan produksi yang melayani ribuan transaksi per detik, Anda tidak boleh memperbarui semua server secara bersamaan. Ansible menyediakan mekanisme penjadwalan rolling yang sangat presisi:

### 2.1 Pola Rolling Deployment Progresif (`serial`)
Parameter `serial` menentukan berapa banyak server yang diproses dalam satu gelombang (*batch*):

```yaml
- name: Pembaruan kernel dan patching OS klaster k3s
  hosts: k3s_cluster
  become: true
  # Gelombang 1: 1 server (Canary Test)
  # Gelombang 2: 2 server (Early Adopters)
  # Gelombang 3: 20% dari sisa server (Rolling scale-out)
  serial:
    - 1
    - 2
    - "20%"
  
  # SEKERING OTOMATIS: Batalkan seluruh antrean jika ada 1 server saja yang gagal!
  any_errors_fatal: true
  max_fail_percentage: 0 # Toleransi kegagalan = 0% (Strict Zero Tolerance)
```

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR EKSEKUSI PROGRESSIVE CANARY ROLLOUT                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ GELOMBANG 1: 1 HOST (CANARY) ] ─────────► [ SUKSES? ]                    │
│                                                   │ YA                      │
│                                                   ▼                         │
│  [ GELOMBANG 2: 2 HOSTS ] ─────────────────► [ SUKSES? ]                    │
│                                                   │ YA                      │
│                                                   ▼                         │
│  [ GELOMBANG 3: 20% SISA HOST ] ───────────► [ SUKSES? ] ──► [ SELESAI ]    │
│                                                   │                         │
│                                                   ▼ TIDAK (Ada 1 Gagal!)    │
│                                            [ JEGLEK! (ABORT) ]              │
│                                            Sisa 80% Server Tetap Sehat!     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Penanganan Pengecualian Tangguh: `block`, `rescue`, dan `always`

Pola `block/rescue/always` pada Ansible setara dengan struktur `try/catch/finally` pada bahasa pemrograman modern seperti Python atau Go.

```yaml
- name: Modifikasi parameter kernel sysctl dengan jaminan pemulihan
  hosts: k3s_servers
  become: true
  tasks:
    - name: Blok eksekusi utama dan penanganan insiden
      block:
        - name: 1. Backup konfigurasi lama sebelum modifikasi
          ansible.builtin.copy:
            src: /etc/sysctl.d/99-kubernetes.conf
            dest: /etc/sysctl.d/99-kubernetes.conf.bak
            remote_src: true

        - name: 2. Terapkan konfigurasi sysctl baru
          ansible.posix.sysctl:
            name: vm.max_map_count
            value: "262144"
            state: present
            sysctl_file: /etc/sysctl.d/99-kubernetes.conf
            reload: true

      rescue:
        - name: PERINGATAN DARURAT: Kembalikan file backup (Rollback)
          ansible.builtin.copy:
            src: /etc/sysctl.d/99-kubernetes.conf.bak
            dest: /etc/sysctl.d/99-kubernetes.conf
            remote_src: true

        - name: Muat ulang kernel sysctl darurat
          ansible.builtin.command: sysctl --system

        - name: Kirim notifikasi kegagalan ke sistem pemantauan SRE
          ansible.builtin.debug:
            msg: "[ALERT] Perubahan sysctl pada {{ inventory_hostname }} gagal dan berhasil di-rollback!"

        - name: Gagalkan task secara eksplisit agar pipeline CI tahu ada insiden!
          ansible.builtin.fail:
            msg: "Playbook dihentikan di host {{ inventory_hostname }} karena kegagalan sysctl."

      always:
        - name: Catat jejak audit non-sensitif ke berkas log lokal
          ansible.builtin.lineinfile:
            path: /var/log/sre-audit.log
            create: true
            line: "[{{ ansible_date_time.iso8601 }}] Operator: SRE-Automator, Task: sysctl-change, Host: {{ inventory_hostname }}"
            mode: "0640"
```

> [!IMPORTANT]
> **Anti-Pattern SRE: Bahaya `ignore_errors: true`:**
> Jangan pernah menggunakan `ignore_errors: true` pada task infrastruktur kritis sebagai jalan pintas! Menambahkan `ignore_errors` sama saja dengan mematikan lampu indikator bahaya di dashboard mobil. Gunakan selalu `block/rescue` dengan langkah mitigasi yang jelas dan akhiri blok rescue dengan modul `ansible.builtin.fail` agar status kegagalan tetap tercatat di sistem pemantauan.

---

## 4. Validasi Pertahanan (*Defensive Assertions*)

Sebelum melakukan perubahan destruktif (seperti formatting disk, menginstall ulang k3s, atau merombak interface jaringan), gunakan modul `ansible.builtin.assert` untuk menguji asumsi awal:

```yaml
- name: Gerbang Kesiapan Pra-Penerapan (Preflight Readiness Gate)
  hosts: k3s_cluster
  gather_facts: true
  tasks:
    - name: Pastikan seluruh syarat teknis terpenuhi sebelum memulai mutasi
      ansible.builtin.assert:
        that:
          # 1. Pastikan environment yang dieksekusi valid
          - target_environment in ['lab', 'staging', 'production']
          
          # 2. Pastikan swap memory benar-benar 0 bytes (Kewajiban Kubelet)
          - ansible_facts.swaptotal_mb == 0
          
          # 3. Pastikan partisi /var memiliki ruang sisa minimal 20GB
          - (ansible_facts.mounts | selectattr('mount', 'equalto', '/') | map(attribute='size_available') | first) > (20 * 1024 * 1024 * 1024)
          
          # 4. Pastikan arsitektur CPU didukung oleh template k3s
          - ansible_facts.architecture in ['aarch64', 'x86_64']
        fail_msg: "GERBANG KESIAPAN GAGAL: Host {{ inventory_hostname }} tidak memenuhi syarat teknis dasar!"
        success_msg: "GERBANG KESIAPAN LULUS: Host {{ inventory_hostname }} siap dikonfigurasi."
```

---

## 5. Arsitektur Pipeline CI/CD Ansible Produksi

Di perusahaan teknologi terkemuka, Ansible dijalankan melalui pipeline CI/CD otomatis (misalnya **GitLab CI**). Pipeline dirancang dengan prinsip pertahanan berlapis:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ARSITEKTUR PIPELINE CI/CD ANSIBLE PRODUKSI                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TAHAP 1: STATIC CODE QUALITY & LINTING ] (Otomatis pada setiap commit)   │
│  - yamllint: Memeriksa spasi, indentasi, dan sintaksis YAML.                │
│  - ansible-lint: Menegakkan standar industri, FQCN, dan keamanan.           │
│                                     │                                       │
│                                     ▼ LULUS                                 │
│                                                                             │
│  [ TAHAP 2: OFFLINE SYNTAX CHECK & INVENTORY GRAPH ]                        │
│  - `ansible-playbook --syntax-check playbooks/site.yml`                     │
│  - `ansible-inventory -i inventories/production/hosts.yml --graph`          │
│                                     │                                       │
│                                     ▼ LULUS                                 │
│                                                                             │
│  [ TAHAP 3: STAGING DRY-RUN & DIFF AUDIT ]                                  │
│  - `ansible-playbook -i inventories/staging/hosts.yml site.yml --check --diff`│
│  - Mengunggah berkas artefak selisih konfigurasi untuk ditinjau.            │
│                                     │                                       │
│                                     ▼ LULUS                                 │
│                                                                             │
│  [ TAHAP 4: GERBANG PERSETUJUAN DUA ORANG (4-EYES HUMAN APPROVAL) ]         │
│  - Tombol manual di UI GitLab CI: "Approve Production Deployment".          │
│  - Wajib disetujui oleh minimal 1 orang Senior/Lead SRE.                    │
│                                     │                                       │
│                                     ▼ DISETUJUI                             │
│                                                                             │
│  [ TAHAP 5: PRODUCTION PROGRESSIVE ROLLOUT ]                                │
│  - Pipeline mengeksekusi ke 1 node canary terlebih dahulu:                  │
│    `ansible-playbook -i inventories/production/ site.yml --limit canary`    │
│  - Verifikasi kesehatan via API probe.                                      │
│  - Lanjutkan ke sisa cluster secara rolling (`serial`).                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Cuplikan Berkas Pipeline GitLab CI (`.gitlab-ci.yml`)

```yaml
stages:
  - lint
  - syntax
  - staging_dry_run
  - production_deploy

variables:
  ANSIBLE_CONFIG: "ansible.cfg"

lint_ansible:
  stage: lint
  image: cytopia/ansible-lint:latest
  script:
    - yamllint -c .yamllint .
    - ansible-lint playbooks/

syntax_check:
  stage: syntax
  image: python:3.11-slim
  before_script:
    - pip install ansible-core
    - ansible-galaxy collection install -r collections/requirements.yml
  script:
    - ansible-playbook playbooks/site.yml --syntax-check

staging_simulation:
  stage: staging_dry_run
  image: python:3.11-slim
  script:
    - echo "$STAGING_VAULT_PASS" > /tmp/.vault_pass
    - ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --check --diff --vault-password-file /tmp/.vault_pass
  after_script:
    - rm -f /tmp/.vault_pass

deploy_to_production:
  stage: production_deploy
  image: python:3.11-slim
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual # Mewajibkan klik tombol persetujuan manusia!
  script:
    - echo "$PROD_VAULT_PASS" > /tmp/.vault_pass
    - ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml --vault-password-file /tmp/.vault_pass
  after_script:
    - rm -f /tmp/.vault_pass
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA GUARDRAIL & CI/CD                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: CHECK MODE MELEMPAR FALSE POSITIVE ERROR ]                      │
│  Gejala: Playbook gagal di tahap `--check`, padahal eksekusi normal berhasil│
│  Akar Masalah:                                                              │
│  - Task A membuat direktori `/opt/app`. Task B memasang konfigurasi di      │
│    dalam direktori tersebut. Saat mode `--check`, direktori `/opt/app`     │
│    TIDAK BENAR-BENAR DIBUAT di disk. Akibatnya, Task B error karena path    │
│    belum ada!                                                               │
│  Solusi SRE:                                                                │
│  Pada Task A, tambahkan instruksi agar tetap dieksekusi atau dilewati:      │
│  check_mode: false # Paksa eksekusi pembuatan direktori meskipun dry-run   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: PIPELINE TERHENTI KARENA PROMPT SSH HOST KEY ]                 │
│  Gejala: Job CI/CD menggantung (*hung*) sampai mencapai timeout 60 menit.   │
│  Akar Masalah:                                                              │
│  - Runner CI/CD mencoba SSH ke server baru yang belum ada di `known_hosts`. │
│    SSH mengeluarkan prompt: "Are you sure you want to continue (yes/no)?",  │
│    namun karena tidak ada manusia di CI, runner terdiam selamanya!          │
│  Solusi SRE:                                                                │
│  Di dalam file `ansible.cfg` khusus runner atau via environment variable:  │
│  export ANSIBLE_HOST_KEY_CHECKING=False                                     │
│  Atau masukkan host keys resmi melalui task pre-flight SSH keyscan.         │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: PARTIAL STATE MUTATION SAAT GAGAL ]                             │
│  Gejala: Dari 3 server, server 1 dan 2 selesai di-update, server 3 gagal.   │
│  Akar Masalah:                                                              │
│  - Playbook berhenti di tengah jalan, meninggalkan klaster dalam status     │
│    setengah baru setengah lama (*split configuration state*).               │
│  Solusi SRE:                                                                │
│  1. JANGAN ULANGI SELURUH PLAYBOOK KE SEMUA SERVER!                         │
│  2. Selidiki akar masalah pada server 3 menggunakan log audit.              │
│  3. Eksekusi perbaikan TERISOLASI hanya ke server 3 menggunakan limit:      │
│     ansible-playbook playbooks/site.yml --limit sre-k3s-cp-03               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menjalankan linter standar industri pada seluruh direktori proyek
ansible-lint

# 2. Uji syntax playbook secara cepat tanpa memicu koneksi jaringan
ansible-playbook playbooks/site.yml --syntax-check

# 3. Dry-run dan Diff audit pada satu node Canary terpilih
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml \
  --check --diff --limit sre-k3s-cp-01

# 4. Melompati task tertentu saat pengujian darurat (Bypass Tags)
ansible-playbook playbooks/site.yml --skip-tags "reboot,packages"

# 5. Melanjutkan eksekusi dari task yang gagal tanpa mengulang dari awal
ansible-playbook playbooks/site.yml --start-at-task="Inisialisasi k3s cluster"

# 6. Menjalankan playbook dengan konkurensi terkontrol langsung dari CLI
ansible-playbook playbooks/site.yml -f 3
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Dalam konfigurasi rolling deployment Ansible, apa fungsi dari parameter `any_errors_fatal: true`?
- A. Mengabaikan semua error yang terjadi agar playbook selesai secepatnya.
- B. Berfungsi sebagai *Circuit Breaker*: jika terjadi satu kegagalan pada salah satu host di gelombang (*batch*) yang aktif, Ansible akan langsung menghentikan eksekusi di seluruh host lain yang tersisa, mencegah kerusakan massal.
- C. Otomatis menghapus instalasi operating system pada host yang rusak.
- D. Mematikan seluruh firewall di server target.

*Kunci Jawaban:* **B**. `any_errors_fatal` adalah pelindung sistemik yang membatalkan eksekusi lanjutan jika ada kegagalan fatal pada salah satu node dalam batch.

### Soal 2 (Pilihan Ganda)
Mengapa penggunaan `ignore_errors: true` pada task konfigurasi infrastruktur misi kritis (seperti pembuatan mount point disk atau penyiapan swap) dianggap sebagai *anti-pola* berbahaya dalam praktik SRE?
- A. Karena dapat memperlambat koneksi internet control node.
- B. Karena `ignore_errors` menyamarkan kegagalan fatal (*silent failure*), membuat status pipeline tampak sukses padahal infrastruktur dalam kondisi cacat, yang akan memicu kegagalan berantai yang jauh lebih parah pada tahap berikutnya.
- C. Karena modul tersebut otomatis memakan RAM sebesar 4GB.
- D. Karena `ignore_errors` hanya didukung pada sistem operasi Windows.

*Kunci Jawaban:* **B**. Kesalahan fatal yang diabaikan akan menghasilkan *broken foundation* yang menyebabkan kegagalan misterius saat Kubernetes k3s dipasang di atasnya.

### Soal 3 (Pilihan Ganda)
Dalam blok `block / rescue / always`, kapan task-task yang berada di dalam blok `always` akan dieksekusi oleh Ansible?
- A. Hanya jika seluruh task di dalam `block` berhasil tanpa error.
- B. Hanya jika terjadi error pada blok `block` dan ditangani oleh `rescue`.
- C. Selalu dieksekusi di akhir, baik saat blok `block` berhasil sempurna maupun saat terjadi kegagalan/rescue, ideal untuk pencatatan audit log forensik atau pembersihan file temporary.
- D. Tidak pernah dieksekusi jika mode `--check` aktif.

*Kunci Jawaban:* **C**. Blok `always` memiliki sifat persis seperti klausa `finally`, menjamin eksekusi pembersihan atau pencatatan jejak audit tanpa terpengaruh status akhir.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Anda adalah Lead SRE yang meninjau Merge Request dari junior engineer. Ia mengajukan script pipeline CI/CD yang langsung mengeksekusi `ansible-playbook -i inventories/production/hosts.yml playbooks/update-kernel.yml` ke seluruh 40 server bare-metal produksi setiap kali ada push ke branch `main`. Kritik desain pipeline tersebut dan susun 4 tahap pengaman (*guardrail pipeline*) yang wajib ditambahkan sebelum kode diizinkan masuk ke server produksi!

*Panduan Solusi:*
1. **Kritik Desain:**
   Pipeline tersebut memiliki risiko katastropik (*Uncontrolled Blast Radius*). Pembaruan kernel adalah operasi berisiko tinggi. Jika paket kernel baru memiliki *bug incompatibilty* atau *kernel panic*, seluruh 40 server bare-metal produksi akan *crash* secara serentak dalam waktu bersamaan (*Total Outage*), melumpuhkan operasional bisnis perusahaan.
2. **Empat Tahap Pengaman (Guardrails):**
   - **Tahap 1: Static Quality Gate & Syntax Check:**
     Wajibkan `ansible-lint` dan `ansible-playbook --syntax-check` di lingkungan CI runner sebelum ada koneksi SSH ke server.
   - **Tahap 2: Staging Canary Verification:**
     Uji perubahan pada lingkungan staging terlebih dahulu dengan verifikasi kesehatan layanan otomatis (*smoke test*).
   - **Tahap 3: Manual Approval Gate (4-Eyes Principle):**
     Jangan deploy otomatis saat push! Pasang aturan `when: manual` pada GitLab CI yang mewajibkan persetujuan Lead SRE setelah meninjau selisih *diff*.
   - **Tahap 4: Progressive Canary Rollout di Playbook:**
     Pada berkas playbook, wajibkan parameter pembatasan rolling:
     ```yaml
     serial:
       - 1    # Canary: 1 server dulu
       - 2    # Batch 2: 2 server
       - "25%"# Batch 3: bertahap
     any_errors_fatal: true
     ```
     Dengan arsitektur ini, jika terjadi kernel panic, insiden hanya mengenai 1 server canary, dan 39 server lainnya tetap aman melayani pengguna.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Roles, Collections, dan Struktur Repository](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/01-roles-collections-repository.md) — Mengorganisasikan kode ke dalam roles modular.
  - [02 — Jinja2, Variable Precedence, dan Vault Boundary](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/02-jinja-variables-vault.md) — Menjaga kerahasiaan data pada proses otomatisasi.
- **Praktik Laboratorium:**
  - [LAB-01 — Role Common Hardening](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/lab/LAB-01-role-common-hardening.md) — Membangun role OS hardening mandiri.
  - [LAB-02 — Vault dan Validasi CI Aman](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/lab/LAB-02-vault-safe-ci-validation.md) — Menguji skenario guardrail dan penanganan error di pipeline.
- **Tahap Berikutnya:**
  - [Modul 4.3 — Ansible k3s On-Prem](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/) — Menerapkan seluruh guardrail produksi ini untuk mengorkestrasi cluster Kubernetes k3s multi-node ber-MetalLB.
