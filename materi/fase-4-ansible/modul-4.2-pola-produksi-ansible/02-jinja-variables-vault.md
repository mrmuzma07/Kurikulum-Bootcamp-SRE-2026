# 02 — Jinja2 Templating, Variable Precedence, dan Vault Boundary: Merancang Konfigurasi Dinamis Tanpa Membocorkan Rahasia

> **Ringkasan:** Otomatisasi konfigurasi tingkat lanjut memerlukan kemampuan menghasilkan berkas konfigurasi secara dinamis menyesuaikan peran, sistem operasi, dan lingkungan server. Mesin template **Jinja2** memungkinkan SRE menyusun konfigurasi yang fleksibel, sementara **Ansible Vault** memastikan data rahasia (*secrets*) seperti token klaster k3s, sertifikat TLS, dan kata sandi terenkripsi secara kriptografis menggunakan algoritma AES-256 di repositori Git. Dokumen ini membedah sintaksis Jinja2, teknik validasi berkas pra-aplikasi (*pre-apply validation*), arsitektur pemisahan berkas rahasia, serta integrasi kunci brankas pada pipeline CI/CD tanpa risiko kebocoran log.

---

## 1. Analogi Pemula: Surat Undangan Massal & Brankas Baja Kombinasi

Untuk memahami sinergi antara Jinja2 dan Ansible Vault, mari gunakan analogi persuratan kantor dan brankas finansial:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI JINJA2 TEMPLATE & ANSIBLE VAULT                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ JINJA2 TEMPLATE: SURAT RESMI CETAK MASSAL ]                              │
│  - Masalah: Anda harus mengirim surat resmi ke 100 karyawan dengan gaji,    │
│    nama, dan divisi yang berbeda-beda. Mengetik 100 surat manual memakan    │
│    waktu dan rentan salah ketik!                                            │
│  - Solusi Jinja2: Anda membuat satu cetak biru surat dengan penampung       │
│    variabel (*placeholders*):                                               │
│    "Yth. Saudara {{ nama }}, Anda ditugaskan di server {{ hostname }}..."   │
│  - Mesin Jinja2 mencetak 100 surat unik dalam 1 detik secara presisi!       │
│                                                                             │
│  [ ANSIBLE VAULT: BRANKAS BAJA TERKUNCI AES-256 ]                           │
│  - Masalah: Surat yang memuat kode PIN brankas atau bonus rahasia tidak     │
│    boleh tergeletak di atas meja kantor terbuka (Repositori Git Publik!).   │
│  - Solusi Vault: Seluruh dokumen rahasia dienkripsi dengan sandi baja.      │
│  - Di mata orang luar (atau hacker yang mencuri repositori Git), file       │
│    tersebut hanyalah teks acak tak terbaca:                                 │
│    `$ANSIBLE_VAULT;1.1;AES256;...`                                          │
│  - HANYA saat kurir resmi (Ansible Control Node) membawa "Kunci Kombinasi"  │
│    (Vault Password), dokumen tersebut dibuka sesaat di dalam memori, lalu   │
│    disuntikkan ke target, dan brankas terkunci kembali!                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Sintaksis & Filter Dinamis Jinja2 untuk SRE

Template Jinja2 disimpan dengan ekstensi berkas `.j2`. Di dalam template, ada tiga jenis penanda utama:
1. `{{ ekspresi }}` : Digunakan untuk mencetak nilai variabel atau hasil filter ke teks berkas.
2. `{% logika %}`   : Digunakan untuk struktur kontrol (perulangan `for` dan percabangan `if`).
3. `{# komentar #}` : Digunakan untuk catatan dokumentasi yang tidak akan dicetak ke berkas akhir.

### 2.1 Filter-Filter Esensial Jinja2 bagi SRE

```jinja2
{# 1. Mencegah File Rusak Jika Variabel Tidak Didefinisikan (Default Fallback) #}
server_port: {{ k3s_listen_port | default(6443, true) }}

{# 2. Menggabungkan Daftar Array Menjadi String yang Dipisahkan Koma #}
etcd_endpoints: "{{ k3s_server_ips | join(',') }}"

{# 3. Mengubah Format Huruf #}
log_level: {{ app_log_level | lower }}

{# 4. Manipulasi Alamat Jaringan (Membutuhkan ansible.utils collection) #}
cluster_network_gateway: {{ k3s_cluster_cidr | ansible.utils.ipaddr('net') }}
```

