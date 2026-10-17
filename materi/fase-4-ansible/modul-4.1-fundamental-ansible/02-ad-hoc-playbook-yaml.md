# 02 — Ad-hoc Command dan Playbook YAML: Dari Perintah Spontan Menuju Otomatisasi Terstruktur

> **Ringkasan:** Mengelola konfigurasi server dapat dilakukan melalui dua cara: perintah langsung satu baris (*Ad-hoc Command*) untuk investigasi cepat, atau dokumen prosedur otomatis (*Playbook YAML*) untuk perubahan terencana yang berulang. Di lingkungan *production*, mengandalkan ad-hoc untuk perubahan sistem adalah anti-pola yang melahirkan *snowflake servers* dan merusak prinsip *Infrastructure as Code*. Dokumen ini membedah perbedaan mendasar antara perintah ad-hoc dan playbook, menguraikan anatomi hierarki YAML, memanfaatkan pengendali alur (*loops*, *conditions*, *handlers*), serta membekali SRE dengan empat perisai keselamatan eksekusi: `--check`, `--diff`, `--limit`, dan `--tags`.

---

## 1. Analogi Pemula: Telepon Darurat Walkie-Talkie vs Buku SOP Gedung

Bagi engineer pemula, menulis perintah ad-hoc terasa sangat menggoda karena mirip dengan mengetik perintah shell biasa di terminal:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PERINTAH AD-HOC vs PLAYBOOK YAML                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ AD-HOC COMMAND: WALKIE-TALKIE DARURAT ]                                  │
│  - Situasi: Komandan SRE memegang walkie-talkie dan berkata:                │
│    "Pak Satpam di Server 3, tolong cek apakah lampu genset menyala!"        │
│  - Sifat: Spontan, cepat, sekali pakai, untuk observasi status.             │
│  - Kelemahan Fatal: Tidak ada jejak dokumen resmi, tidak ada tinjauan tim   │
│    (*peer review*), dan tidak bisa diulang secara identik oleh orang lain!  │
│                                                                             │
│  [ PLAYBOOK YAML: BUKU STANDAR OPERASIONAL PROSEDUR (SOP) RESMI ]           │
│  - Situasi: Tim SRE menyusun buku saku tebal: "Prosedur Penguncian Gedung   │
│    dan Hardening Keamanan Server Setiap Pergantian Shift".                  │
│  - Sifat: Tertulis rapi di Git, melalui peninjauan Pull Request, diuji coba│
│    di lingkungan simulasi, dan dieksekusi secara otomatis setiap saat.      │
│  - Keunggulan: Menghasilkan keadaan sistem yang seragam di 100 server       │
│    tanpa ada langkah yang terlewat atau bergantung pada daya ingat manusia! │
│                                                                             │
│  PRINSIP SRE PRODUCTION:                                                    │
│  Gunakan Ad-hoc HANYA untuk "Melihat & Memeriksa" (Read-Only).              │
│  Gunakan Playbook untuk SEMUA "Mengubah & Memasang" (Mutation State).        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Ad-hoc Commands: Kapan Digunakan dan Kapan Dilarang?

Perintah ad-hoc dijalankan melalui CLI `ansible` dengan sintaks:
```bash
ansible <target-hosts> -m <module-name> -a "<module-arguments>"
```

### 2.1 Matriks Etika Operasional Ad-hoc di Lingkungan Produksi

