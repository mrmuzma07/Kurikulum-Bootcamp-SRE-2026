# 01 — Arsitektur Ansible, Inventory, dan SSH: Mengorkestrasi Server Tanpa Agen (Agentless)

> **Ringkasan:** Di dunia *Site Reliability Engineering* (SRE), mengelola puluhan hingga ratusan server fisik atau mesin virtual (VM) secara manual via SSH adalah bentuk *toil* (pekerjaan repetitif tanpa nilai tambah) yang berbahaya dan memicu *snowflake servers*. Ansible hadir sebagai alat otomatisasi konfigurasi (*Configuration Management*) yang revolusioner karena beroperasi secara **Agentless**—ia tidak memerlukan perangkat lunak agen latar belakang yang membebani memori server target. Dokumen ini membedah arsitektur dasar Ansible, interaksi *Control Node* dengan *Managed Node*, perancangan inventaris (*inventory*) hirarkis kelas produksi, penyusunan `ansible.cfg` berkinerja tinggi, serta validasi konektivitas SSH pra-terbang.

---

## 1. Analogi Pemula: Konduktor Orkestra Simfoni & Musisi Profesional

Bagi pemula yang baru mengenal otomatisasi server, pertanyaan pertama yang kerap muncul adalah:
> *"Bagaimana mungkin Ansible bisa mengendalikan ratusan server Linux tanpa ada software Ansible yang diinstal di server-server tersebut?"*

Mari gunakan analogi pertunjukan musik orkestra simfoni:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI SISTEM AGENT-BASED vs AGENTLESS (ANSIBLE)           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PENDEKATAN LAMA: AGENT-BASED (PUPPET / CHEF) ]                           │
│  - Setiap pemain musik (Server) harus dipasangi alat "earphone pengawas"    │
│    khusus (Daemon Agent) yang menyala 24 jam.                               │
│  - Earphone ini memakan daya baterai/RAM server secara konstan.             │
│  - Masalah timbul jika versi earphone sudah usang: musisi tidak bisa        │
│    mendengarkan perintah baru sampai teknisi mendatangi kursi mereka satu   │
│    per satu untuk mengupgrade earphone tersebut.                            │
│                                                                             │
│  [ PENDEKATAN MODERN: AGENTLESS (ANSIBLE) ]                                 │
│  - Tidak ada alat pengawas khusus yang dipasang di server target.           │
│  - Konduktor Orkestra (Control Node: Laptop Mac Anda / CI Runner) cukup     │
│    mengirimkan "Kurir Lembar Partitur Lagu" (Modul Python) melalui jalur    │
│    pintu masuk resmi gedung yang sudah ada sejak zaman purba: **SSH**!      │
│  - Pemain musik (Managed Node: Ubuntu Server) membaca partitur dengan mata  │
│    dan instrumen bawaan mereka sendiri: **Python Interpreter bawaan OS**!   │
│  - Selesai lagu dimainkan, lembar partitur langsung dibakar/dibersihkan     │
│    dari meja (`/tmp/.ansible/...`), dan sang kurir SSH pamit pulang.        │
│                                                                             │
│  KEUNTUNGAN SRE:                                                            │
│  1. Server target tetap bersih (Zero Footprint).                            │
│  2. Tidak ada daemon agen yang bisa crash atau memakan CPU 100%.            │
│  3. Aman: memanfaatkan protokol keamanan standar industri (SSH Key).       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Arsitektur Control Node vs Managed Node