### 2.2 Contoh Nyata: Konfigurasi Dinamis k3s Server (`config.yaml.j2`)

Perhatikan bagaimana template di bawah ini menyesuaikan diri apakah ia sedang dijalankan pada master pertama (bootstrap) atau master tambahan:

```yaml
{# templates/config.yaml.j2 - Konfigurasi Resmi k3s Server #}
# Berkas ini dikelola secara otomatis oleh Ansible - JANGAN EDIT MANUAL!
# Dibuat pada: {{ ansible_date_time.iso8601 }}

write-kubeconfig-mode: "0644"
tls-san:
  - "{{ k3s_vip_address }}"
  - "{{ ansible_facts.default_ipv4.address }}"
  - "{{ inventory_hostname }}"

# Konfigurasi Jaringan Cluster
cluster-cidr: "{{ k3s_pod_network_cidr | default('10.42.0.0/16') }}"
service-cidr: "{{ k3s_service_network_cidr | default('10.43.0.0/16') }}"

{% if is_bootstrap_master %}
# Node ini adalah inisiator Quorum Etcd Pertama
cluster-init: true
{% else %}
# Node ini bergabung ke cluster yang sudah ada
server: "https://{{ k3s_bootstrap_server_ip }}:6443"
token: "{{ vault_k3s_cluster_token }}"
{% endif %}

# Komponen yang Dinonaktifkan sesuai Arsitektur Bootcamp SRE On-Prem
disable:
  - servicelb # Digantikan oleh MetalLB L2/BGP
  - traefik   # Digantikan oleh Nginx Ingress Controller atau Helm Chart terpisah
```

---

## 3. Validasi Sintaksis Pra-Aplikasi (`validate` Parameter)

Salah satu kesalahan paling berbahaya yang dilakukan pemula adalah menyalin template konfigurasi yang memiliki salah ketik (*typo*) ke direktori sistem seperti `/etc/sudoers`, `/etc/nginx/nginx.conf`, atau `/etc/ssh/sshd_config`. Ketika service di-reload, daemon akan *crash* seketika!

Ansible menyediakan parameter penyelamat: **`validate`**.

```yaml
- name: Terapkan konfigurasi SSHD dengan proteksi validasi
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: "0600"
    # %s merepresentasikan path berkas temporary sebelum berkas asli ditimpa!
    validate: "/usr/sbin/sshd -t -f %s"
  notify: Reload SSH Daemon
```

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 MEKANISME KERJA PARAMETER VALIDATE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Ansible merender template ke berkas temporary: `/tmp/.ansible_xyz123`.  │
│  2. Ansible mengeksekusi perintah validasi: `/usr/sbin/sshd -t -f /tmp/...` │
│  3. Cabang Keputusan:                                                       │
│     - JIKA EXIT CODE = 0 (Lolos): Berkas temporary dipindahkan menimpa      │
│       `/etc/ssh/sshd_config`, lalu memicu sinyal `notify`.                  │
│     - JIKA EXIT CODE != 0 (Gagal Sintaks): Eksekusi BERHENTI SEKETIKA!      │
│       Berkas asli `/etc/ssh/sshd_config` TETAP UTUH & SERVICE TIDAK MATI!   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Contoh `validate` esensial lainnya:
- Sudoers: `validate: "/usr/sbin/visudo -cf %s"`
- Nginx: `validate: "/usr/sbin/nginx -t -c %s"`
- Crontab: `validate: "/usr/bin/crontab -u root -T %s"`

---

## 4. Arsitektur Ansible Vault: Mengamankan Rahasia Produksi

**Ansible Vault** mengenkripsi variabel atau seluruh berkas teks menggunakan enkripsi simetris **AES-256**. Berkas terenkripsi aman disimpan ke dalam repositori Git publik maupun privat.