```text
┌──────────────────────────────────────┬──────────────────────────────────────┐
│       DIIZINKAN (READ-ONLY / PREFLIGHT)│    DILARANG KERAS (STATE MUTATION)   │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ [✓] Cek konektivitas SSH (`ping`)     │ [✗] Install paket software (`apt/yum`)│
│ [✓] Periksa sisa disk space (`df -h`) │ [✗] Edit file config secara langsung │
│ [✓] Kumpulkan metrik sistem (`setup`) │ [✗] Buka/tutup port firewall (`ufw`) │
│ [✓] Cek status layanan (`systemctl`)  │ [✗] Buat user baru atau ubah sudoers │
│ [✓] Verifikasi jam server (`chronyc`) │ [✗] Jalankan script migrasi database │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

### 2.2 Perbandingan Modul `command` vs `shell` vs Modul Native

Banyak pemula tergoda menggunakan modul `shell` untuk semua tugas:
```bash
# CARA BURUK (Fragile & Non-Idempotent):
ansible all -m shell -a "apt-get install -y nginx && systemctl start nginx"
```
*Mengapa ini buruk?*
1. **Tidak Idempoten:** Perintah di atas akan selalu melapor status `changed` dan berusaha menginstall ulang paket meskipun NGINX sudah terpasang sempurna.
2. **Bahaya Karakter Khusus:** Modul `shell` membuka sub-shell `/bin/sh` yang rentan terhadap *shell injection* jika argumen berasal dari variabel dinamis.
3. **Standar SRE:** Gunakan modul native Ansible (`ansible.builtin.apt` dan `ansible.builtin.systemd_service`) yang memiliki logika deteksi status (*state detection*) bawaan!

---

## 3. Anatomi Playbook YAML: Plays, Tasks, Modules, dan Handlers

Playbook adalah berkas teks berformat YAML yang mendefinisikan *desired state* (keadaan akhir yang diinginkan) dari sekumpulan server.

### 3.1 Struktur Hierarkis Dokumen Playbook

```text
Playbook (.yml)
  └── Play (Mendefinisikan target host, user, dan variabel)
        ├── Pre-tasks (Pemeriksaan awal)
        ├── Tasks (Daftar modul yang dieksekusi secara berurutan)
        │     ├── Task 1: Pastikan paket terinstal (Module: package)
        │     ├── Task 2: Salin konfigurasi (Module: template) ──> memicu Handler!
        │     └── Task 3: Pastikan service aktif (Module: service)
        └── Handlers (Task khusus yang hanya aktif jika dipicu oleh sinyal 'notify')
```

### 3.2 Contoh Playbook Baseline Hardening (`playbooks/baseline.yml`)

Perhatikan penulisan modul deklaratif dengan *Fully Qualified Collection Name* (FQCN) seperti `ansible.builtin.package`:

```yaml
---
- name: Terapkan konfigurasi baseline OS dan keamanan dasar
  hosts: k3s_cluster
  become: true
  gather_facts: true
  serial: 2 # Eksekusi bertahap maksimal 2 host sekaligus (Blast radius control)

  vars:
    required_packages:
      - curl
      - htop
      - iotop
      - chrony
      - jq
    security_banner_text: "PERINGATAN: Server ini dipantau oleh Tim SRE Bootcamp 2026."

  tasks:
    - name: 1. Pastikan paket utilitas dasar terinstal
      ansible.builtin.package:
        name: "{{ required_packages }}"
        state: present
      tags: [packages]

    - name: 2. Konfigurasikan banner peringatan SSH (/etc/issue.net)
      ansible.builtin.copy:
        content: "{{ security_banner_text }}\n"
        dest: /etc/issue.net
        owner: root
        group: root
        mode: "0644"
      notify: Restart layanan SSH
      tags: [security, ssh]

    - name: 3. Pastikan layanan sinkronisasi waktu (Chrony) berjalan
      ansible.builtin.service:
        name: chrony
        state: started
        enabled: true
      tags: [time, chrony]

  handlers:
    - name: Restart layanan SSH
      ansible.builtin.service:
        name: sshd
        state: restarted