Sistem Ansible bekerja berdasarkan model hierarki sederhana yang terdiri dari dua pihak:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   ARSITEKTUR EKSEKUSI AGENTLESS ANSIBLE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ CONTROL NODE ] (Laptop Mac M5 ARM64 / GitLab CI Runner)                  │
│  - Menyimpan Repositori Git: Playbooks, Roles, Inventory, dan Config.       │
│  - Terinstal: `ansible`, `ansible-core`, `python3`.                         │
│  - Membaca target inventaris dan memaketkan modul Ansible menjadi skrip.   │
│                                     │                                       │
│                 SFTP / SCP (Kirim Payload Python)                           │
│                                     │                                       │
│                                     ▼ Jalur SSH (Port 22)                   │
│                                                                             │
│  [ MANAGED NODES ] (Ubuntu 22.04 LTS VM / Bare-Metal Server)                │
│  - TIDAK PERLU INSTAL ANSIBLE!                                              │
│  - Prasyarat minimal:                                                       │
│    1. OpenSSH Server aktif (`sshd.service`).                                │
│    2. Kunci publik SSH terdaftar di `~/.ssh/authorized_keys`.               │
│    3. Python 3 runtime terinstal (`/usr/bin/python3`).                      │
│    4. Hak eskalasi `sudo` tanpa password (atau via sudoers policy).         │
│                                                                             │
│  Alur di Managed Node:                                                      │
│  1. SSH menerima payload modul Ansible di direktori temporary (`/tmp/...`). │
│  2. Payload dieksekusi oleh `/usr/bin/python3`.                             │
│  3. Hasil eksekusi dikembalikan dalam format string **JSON** ke stdout.     │
│  4. File payload temporary langsung dihapus bersih secara otomatis.         │
│  5. Koneksi SSH ditutup (atau di-reuse via SSH Multiplexing).               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Anatomi Inventory Produksi: Format YAML vs INI

*Inventory* adalah buku daftar alamat dan identitas seluruh mesin yang dikelola oleh Ansible. Meskipun format INI klasik masih didukung, **standar industri SRE modern mewajibkan format YAML** karena mendukung validasi skema yang ketat (*strict schema*), struktur bersarang (*nested groups*), dan tipe data variabel yang kaya (array, boolean, dictionary).

### 3.1 Pembedaan Krusial: Hostname vs Management IP vs Data Plane IP

Kesalahan paling umum pemula adalah mencampuradukkan ketiga alamat ini:
1. `inventory_hostname` (Label Logis): Nama host unik di mata Ansible (misal: `sre-k3s-cp-01`).
2. `ansible_host` (Management IP): Alamat IP yang digunakan Ansible untuk melakukan koneksi SSH (bisa berupa IP lokal data center atau IP bridge OrbStack).
3. `k3s_advertise_ip` (Data Plane IP): Alamat IP yang digunakan antar-node Kubernetes untuk komunikasi internal cluster / VIP LoadBalancer.

### 3.2 Contoh File Inventory YAML Produksi (`inventories/lab/hosts.yml`)

Berikut adalah template inventaris standar yang memisahkan peran node secara rapi:

```yaml
---
all:
  vars:
    # Variabel global untuk seluruh host dalam inventaris
    environment_tier: "lab"
    ansible_user: "sre-admin"
    ansible_ssh_private_key_file: "~/.ssh/id_ed25519"
    ansible_python_interpreter: "/usr/bin/python3"
    
  children:
    # ---------------------------------------------------------
    # KELOMPOK CONTROL PLANE KUBERNETES K3S
    # ---------------------------------------------------------
    k3s_servers:
      hosts:
        sre-k3s-cp-01:
          ansible_host: 192.168.10.11
          node_role: "server"
          is_bootstrap_master: true
        sre-k3s-cp-02:
          ansible_host: 192.168.10.12
          node_role: "server"
          is_bootstrap_master: false
        sre-k3s-cp-03:
          ansible_host: 192.168.10.13
          node_role: "server"
          is_bootstrap_master: false

    # ---------------------------------------------------------
    # KELOMPOK WORKER / AGENT KUBERNETES K3S
    # ---------------------------------------------------------
    k3s_agents:
      hosts:
        sre-k3s-worker-01:
          ansible_host: 192.168.10.21
          node_role: "agent"
        sre-k3s-worker-02:
          ansible_host: 192.168.10.22
          node_role: "agent"

    # ---------------------------------------------------------
    # KELOMPOK GABUNGAN CLUSTER UTUH
    # ---------------------------------------------------------
    k3s_cluster:
      children:
        k3s_servers:
        k3s_agents:
```

> [!CAUTION]
> **Strict Secrets Hygiene pada Inventory:**
> DILARANG KERAS menaruh `ansible_password`, token k3s, atau raw string private key di dalam file `hosts.yml`! Gunakan selalu autentikasi berbasis kunci SSH publik (`id_ed25519`), dan simpan rahasia aplikasi di dalam **Ansible Vault** atau variabel environment CI/CD.