### 4.1 Strategi Produksi SRE: File-Level vs Variable-Level Encryption

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DUA STRATEGI ENKRIPSI ANSIBLE VAULT                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ STRATEGI A: ENKRIPSI SELURUH FILE (FILE-LEVEL) ]                         │
│  - Seluruh isi berkas `group_vars/production/vault.yml` dienkripsi total.   │
│  - Kelemahan SRE: Saat melakukan Pull Request di Git, rekan tim Anda tidak  │
│    bisa melihat apa yang berubah (*git diff* hanya menampilkan ciphertext). │
│                                                                             │
│  [ STRATEGI B: HYBRID STRUCTURE (SANGAT DIREKOMENDASIKAN) ]                 │
│  - Buat dua file berdampingan:                                              │
│    1. `group_vars/production/vars.yml` (Plaintext - Bebas rahasia):         │
│       Memuat port, nama host, CIDR, opsi non-sensitif (Terbaca di Git Diff).│
│    2. `group_vars/production/vault.yml` (Enkripsi Penuh - Khusus Rahasia):  │
│       HANYA memuat: kata sandi database, API tokens, k3s cluster join token,│
│       dan private TLS keys.                                                 │
│                                                                             │
│  [ STRATEGI C: INLINE VARIABLE ENCRYPTION (TERBAIK UNTUK AUDIT) ]           │
│  - Hanya satu variabel rahasia yang dienkripsi langsung di dalam berkas:    │
│    k3s_api_token: !vault |                                                  │
│          $ANSIBLE_VAULT;1.1;AES256                                          │
│          3432393361306338663866633630656661386131343734616262333036643232...│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Perintah Operasional CLI Ansible Vault

```bash
# 1. Membuat file rahasia baru yang terenkripsi
ansible-vault create inventories/production/group_vars/vault.yml

# 2. Mengenkripsi file plaintext yang sudah ada
ansible-vault encrypt inventories/production/group_vars/vault.yml

# 3. Melihat isi berkas tanpa mendekripsi file di disk (Read-Only)
ansible-vault view inventories/production/group_vars/vault.yml

# 4. Mengedit isi rahasia (membuka editor nano/vim sementara)
ansible-vault edit inventories/production/group_vars/vault.yml

# 5. Mengganti kata sandi master vault (Rekey)
ansible-vault rekey inventories/production/group_vars/vault.yml

# 6. Mengenkripsi string teks pendek untuk disalin inline (!vault)
ansible-vault encrypt_string --name 'vault_k3s_token' 'K3sSuperSecretToken2026!'
```

---

## 5. Integrasi CI/CD & Proteksi Log (`no_log: true`)

Di lingkungan otomatisasi GitOps atau pipeline CI/CD (seperti GitLab CI), manusia tidak berada di depan terminal untuk mengetikkan password vault secara interaktif.

### 5.1 Menyuplai Password Vault Secara Non-Interaktif
Gunakan berkas kunci sementara yang disuntikkan dari Secret Variable GitLab CI:

```bash
# Opsi 1: Menggunakan flag --vault-password-file
echo "$CI_VAULT_PASSWORD" > /tmp/.vault_pass
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml \
  --vault-password-file /tmp/.vault_pass
rm -f /tmp/.vault_pass

# Opsi 2: Menggunakan Environment Variable resmi Ansible
export ANSIBLE_VAULT_PASSWORD_FILE=/path/to/.vault_pass
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml
```

### 5.2 Perlindungan Kebocoran Log dengan `no_log: true`
Ansible secara default akan mencetak argumen modul ke layar output. Jika sebuah task menangani variabel rahasia, SRE **WAJIB** menambahkan `no_log: true`:

```yaml
- name: Tuliskan join token k3s ke berkas sistem terisolasi
  ansible.builtin.copy:
    content: "{{ vault_k3s_cluster_token }}\n"
    dest: /var/lib/rancher/k3s/server/token
    owner: root
    group: root
    mode: "0600"
  no_log: true # MENCEGAH KATA SANDI TERCETAK DI LOG CI/CD DAN TERMINAL!
```