```

> [!NOTE]
> **Cara Kerja Handlers & Sinyal `notify`:**
> Handler adalah *task pasif*. Pada task nomor 2 di atas, jika file `/etc/issue.net` sudah berisi teks yang sama persis, Ansible akan melapor `ok` (tidak ada perubahan). Karena tidak ada perubahan (*no change*), sinyal `notify` **TIDAK AKAN** memanggil handler `Restart layanan SSH`. Handler HANYA dieksekusi sekali di akhir Play jika dan hanya jika task yang bersangkutan berstatus `changed: true`!

---

## 4. Pengendali Alur Eksekusi: Loops, Conditions, dan Register

Playbook produksi membutuhkan logika cerdas untuk menyesuaikan konfigurasi dengan kondisi dinamis host.

### 4.1 Perulangan Elegan dengan `loop`
Daripada menulis 5 task terpisah untuk membuat 5 direktori, gunakan perulangan:

```yaml
- name: Buat struktur direktori operasional SRE
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: directory
    owner: "{{ item.owner }}"
    group: root
    mode: "{{ item.mode }}"
  loop:
    - { path: '/opt/sre/bin', owner: 'root', mode: '0755' }
    - { path: '/opt/sre/logs', owner: 'sre-admin', mode: '0750' }
    - { path: '/var/lib/k3s-audit', owner: 'root', mode: '0700' }
```

### 4.2 Percabangan Logika dengan `when`
Pastikan task hanya berjalan pada sistem operasi atau arsitektur tertentu:

```yaml
- name: Pasang modul kernel khusus Debian/Ubuntu
  ansible.builtin.modprobe:
    name: "{{ item }}"
    state: present
  loop:
    - overlay
    - br_netfilter
  when: ansible_facts.os_family == "Debian"
```

### 4.3 Menangkap Output dengan `register` dan `failed_when`
Terkadang Anda perlu menangkap hasil perintah CLI dan membuat keputusan berdasarkan output tersebut:

```yaml
- name: Cek status partisi swap memory
  ansible.builtin.command: swapon --show
  register: swap_check
  changed_when: false # Menjalankan perintah ini bukan bentuk perubahan konfigurasi!
  failed_when: false  # Jangan hentikan playbook jika exit code bukan 0

- name: Matikan swap secara darurat jika ditemukan aktif
  ansible.builtin.command: swapoff -a
  when: swap_check.stdout != ""
  changed_when: true
```

---

## 5. Empat Perisai Keselamatan SRE: Check, Diff, Limit, dan Tags

Sebelum menjalankan playbook yang berpotensi mengubah konfigurasi di server produksi, SRE wajib mengaktifkan **Empat Perisai Keselamatan**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   EMPAT PERISAI KESELAMATAN EKSEKUSI ANSIBLE                │
├─────────────────┬───────────────────────────────────────────────────────────┤
│ Perisai SRE     │ Fungsi & Cara Penggunaan                                  │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ 1. --check      │ Simulasi / Dry-Run: Menanyakan ke server apa yang akan     │
│                 │ berubah TANPA benar-benar melakukan modifikasi fisik.     │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ 2. --diff       │ Unified Diff: Menampilkan selisih baris teks konfigurasi   │
│                 │ (warna merah = dihapus, warna hijau = ditambah).          │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ 3. --limit      │ Blast Radius Control: Membatasi target eksekusi hanya ke  │
│                 │ 1 server kelinci percobaan (Canary), bukan seluruh data   │
│                 │ center sekaligus! Contoh: `--limit sre-k3s-cp-01`.        │
├─────────────────┼───────────────────────────────────────────────────────────┤
│ 4. --tags       │ Segmentasi Tugas: Hanya mengeksekusi task berlabel tag    │
│                 │ tertentu (misal: `--tags chrony`).                        │
└─────────────────┴───────────────────────────────────────────────────────────┘
```