---

## 4. Konfigurasi Produksi: Menjinakkan `ansible.cfg`

Secara default, Ansible berjalan dengan konfigurasi lambat dan konservatif. File `ansible.cfg` yang diletakkan di root repositori proyek memungkinkan SRE menyetel parameter performa tinggi dan keamanan standar:

```ini
# ansible.cfg - Konfigurasi Tingkat Produksi Bootcamp SRE

[defaults]
# Lokasi default inventaris agar tidak perlu mengetik -i setiap saat
inventory = inventories/lab/hosts.yml

# Nonaktifkan pembuatan file .retry yang mengotori workspace saat task gagal
retry_files_enabled = false

# Matikan peringatan Python interpreter discovery untuk output yang bersih
interpreter_python = auto_silent

# Jumlah eksekusi paralel (default 5, naikkan ke 10-20 untuk data center besar)
forks = 10

# Format output konsol yang mudah dibaca manusia (YAML callback)
stdout_callback = yaml
bin_ansible_callbacks = True

# Pengelolaan Host Key SSH (Di lab/CI bisa false, di prod ketat true)
host_key_checking = true

# Timeout koneksi SSH dalam detik
timeout = 30

[privilege_escalation]
# Konfigurasi eskalasi root (sudo)
become = true
become_method = sudo
become_user = root
become_ask_pass = false

[ssh_connection]
# AKSELERASI PERFORMA TINGGI (SRE ESSENTIAL):
# Pipelining memangkas transfer file modul temporary via SFTP,
# langsung mengeksekusinya via stdin SSH. Meningkatkan kecepatan 3x-4x lipat!
pipelining = true

# Optimasi koneksi SSH ControlMaster (Multiplexing)
# Menggunakan kembali satu sesi socket SSH untuk banyak task berturut-turut
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o PreferredAuthentications=publickey
```

---

## 5. Hands-on Step-by-Step & SSH Preflight

Mari mempraktikkan langkah operasional untuk memverifikasi kesiapan kontrol node ke managed node di lingkungan lokal OrbStack macOS.

### Langkah 1: Buat Kunci SSH Modern (Ed25519)
Di mesin Control Node (Mac Anda), buat pasangan kunci SSH khusus automasi:
```bash
# Membuat kunci SSH Ed25519 tanpa passphrase untuk CI/automation
ssh-keygen -t ed25519 -C "ansible-automation@bootcamp.local" -f ~/.ssh/id_ed25519 -N ""
```

### Langkah 2: Distribusikan Kunci Publik ke Server Target
Salin kunci publik ke mesin virtual (misalnya VM OrbStack yang dibuat di Fase 3):
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub sre-admin@192.168.10.11
ssh-copy-id -i ~/.ssh/id_ed25519.pub sre-admin@192.168.10.12
```

### Langkah 3: Periksa Struktur Grafik Inventaris
Pastikan Ansible berhasil mem-parsing file YAML inventaris dengan benar:
```bash
ansible-inventory --graph
```
*Output yang Diharapkan:*
```text
@all:
  |--@k3s_cluster:
  |  |--@k3s_servers:
  |  |  |--sre-k3s-cp-01
  |  |  |--sre-k3s-cp-02
  |  |  |--sre-k3s-cp-03
  |  |--@k3s_agents:
  |  |  |--sre-k3s-worker-01
  |  |  |--sre-k3s-worker-02
  |--@ungrouped:
```

### Langkah 4: Uji Konektivitas Preflight dengan Ping Module
Modul `ansible.builtin.ping` bukan sekadar `ping ICMP`, melainkan memverifikasi koneksi SSH, autentikasi user, eksekusi Python, dan penerimaan output JSON:
```bash
ansible k3s_servers -m ping
```
*Output yang Diharapkan:*
```text
sre-k3s-cp-01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
sre-k3s-cp-02 | SUCCESS => { ... "ping": "pong" }
sre-k3s-cp-03 | SUCCESS => { ... "ping": "pong" }
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

