# 01 — Roles, Collections, dan Struktur Repository: Mengorganisasi Kode Menuju Skala Produksi

> **Ringkasan:** Ketika otomatisasi berkembang dari beberapa server lab menjadi ratusan server produksi, meletakkan seluruh task di dalam satu file playbook monolitik adalah resep bencana. Kode menjadi sulit dipelihara, rentan terhadap konflik variabel, dan mustahil diuji secara independen. Ansible memecahkan tantangan ini melalui konsep **Roles** (modularisasi kode berbasis direktori) dan **Collections** (paket modul dan plugin eksternal standar industri). Dokumen ini membedah arsitektur repositori produksi, mendefinisikan kontrak peran (*Role Contract*), mengunci versi dependensi eksternal, serta menerapkan kerangka pengujian role secara aman.

---

## 1. Analogi Pemula: Laci Dapur Berantakan vs Kotak Perkakas Mekanik F1

Bagi pemula, menulis semua instruksi di dalam satu file `site.yml` setebal 2.000 baris terasa cepat di awal:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PLAYBOOK MONOLITIK vs ANSIBLE ROLES                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PLAYBOOK MONOLITIK: LACI DAPUR KEKACAUAN ]                              │
│  - Pisau daging, garpu, bumbu dapur, obeng, dan deterjen ditaruh di satu    │
│    laci besar yang sama.                                                    │
│  - Saat Anda hanya butuh menabur garam (mengupdate Chrony NTP), Anda        │
│    harus mengaduk-aduk seluruh laci dan berisiko menumpahkan deterjen ke    │
│    dalam masakan (merusak konfigurasi firewall atau swap)!                  │
│                                                                             │
│  [ ANSIBLE ROLES: KOTAK PERKAKAS MEKANIK FORMULA 1 (F1) ]                   │
│  - Kotak perkakas mekanik F1 memiliki laci-laci modular berlabel baku:      │
│    ├── Laci `defaults/`: Suku cadang standar bawaan pabrik.                │
│    ├── Laci `tasks/`: Urutan langkah kerja pembongkaran mesin.              │
│    ├── Laci `handlers/`: Prosedur darurat menyalakan APAR jika ada api.     │
│    ├── Laci `templates/`: Cetak biru desain sayap mobil.                    │
│    └── Laci `meta/`: Buku manual tipe mobil dan dependensi oli.             │
│                                                                             │
│  KEUNGGULAN SRE:                                                            │
│  1. Terisolasi (*Encapsulation*): Memperbaiki role Chrony tidak akan        │
│     menyenggol role Kubernetes atau firewall.                               │
│  2. Dapat Digunakan Ulang (*Reusability*): Role yang sama bisa dipakai di   │
│     cluster lab, staging, hingga multi-datacenter on-premises.              │
│  3. Kolaborasi Tim: 5 engineer SRE bisa mengembangkan 5 role berbeda        │
│     secara paralel tanpa konflik merge Git!                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Standar Struktur Direktori Repositori Ansible Produksi

Struktur direktori yang rapi adalah fondasi tata kelola (*governance*) infrastruktur. Di bawah ini adalah struktur repositori standar industri yang digunakan pada Bootcamp SRE:

```text
ansible-production/
├── ansible.cfg                          # Konfigurasi global Ansible & optimasi SSH
├── collections/
│   └── requirements.yml                 # Daftar & versi lock Ansible Collections eksternal
├── inventories/                         # Inventaris dipisahkan ketat per environment
│   ├── lab/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── k3s_cluster.yml
│   └── production/
│       ├── hosts.yml
│       ├── group_vars/
│       │   ├── all.yml
│       │   └── vault.yml                # Variabel rahasia terenkripsi (Ansible Vault)
│       └── host_vars/
│           └── sre-k3s-cp-01.yml
├── playbooks/                           # Orkestrasi tingkat tinggi (hanya memanggil roles)
│   ├── site.yml                         # Master playbook
│   ├── bootstrap-os.yml                 # Playbook inisialisasi awal server
│   └── deploy-k3s.yml                   # Playbook instalasi cluster k3s
└── roles/                               # Modul-modul tugas independen
    ├── common_hardening/               # Role OS baseline, user, sysctl, banner
    ├── chrony_time/                     # Role sinkronisasi waktu presisi
    ├── ufw_firewall/                    # Role pengelolaan port & iptables
    └── k3s_server/                      # Role deployment k3s control plane
```