> [!CAUTION]
> **Dua Sisi Mata Uang `no_log: true`:**
> Parameter `no_log: true` mematikan semua output stdout/stderr dari task tersebut. Jika task mengalami kegagalan, Ansible hanya akan melapor `fatal: [host]: FAILED! => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this task"}`. Saat melakukan *debugging* di lab, Anda dapat menonaktifkannya sementara, tetapi di lingkungan produksi parameter ini wajib menyala!

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA JINJA2 DAN ANSIBLE VAULT                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "Decryption failed (no vault secrets found that can decrypt)" ] │
│  Gejala: Playbook langsung mati saat membaca file `vault.yml`.              │
│  Akar Masalah:                                                              │
│  - Password vault yang dimasukkan salah, atau file password kosong di CI.   │
│  - Menggunakan Vault ID ganda (`--vault-id`) yang labelnya tidak cocok.     │
│  Solusi SRE:                                                                │
│  Uji dekripsi secara manual dari terminal:                                  │
│  ansible-vault view group_vars/vault.yml --vault-password-file /path/pass   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "AnsibleUndefinedVariable: 'xxx' is undefined" ]                │
│  Gejala: Task template gagal me-render berkas karena variabel tidak ada.    │
│  Akar Masalah:                                                              │
│  - Penamaan variabel salah ketik (*typo*) antara inventory dan template.    │
│  Solusi SRE:                                                                │
│  1. Selalu sediakan filter default di template: `{{ my_var | default('') }}`│
│  2. Pasang preflight assertion di awal playbook:                            │
│     - ansible.builtin.assert:                                               │
│         that: my_var is defined                                             │
│         fail_msg: "Variabel my_var wajib didefinisikan sebelum lanjut!"     │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: KEBOCORAN RAHASIA LEWAT FLAG `--diff` ]                         │
│  Gejala: Operator menjalankan `ansible-playbook --diff`. Seluruh password   │
│          plaintext terpampang jelas di konsol perbandingan baris!           │
│  Akar Masalah:                                                              │
│  - Task template atau copy yang menangani password tidak diberi `no_log`.   │
│  Solusi SRE:                                                                │
│  Task yang memiliki `no_log: true` secara otomatis akan menyembunyikan     │
│  tampilan diff, melindungi kredensial dari rekaman riwayat konsol.          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menjalankan playbook dengan prompt password vault interaktif
ansible-playbook -i inventories/lab/hosts.yml playbooks/site.yml --ask-vault-pass

# 2. Menjalankan playbook dengan password dari file lokal tersembunyi
ansible-playbook -i inventories/lab/hosts.yml playbooks/site.yml --vault-password-file ~/.vault_pass

# 3. Mendekripsi file vault secara permanen (Hati-hati: jangan commit file ini!)
ansible-vault decrypt inventories/lab/group_vars/vault.yml

# 4. Mengenkripsi kembali file ke format brankas terlindungi
ansible-vault encrypt inventories/lab/group_vars/vault.yml

# 5. Memeriksa file template apakah menghasilkan sintaks yang valid secara offline
ansible all -i "localhost," -c local -m template -a "src=templates/config.j2 dest=/tmp/test.conf"
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa seorang SRE sangat disarankan menambahkan parameter `validate: "/usr/sbin/nginx -t -c %s"` pada task yang merender template konfigurasi Nginx?
- A. Agar Nginx otomatis melakukan reload tanpa perlu handler.
- B. Memastikan template diuji sintaksisnya pada file temporary sebelum menimpa file konfigurasi asli di `/etc/nginx/`, mencegah layanan crash jika ada kesalahan ketik.
- C. Untuk mengompresi file konfigurasi menjadi gzip.
- D. Agar Nginx dapat berjalan tanpa izin akses root.

*Kunci Jawaban:* **B**. Parameter `validate` bertindak sebagai *safety net* yang mencegah file konfigurasi yang cacat merusak kestabilan layanan sistem operasi.