### Contoh Eksekusi Standar Emas SRE (Golden Run):
```bash
# Langkah 1: Syntax Check
ansible-playbook -i inventories/lab/hosts.yml playbooks/baseline.yml --syntax-check

# Langkah 2: Dry-run & Diff pada 1 node Canary
ansible-playbook -i inventories/lab/hosts.yml playbooks/baseline.yml --check --diff --limit sre-k3s-cp-01

# Langkah 3: Eksekusi nyata bertahap setelah hasil diff diverifikasi
ansible-playbook -i inventories/lab/hosts.yml playbooks/baseline.yml --limit sre-k3s-cp-01
ansible-playbook -i inventories/lab/hosts.yml playbooks/baseline.yml
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN PLAYBOOK ANSIBLE                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: YAML SYNTAX PARSING ERROR ]                                     │
│  Gejala: Playbook langsung berhenti dengan error:                           │
│          "syntax error: mapping values are not allowed in this context"     │
│  Akar Masalah:                                                              │
│  - Menggunakan tombol TAB alih-alih SPASI. YAML membenci karakter tab!      │
│  - Terdapat titik dua tanpa spasi (`key:value` seharusnya `key: value`).   │
│  - String yang diawali tanda kurung kurawal `{{ var }}` tidak diapit kutip. │
│  Solusi SRE:                                                                │
│  1. Jalankan linter otomatis: `ansible-lint playbooks/baseline.yml`         │
│  2. Selalu apit string Jinja2 dengan kutip ganda: `name: "{{ my_var }}"`    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: HANDLER TIDAK PERNAH DIJALANKAN ]                               │
│  Gejala: Konfigurasi file sudah berubah, tapi service tidak pernah reload.  │
│  Akar Masalah:                                                              │
│  - Nama pada `notify` dan nama pada `handlers` berbeda huruf besar/kecil.   │
│  - Playbook mengalami kegagalan (*fatal error*) pada task setelahnya,       │
│    sehingga Ansible membatalkan eksekusi seluruh handler di akhir Play!     │
│  Solusi SRE:                                                                │
│  1. Samakan string `notify` dengan `handlers` secara presisi.               │
│  2. Jika handler wajib dijalankan meskipun ada task berikutnya yang error,  │
│     tambahkan opsi: `force_handlers: true` pada tingkat Play.               │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: MODUL SHELL SELALU MELAPOR "CHANGED" ]                          │
│  Gejala: Setiap kali playbook dijalankan, summary selalu `changed=1`,       │
│          merusak prinsip idempotency dan memicu handler berulang-ulang.     │
│  Akar Masalah:                                                              │
│  - Modul `shell` dan `command` tidak tahu apakah aksinya mengubah sistem    │
│    atau tidak, sehingga default-nya selalu mengembalikan `changed: true`.   │
│  Solusi SRE:                                                                │
│  1. Tambahkan `changed_when`:                                               │
│     - name: Jalankan script cek                                             │
│       ansible.builtin.shell: /opt/scripts/check.sh                          │
│       register: res                                                         │
│       changed_when: "'UPDATED' in res.stdout"                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Cek sintaksis playbook tanpa melakukan koneksi jaringan
ansible-playbook playbooks/site.yml --syntax-check

# 2. Menampilkan seluruh daftar task yang ada di dalam playbook
ansible-playbook playbooks/site.yml --list-tasks

# 3. Menampilkan seluruh daftar tag yang tersedia
ansible-playbook playbooks/site.yml --list-tags

# 4. Menjalankan playbook dalam mode simulasi visual perbandingan diff
ansible-playbook playbooks/site.yml --check --diff

# 5. Menjalankan playbook hanya untuk task yang memiliki tag tertentu
ansible-playbook playbooks/site.yml --tags "packages,security"

# 6. Memulai eksekusi playbook dari task tertentu yang sempat gagal
ansible-playbook playbooks/site.yml --start-at-task="Konfigurasikan banner"
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa seorang SRE tidak disarankan menggunakan modul `ansible.builtin.shell` untuk menginstal paket atau mengelola file konfigurasi di lingkungan produksi?
- A. Karena modul shell berjalan lebih lambat daripada modul command.
- B. Karena modul shell secara default tidak idempoten (selalu menghasilkan status `changed`), rentan terhadap shell injection, dan tidak memiliki penanganan status bawaan.
- C. Karena modul shell tidak didukung di sistem operasi Ubuntu Linux.
- D. Karena modul shell membutuhkan compiler gcc di server target.

*Kunci Jawaban:* **B**. Modul native Ansible seperti `package`, `copy`, dan `template` mengecek status aktual sebelum bertindak, menjamin konvergensi sistem yang aman.

### Soal 2 (Pilihan Ganda)
Sebuah task memiliki deklarasi `notify: Reload Nginx`, tetapi setelah playbook selesai dijalankan, service Nginx tidak mengalami reload sama sekali. Tidak ada pesan error yang muncul. Apa penyebab paling logis dari kejadian ini?
- A. Handler Nginx otomatis dibatalkan jika server memiliki RAM di atas 16GB.
- B. Task yang memicu `notify` berstatus `ok` (konfigurasi sudah sesuai/tidak ada perubahan), sehingga Ansible tidak memicu handler yang bersangkutan.
- C. Port 80 terblokir oleh firewall lokal.
- D. Nginx hanya bisa direstart menggunakan ad-hoc command.

*Kunci Jawaban:* **B**. Handler bersifat pasif dan hanya dijalankan jika task yang menotifikasinya menghasilkan perubahan nyata pada sistem (`changed: true`).

### Soal 3 (Pilihan Ganda)
Kapan flag `--limit` wajib disertakan saat menjalankan `ansible-playbook` di lingkungan produksi?
- A. Setiap kali SRE ingin menguji perubahan konfigurasi baru pada satu node percontohan (*canary*) guna membatasi radius dampak (*blast radius*).
- B. Hanya saat server kehabisan memori swap.
- C. Saat operator ingin mengubah password root.
- D. Saat file inventaris ditulis dalam format INI.

*Kunci Jawaban:* **A**. `--limit` adalah mekanisme kontrol blast radius utama untuk mencegah kegagalan konfigurasi serentak di seluruh klaster server.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Anda diminta mengonfigurasi aturan firewall `ufw` di 20 server produksi. Rekan tim Anda mengusulkan perintah ad-hoc:
```bash
ansible all -m shell -a "ufw default deny incoming && ufw enable"
```
Jelaskan bahaya fatal dari perintah tersebut bagi ketersediaan sistem, dan rancang pendekatan yang aman menggunakan playbook!

*Panduan Solusi:*
1. **Bahaya Fatal:**
   Perintah tersebut langsung mengaktifkan kebijakan `deny incoming` tanpa terlebih dahulu memastikan port SSH (port 22) diizinkan (`ufw allow 22/tcp`). Akibatnya, seketika perintah tersebut dieksekusi, koneksi SSH Ansible akan terputus seketika, dan seluruh 20 server produksi akan terkunci (*locked out*) dari akses remote! Administrator harus mendatangi fisik server atau konsol KVM hypervisor satu per satu.
2. **Pendekatan Playbook yang Aman:**
   Gunakan modul deklaratif `community.general.ufw` dengan urutan yang benar:
   ```yaml
   - name: Konfigurasi firewall aman
     hosts: all
     become: true
     tasks:
       - name: 1. Buka port SSH sebelum mengaktifkan firewall!
         community.general.ufw:
           rule: allow
           port: '22'
           proto: tcp

       - name: 2. Terapkan default deny incoming
         community.general.ufw:
           default: deny
           direction: incoming

       - name: 3. Aktifkan firewall
         community.general.ufw:
           state: enabled
   ```
   Uji terlebih dahulu menggunakan `--limit satu-server-lab` sebelum dijalankan ke cluster produksi.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Arsitektur Ansible, Inventory, dan SSH](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/01-arsitektur-inventory-ssh.md) — Memahami kontrol node, managed node, dan SSH preflight.
- **Topik Berikutnya:**
  - [03 — Idempotency, Variables, Facts, dan Handlers](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/03-idempotency-variables-facts-handlers.md) — Mendalami siklus hidup fakta sistem dan hierarki variabel.
  - [LAB-01 — Inventory, SSH, dan Ad-hoc](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/lab/LAB-01-inventory-ssh-ad-hoc.md) — Latihan praktikum di terminal.