---

## 3. Anatomi Sebuah Role & Role Contract

Setiap role memiliki konvensi folder internal yang telah dipahami oleh Ansible engine secara otomatis. Anda tidak perlu menggunakan kata kunci `include_tasks` untuk file standar:

```text
roles/chrony_time/
├── defaults/
│   └── main.yml        # [PUBLIC API] Nilai default aman yang boleh dioverride pengguna
├── vars/
│   └── main.yml        # [PRIVATE CONSTANTS] Konstanta internal role (JANGAN dioverride!)
├── tasks/
│   └── main.yml        # [WORKFLOW] Alur task deklaratif yang dieksekusi
├── handlers/
│   └── main.yml        # [EVENT TRIGGERS] Aksi restart/reload daemon
├── templates/
│   └── chrony.conf.j2  # [CONFIG BLUEPRINT] Template dinamis Jinja2
└── meta/
    └── main.yml        # [METADATA] Daftar dependensi role & kompatibilitas OS
```

### Definisi Kontrak Peran (*The Role Contract*)
Sebuah role kelas produksi harus memiliki kontrak yang jelas pada file `defaults/main.yml`. Nilai default harus didesain **aman (*fail-safe*)**:

```yaml
# roles/chrony_time/defaults/main.yml
---
# Parameter Publik Role Chrony Time (Dapat dioverride via group_vars)

# Daftar server NTP upstream (Default ke pool publik, disarankan override ke IP lokal on-prem)
chrony_ntp_servers:
  - "0.pool.ntp.org"
  - "1.pool.ntp.org"

# Maksimum toleransi perbedaan waktu sebelum service memaksa lonjakan waktu (step)
chrony_makestep_threshold: 0.1
chrony_makestep_limit: 3

# Status layanan di systemd
chrony_service_enabled: true
chrony_service_state: "started"

# Logging & metrik
chrony_enable_tracking_log: true
```

Task di dalam role (`roles/chrony_time/tasks/main.yml`):
```yaml
---
- name: 1. Pastikan paket chrony terpasang
  ansible.builtin.package:
    name: chrony
    state: present

- name: 2. Terapkan file konfigurasi chrony dari template
  ansible.builtin.template:
    src: chrony.conf.j2
    dest: /etc/chrony/chrony.conf
    owner: root
    group: root
    mode: "0644"
  notify: Restart Chrony Daemon

- name: 3. Pastikan daemon chrony berjalan dan aktif saat boot
  ansible.builtin.service:
    name: chrony
    enabled: "{{ chrony_service_enabled }}"
    state: "{{ chrony_service_state }}"
```

Handler di dalam role (`roles/chrony_time/handlers/main.yml`):
```yaml
---
- name: Restart Chrony Daemon
  ansible.builtin.service:
    name: chrony
    state: restarted
```

---

## 4. Ansible Collections & Penguncian Versi (Version Pinning)

Sebelum Ansible 2.9, semua modul (AWS, Azure, Docker, Linux, Windows) digabung ke dalam satu binary raksasa. Mulai Ansible 2.10+, arsitektur dipecah menjadi **Ansible Core** (engine minimalis) dan **Ansible Collections** (koleksi modul independen yang dikelola komunitas atau vendor).

### 4.1 Mendefinisikan Dependensi di `collections/requirements.yml`
Di lingkungan produksi, **DILARANG MENGGUNAKAN VERSI TERBARU SECARA BUTA (*FLOATING VERSION*)**. Anda wajib mengunci versi (*pinning*) agar rilis baru modul pihak ketiga tidak merusak pipeline CI/CD:

```yaml
---
# collections/requirements.yml
collections:
  # Koleksi modul POSIX (mount, sysctl, firewalld, cron)
  - name: ansible.posix
    version: "1.5.4"

  # Koleksi modul komunitas (ufw, docker, general utilities)
  - name: community.general
    version: "9.5.0"

  # Koleksi pengelolaan sertifikat TLS / x509
  - name: community.crypto
    version: "2.22.1"
```

### 4.2 Mengunduh dan Memvalidasi Collections
Jalankan instalasi koleksi sebelum mengeksekusi playbook:
```bash
ansible-galaxy collection install -r collections/requirements.yml -p ./collections
```