Berikut adalah 4 masalah nyata koneksi SSH Ansible yang paling sering dialami di lapangan:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN KONEKSI ANSIBLE                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "Host key verification failed" (UNREACHABLE) ]                  │
│  Gejala: Ansible menolak koneksi dengan pesan:                              │
│          "fatal: [sre-k3s-cp-01]: UNREACHABLE! => Host key verification..." │
│  Akar Masalah:                                                              │
│  - Fingerprint SSH server target berubah karena VM baru saja di-recreate    │
│    oleh OpenTofu, tetapi fingerprint lama masih tersimpan di `known_hosts`. │
│  Solusi SRE:                                                                │
│  1. Hapus fingerprint lama pada control node:                               │
│     ssh-keygen -R 192.168.10.11                                             │
│  2. Di lingkungan disposable lab/CI, set di ansible.cfg:                   │
│     host_key_checking = false                                               │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "Permission denied (publickey)" ]                               │
│  Gejala: Ansible tidak dapat login ke managed node.                         │
│  Akar Masalah:                                                              │
│  - File permission pada target terlalu longgar (`chmod 777 ~/.ssh`), atau   │
│  - Kunci publik belum masuk ke `/home/sre-admin/.ssh/authorized_keys`.      │
│  Solusi SRE:                                                                │
│  1. Login manual dengan debug verbose: `ssh -vvv sre-admin@192.168.10.11`   │
│  2. Perbaiki permission di target:                                          │
│     chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys                    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: "sudo: a password is required" (Privilege Escalation Gagal) ]   │
│  Gejala: Modul ping sukses, tetapi task yang butuh root gagal saat become.  │
│  Akar Masalah:                                                              │
│  - User `sre-admin` belum diberikan hak `NOPASSWD` di file sudoers.         │
│  Solusi SRE:                                                                │
│  1. Masukkan konfigurasi sudoers tanpa password untuk user otomatisasi:     │
│     echo "sre-admin ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/sre-admin      │
│     chmod 0440 /etc/sudoers.d/sre-admin                                     │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 4: "/usr/bin/python3: not found" ]                                 │
│  Gejala: SSH berhasil tersambung, tetapi muncul error binary Python hilang. │
│  Akar Masalah:                                                              │
│  - Base image Linux yang digunakan adalah minimal/cloud image tanpa Python. │
│  Solusi SRE:                                                                │
│  1. Manfaatkan modul bawaan `raw` untuk bootstrap Python pertama kali:      │
│     ansible k3s_cluster -m raw -a "apt-get update && apt-get install -y     │
│     python3 python3-apt" --become                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menampilkan seluruh host dan hierarki grup inventaris
ansible-inventory -i inventories/lab/hosts.yml --graph

# 2. Mengecek variabel yang terikat pada satu host tertentu
ansible-inventory -i inventories/lab/hosts.yml --host sre-k3s-cp-01

# 3. Preflight ping ke seluruh node dengan pembatasan konkurensi 5 server
ansible all -m ping -f 5

# 4. Mengecek konektivitas hanya pada satu grup atau host tertentu (--limit)
ansible k3s_servers -m ping --limit sre-k3s-cp-01

# 5. Menjalankan perintah shell cepat untuk memeriksa versi kernel OS
ansible all -m command -a "uname -r"

# 6. Menjalankan pemeriksaan memori untuk memastikan swap nonaktif
ansible all -m command -a "free -h"
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa arsitektur *Agentless* pada Ansible dianggap sebagai keunggulan operasional yang signifikan di lingkungan *on-premises* dan private data center?
- A. Karena server target tidak perlu memiliki koneksi jaringan sama sekali.
- B. Karena tidak ada perangkat lunak agen latar belakang (*background daemon*) yang harus dipelihara, diupgrade, atau memakan siklus CPU/RAM pada server target.
- C. Karena Ansible otomatis mengompilasi kode program menjadi file binary C++.
- D. Karena Ansible tidak memerlukan sistem operasi pada target node.

*Kunci Jawaban:* **B**. Pendekatan agentless mengeliminasi beban pemeliharaan agen pihak ketiga (*agent maintenance overhead*) dan menjaga server target tetap bersih (*zero-footprint*).