### Soal 2 (Pilihan Ganda)
Di sebuah repositori Git publik, Anda menemukan file bernama `vault.yml` yang berisi teks:
`$ANSIBLE_VAULT;1.1;AES256;623432...`
Apakah file ini aman di-commit ke sistem kontrol versi Git?
- A. Tidak aman sama sekali, karena format vault bisa didekripsi oleh siapa saja tanpa password.
- B. Aman, karena seluruh konten telah dienkripsi menggunakan algoritma standar militer AES-256 dan hanya dapat dibaca oleh pemegang kunci master Ansible Vault.
- C. Aman hanya jika ukuran filenya di bawah 1 KB.
- D. Tidak aman, karena Ansible Vault hanya melakukan encoding Base64.

*Kunci Jawaban:* **B**. Ansible Vault menggunakan kriptografi simetris AES-256 yang aman untuk disimpan di repositori Git, asalkan password master-nya dijaga ketat dan tidak bocor.

### Soal 3 (Pilihan Ganda)
Apa dampak langsung dari menambahkan baris `no_log: true` pada sebuah task Ansible di lingkungan produksi?
- A. Task tersebut tidak akan dieksekusi oleh managed node.
- B. Ansible tidak akan mencetak argumen, output, dan selisih diff dari task tersebut ke log konsol maupun log file, mencegah kredensial rahasia terbaca di rekaman CI/CD.
- C. Server target otomatis mematikan layanan syslog.
- D. Playbook akan mengeksekusi task berikutnya secara asinkron.

*Kunci Jawaban:* **B**. `no_log: true` adalah instrumen kepatuhan keamanan utama (*Security Hygiene*) untuk mencegah kebocoran rahasia di layar terminal dan log audit.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Seorang junior engineer menyimpan seluruh konfigurasi cluster staging ke dalam satu file terenkripsi `group_vars/staging/vault.yml`. Akibatnya, setiap kali ada perubahan kecil (seperti menaikkan log level dari `INFO` ke `DEBUG`), merge request di GitLab hanya menampilkan tumpukan teks ciphertext yang tidak bisa ditinjau oleh reviewer. Rancang struktur pemisahan variabel yang memenuhi standar transparansi peninjauan kode (*Code Review Transparency*) sekaligus menjaga kerahasiaan data!

*Panduan Solusi:*
1. **Analisis Masalah:**
   Mengenkrpsi variabel non-sensitif merusak prinsip transparansi dan auditabilitas tim. Reviewer tidak bisa mengetahui apakah perubahan tersebut aman atau merusak sistem karena `git diff` tidak terbaca.
2. **Desain Solusi Pemisahan Variabel (The Hybrid Pattern):**
   Pecah menjadi dua berkas terpisah di dalam `inventories/staging/group_vars/`:
   - **`vars.yml` (Plaintext / Terbaca di Git):**
     Menampung seluruh konfigurasi non-rahasia yang boleh dan perlu ditinjau:
     ```yaml
     app_log_level: "DEBUG"
     app_port: 8080
     k3s_node_count: 3
     k3s_cluster_cidr: "10.42.0.0/16"
     ```
   - **`vault.yml` (Hanya Rahasia Murni / Enkripsi AES-256):**
     HANYA menampung variabel yang mengandung data kredensial:
     ```yaml
     vault_database_password: "StagingSecretPassword123!"
     vault_k3s_join_token: "K3sJoinSecretStaging2026"
     ```
   Dengan pemisahan ini, perubahan log level pada `vars.yml` akan terlihat jelas di Git diff (`- app_log_level: "INFO"` -> `+ app_log_level: "DEBUG"`), sementara data kata sandi tetap terlindungi di dalam brankas `vault.yml`.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Roles, Collections, dan Struktur Repository](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/01-roles-collections-repository.md) — Mengorganisasikan template ke dalam folder role.
- **Topik Berikutnya:**
  - [03 — Check, Diff, Limit, Error Handling, dan CI](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/03-check-diff-limit-error-handling-ci.md) — Membangun guardrail eksekusi dan integrasi pipa otomatisasi CI/CD.
  - [LAB-02 — Vault dan Validasi CI Aman](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/lab/LAB-02-vault-safe-ci-validation.md) — Praktik hands-on enkripsi dan pengujian pipeline.