---

## 5. Mengintegrasikan Roles ke dalam Playbook Produksi

Setelah roles terstruktur, file playbook utama (`playbooks/site.yml`) menjadi sangat bersih, ringkas, dan mudah dibaca oleh level manajerial:

```yaml
---
# playbooks/site.yml - Master Orchestration Playbook

- name: 1. Terapkan OS Baseline & Sinkronisasi Jam di Seluruh Node
  hosts: k3s_cluster
  become: true
  roles:
    - role: common_hardening
      tags: [baseline]
    - role: chrony_time
      tags: [time, chrony]

- name: 2. Konfigurasi Firewall & Isolasi Port
  hosts: k3s_cluster
  become: true
  roles:
    - role: ufw_firewall
      tags: [firewall]

- name: 3. Bootstrap Kubernetes k3s Control Plane
  hosts: k3s_servers
  become: true
  serial: 1 # Inisialisasi master satu per satu untuk menjaga quorum etcd
  roles:
    - role: k3s_server
      tags: [k3s, control-plane]
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA MASALAH ROLES & COLLECTIONS                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: ROLE VARIABLE PRECEDENCE COLLISION ]                            │
│  Gejala: Anda mengubah variabel di `group_vars/production.yml`, namun saat  │
│          playbook dijalankan, nilai tersebut tidak berubah!                 │
│  Akar Masalah:                                                              │
│  - Penulis role meletakkan variabel tersebut di `roles/<role>/vars/main.yml`│
│    alih-alih di `defaults/main.yml`. Ingat: `vars/` memiliki prioritas jauh │
│    lebih tinggi daripada `group_vars/`, sehingga tidak bisa dioverride!     │
│  Solusi SRE:                                                                │
│  Pindahkan semua variabel konfigurasi publik ke `defaults/main.yml`.        │
│  Gunakan `vars/main.yml` HANYA untuk konstanta internal yang tidak boleh    │
│  disentuh pengguna luar.                                                    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: DEPENDENCY LOOP PADA ROLE META ]                                │
│  Gejala: Ansible berhenti dengan error siklus tak berujung:                 │
│          "fatal: Dependency loop detected in roles..."                      │
│  Akar Masalah:                                                              │
│  - Role A mendefinisikan Role B di `meta/main.yml`, dan Role B juga        │
│    mendefinisikan Role A sebagai dependensinya.                             │
│  Solusi SRE:                                                                │
│  1. Hindari circular dependency! Buat dependensi bersifat satu arah:        │
│     Role Aplikasi -> bergantung pada -> Role Baseline.                      │
│  2. Role Baseline tidak boleh memiliki dependensi ke role apa pun.          │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: MISSING COLLECTION DI PIPELINE CI/CD ]                          │
│  Gejala: Pipeline GitLab CI gagal dengan pesan:                             │
│          "couldn't resolve module/action 'community.general.ufw'"           │
│  Akar Masalah:                                                              │
│  - Runner CI/CD baru saja di-spawn dan belum mengunduh koleksi eksternal.   │
│  Solusi SRE:                                                                │
│  Tambahkan tahap pre-flight di `.gitlab-ci.yml`:                            │
│  before_script:                                                             │
│    - ansible-galaxy collection install -r collections/requirements.yml      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menginisialisasi kerangka struktur role baru secara otomatis
ansible-galaxy role init roles/my_new_role

# 2. Mengunduh dependensi koleksi eksternal sesuai file requirements
ansible-galaxy collection install -r collections/requirements.yml

# 3. Menampilkan daftar koleksi yang terpasang di sistem beserta versinya
ansible-galaxy collection list

# 4. Menjalankan syntax check pada playbook yang memanggil berbagai roles
ansible-playbook playbooks/site.yml --syntax-check

# 5. Menjalankan hanya satu role tertentu menggunakan filter tags
ansible-playbook playbooks/site.yml --tags "chrony"

# 6. Menjalankan linter standar untuk mendeteksi pelanggaran best practice
ansible-lint roles/chrony_time/
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Di manakah lokasi yang paling tepat untuk mendefinisikan variabel publik di dalam sebuah Ansible Role agar nilainya dapat dengan mudah ditimpa (*override*) oleh konfigurasi di `group_vars` lingkungan produksi?
- A. `roles/<role_name>/vars/main.yml`
- B. `roles/<role_name>/defaults/main.yml`
- C. `roles/<role_name>/meta/main.yml`
- D. `roles/<role_name>/templates/main.yml`

*Kunci Jawaban:* **B**. `defaults/main.yml` memiliki presedensi paling rendah dalam hierarki variabel, menjadikannya lokasi ideal untuk mendefinisikan parameter publik (*Public API*) yang fleksibel.

### Soal 2 (Pilihan Ganda)
Mengapa mengunci versi koleksi (*version pinning*) pada berkas `collections/requirements.yml` merupakan kewajiban operasional dalam tata kelola SRE?
- A. Agar Ansible dapat mengunduh koleksi tanpa koneksi internet.
- B. Mencegah terjadinya kerusakan pipa CI/CD secara mendadak akibat *breaking changes* atau pembaruan modul tak terduga dari repositori publik upstream.
- C. Karena Ansible engine menolak membaca file tanpa deklarasi versi.
- D. Untuk memperkecil ukuran memori RAM server target.

*Kunci Jawaban:* **B**. Pinning versi menjamin stabilitas dan determinisme eksekusi otomatisasi, mencegah efek samping pembaruan liar dari modul pihak ketiga.

### Soal 3 (Pilihan Ganda)
Dalam berkas `playbooks/site.yml`, seorang engineer mendefinisikan `serial: 1` pada Play instalasi control-plane k3s. Apa tujuan operasional SRE dari konfigurasi ini?
- A. Agar instalasi berjalan lebih hemat kuota internet.
- B. Membatasi eksekusi instalasi hanya satu server dalam satu waktu guna melindungi kestabilan *quorum* algoritma konsensus Raft pada etcd.
- C. Memaksa server melakukan reboot satu per satu.
- D. Mengubah format output Ansible menjadi satu baris.

*Kunci Jawaban:* **B**. Inisialisasi cluster ber-etcd membutuhkan urutan deterministik (bootstrap node pertama selesai terlebih dahulu, baru node berikutnya bergabung satu demi satu) agar quorum tidak pecah.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Tim Anda memiliki 3 lingkungan terpisah: `lab`, `staging`, dan `production`. Seorang engineer baru menyarankan untuk membuat 3 role berbeda: `chrony_lab`, `chrony_staging`, dan `chrony_prod`. Sebagai Lead SRE, jelaskan mengapa usulan ini melanggar prinsip *Don't Repeat Yourself (DRY)* dan arsitektur modular, serta jelaskan bagaimana merestrukturisasinya dengan benar!

*Panduan Solusi:*
1. **Kritik Arsitektur:**
   Membuat 3 role terpisah untuk fungsi yang sama menghasilkan duplikasi kode (*code redundancy*). Jika ada pembaruan security patch pada konfigurasi Chrony, tim harus mengedit 3 file terpisah, meningkatkan risiko inkonsistensi (*configuration drift*) dan kesalahan manusia.
2. **Restrukturisasi yang Benar:**
   - Gunakan **SATU ROLE TUNGGAL**: `roles/chrony/`.
   - Simpan logika instalasi dan template di dalam role tersebut dengan nilai default netral pada `roles/chrony/defaults/main.yml`.
   - Pisahkan parameter spesifik lingkungan ke dalam folder inventaris masing-masing:
     - `inventories/lab/group_vars/all.yml` -> mengarah ke NTP publik.
     - `inventories/production/group_vars/all.yml` -> mengarah ke NTP internal data center.
   - Dengan pola ini, kode logika tetap satu sumber kebenaran (*single source of truth*), sedangkan data konfigurasi bervariasi sesuai target lingkungan.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Modul 4.1 — Fundamental Ansible](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/) — Memahami arsitektur agentless, ad-hoc, playbook, dan konvergensi idempoten.
- **Topik Berikutnya:**
  - [02 — Jinja2, Variable Precedence, dan Vault Boundary](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/02-jinja-variables-vault.md) — Menguasai pembuatan file konfigurasi dinamis dan pengamanan rahasia dengan enkripsi Vault.
  - [03 — Check, Diff, Limit, Error Handling, dan CI](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/03-check-diff-limit-error-handling-ci.md) — Membangun guardrail pipeline CI/CD kelas produksi.