### Soal 2 (Pilihan Ganda)
Dalam konfigurasi `ansible.cfg`, parameter `pipelining = true` diaktifkan oleh tim SRE. Apa dampak langsung dari konfigurasi ini terhadap eksekusi Ansible?
- A. Ansible akan mengabaikan seluruh error yang terjadi pada playbook.
- B. Ansible akan mengirimkan modul langsung via pipa *stdin* SSH tanpa harus membuat dan menyalin file temporary ke disk managed node via SFTP, mempercepat eksekusi 3–4 kali lipat.
- C. Ansible otomatis mematikan firewall server target.
- D. Ansible akan mendistribusikan beban kerja secara bergantian menggunakan antrean FIFO.

*Kunci Jawaban:* **B**. Pipelining mengurangi jumlah operasi koneksi SFTP/SCP bolak-balik dengan mengeksekusi payload Python langsung di memori melalui stdin SSH.

### Soal 3 (Pilihan Ganda)
Dalam file inventory, Anda mendefinisikan:
```yaml
sre-k3s-cp-01:
  ansible_host: 10.0.0.15
```
Apa arti dari `sre-k3s-cp-01` dan `ansible_host`?
- A. `sre-k3s-cp-01` adalah nama user SSH, sedangkan `ansible_host` adalah nama grup server.
- B. `sre-k3s-cp-01` adalah identifier logis unik (*inventory_hostname*), sedangkan `ansible_host` adalah alamat IP fisik/jaringan yang digunakan Ansible untuk membuka sesi SSH.
- C. Keduanya wajib bernilai sama persis atau Ansible akan melempar *syntax error*.
- D. `ansible_host` adalah gateway router data center.

*Kunci Jawaban:* **B**. Pemisahan label logis dari alamat fisik memungkinkan SRE mengganti alamat IP atau melakukan migrasi server tanpa mengubah referensi target di dalam playbook.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Di lingkungan data center perusahaan, Anda ditugaskan mengotomasi 50 server bare-metal baru menggunakan Ansible. Saat menjalankan `ansible all -m ping`, 15 server melempar error `sudo: a password is required`, sementara 35 server lainnya berhasil. Analisis apa penyebab perbedaan ini dan bagaimana cara menyelesaikannya secara aman dan baku!

*Panduan Solusi:*
1. **Akar Masalah:**
   15 server tersebut belum memiliki konfigurasi sudoers yang mengizinkan user otomatisasi (`sre-admin`) mengeksekusi perintah dengan hak `root` tanpa prompt password interaktif (`NOPASSWD`). Ansible dirancang untuk berjalan non-interaktif, sehingga ketika prompt password muncul, proses eksekusi akan macet atau gagal.
2. **Langkah Perbaikan Baku:**
   - Masukkan konfigurasi privilege escalation yang tepat pada file `/etc/sudoers.d/sre-admin` di 15 server tersebut:
     ```text
     sre-admin ALL=(ALL) NOPASSWD: ALL
     ```
   - Pastikan izin berkas dikunci secara ketat dengan `chmod 0440 /etc/sudoers.d/sre-admin` dan diverifikasi sintaksnya menggunakan utilitas `visudo -c`.
   - Untuk automasi pertama kali jika password sudo diketahui, eksekusi sementara dengan flag `-K` atau `--ask-become-pass`:
     ```bash
     ansible all -m copy -a "content='sre-admin ALL=(ALL) NOPASSWD: ALL\n' dest=/etc/sudoers.d/sre-admin mode=0440" --become --ask-become-pass
     ```

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Fase 3: 03 — OpenTofu, Ansible, dan k3s Handoff](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/03-opentofu-ansible-k3s-handoff.md) — Memahami bagaimana metadata VM yang diprovisioning OpenTofu diserahkan ke inventaris Ansible.
- **Topik Berikutnya:**
  - [02 — Ad-hoc Command dan Playbook YAML](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/02-ad-hoc-playbook-yaml.md) — Menulis playbook terstruktur dengan modul bawaan deklaratif.
  - [03 — Idempotency, Variables, Facts, dan Handlers](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/03-idempotency-variables-facts-handlers.md) — Menjamin konvergensi sistem dan penanganan event handler yang aman.
