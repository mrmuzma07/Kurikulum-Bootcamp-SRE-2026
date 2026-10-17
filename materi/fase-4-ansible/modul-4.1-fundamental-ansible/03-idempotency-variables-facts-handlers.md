# 03 — Idempotency, Variables, Facts, dan Handlers: Menjamin Konvergensi Sistem dan Siklus Hidup Event

> **Ringkasan:** Dua pilar utama yang membedakan otomatisasi profesional (*Site Reliability Engineering*) dari sekadar skrip Bash amatir adalah **Idempotency** (kemampuan dijalankan berulang kali tanpa mengubah hasil akhir) dan **Event Decoupling** (memisahkan modifikasi konfigurasi dari pemicuan reload layanan). Kesalahan dalam memahami siklus hidup fakta sistem (*facts*), presedensi variabel, atau mekanisme pemicuan handler dapat memicu badai restart layanan (*service restart storms*) yang melumpuhkan sistem produksi. Dokumen ini membedah prinsip konvergensi idempoten, mengurai hierarki variabel, mengoptimalkan pengumpulan fakta sistem, serta menguasai siklus hidup handler tingkat lanjut.

---

## 1. Analogi Pemula: Sakelar Lampu Cerdas vs Sakelar Tarik Tali

Untuk memahami mengapa *Idempotency* adalah hukum tertinggi dalam otomatisasi infrastruktur, bayangkan dua jenis sakelar lampu di rumah Anda:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI IDEMPOTEN vs NON-IDEMPOTEN                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ SAKELAR TARIK TALI (NON-IDEMPOTEN / SKRIP BASH AMATIR) ]                 │
│  - Setiap kali tali ditarik, keadaan lampu akan berbalik (Toggle):          │
│    Tarik 1: Lampu Menyala.                                                  │
│    Tarik 2: Lampu Mati.                                                     │
│  - Masalah: Jika Anda berada di luar ruangan dan tidak bisa melihat lampu,  │
│    lalu Anda menyuruh asisten: "Tolong tarik talinya agar lampu menyala!".  │
│    Jika ternyata lampu *sudah menyala*, tarikan asisten justru MEMATIKAN    │
│    lampu tersebut!                                                          │
│  - Contoh Nyata di Bash: `echo "10.0.0.5 master" >> /etc/hosts`.            │
│    Jika skrip dijalankan 10 kali, file `/etc/hosts` akan memiliki 10 baris  │
│    duplikat yang sama persis (Kotor & Merusak Sistem!).                     │
│                                                                             │
│  [ SAKELAR CERDAS BERLABEL (IDEMPOTEN / ANSIBLE NATIVE) ]                   │
│  - Memiliki tombol khusus bertuliskan: **SET TO ON** dan **SET TO OFF**.    │
│  - Jika Anda menekan tombol "SET TO ON":                                    │
│    - Jika lampu MATI: Sakelar mengubah keadaan -> Lampu MENYALA (`changed`). │
│    - Jika lampu SUDAH MENYALA: Sakelar TIDAK MELAKUKAN APA PUN (`ok`).      │
│  - Anda bisa menekan tombol tersebut 100 kali sehari, dan hasilnya DIJAMIN │
│    tetap sama: Lampu menyala tanpa lonjakan listrik dan tanpa efek samping! │
│                                                                             │
│  RUMUS MATEMATIS SRE:                                                       │
│                           f(f(x)) = f(x)                                    │
│  Menerapkan fungsi f dua kali pada x menghasilkan nilai yang sama dengan    │
│  menerapkannya satu kali.                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Prinsip Konvergensi Idempoten (System Convergence)

Dalam rekayasa keandalan sistem (*SRE*), tujuan utama playbook adalah membawa keadaan sistem aktual (*current state*) menuju keadaan yang dideklarasikan (*desired state*):

```text
Keadaan Awal Sistem             Dijalankan Playbook                  Keadaan Akhir Sistem
───────────────────             ───────────────────                  ────────────────────
Server Kosong (Baru)    ───>    Run 1 (Apply Config)     ───>        Layanan Aktif & Hardened
                                (Hasil: changed = 12)

Layanan Aktif           ───>    Run 2 (Verifikasi Ulang) ───>        Layanan Tetap Aktif
                                (Hasil: changed = 0)                 (Sistem Konvergen!)

Layanan Diotak-atik     ───>    Run 3 (Auto-Heal / Drift)───>        Layanan Kembali Standar
Secara Manual                   (Hasil: changed = 1)                 (Drift Tereleminasi!)
```

> [!IMPORTANT]
> **Peringatan SRE: Arti dari `changed=0`:**
> Status `changed=0` hanya membuktikan bahwa kondisi file, paket, dan status systemd cocok dengan apa yang Anda tulis di playbook. Status ini **BUKAN BUKTI** bahwa aplikasi di dalamnya bebas bug atau mampu melayani request pelanggan. SRE tetap wajib menjalankan *smoke-test* atau pemeriksaan *health-check probe* terpisah!

---

## 3. Hierarki & Presedensi Variabel Ansible

Ansible memiliki 22 tingkatan prioritas variabel. Bagi seorang SRE di lapangan, Anda cukup menguasai **4 Tingkatan Inti** yang paling sering digunakan dalam arsitektur produksi:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 HIERARKI VARIABEL PRODUKSI (DARI LEMAH KE KUAT)             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PRIORITAS 1: ROLE DEFAULTS ] (Paling Lemah)                              │
│  Lokasi: `roles/<role_name>/defaults/main.yml`                              │
│  Fungsi: Nilai bawaan yang aman (fallback value). Mudah ditimpa.             │
│  Contoh: `chrony_ntp_server: "pool.ntp.org"`                                │
│                                     │                                       │
│                                     ▼ Ditimpa oleh                          │
│                                                                             │
│  [ PRIORITAS 2: GROUP VARIABLES ]                                           │
│  Lokasi: `group_vars/all.yml` atau `group_vars/k3s_servers.yml`              │
│  Fungsi: Pengaturan spesifik lingkungan (Lab vs Staging vs Production).     │
│  Contoh: `chrony_ntp_server: "192.168.10.1"` (NTP lokal data center).       │
│                                     │                                       │
│                                     ▼ Ditimpa oleh                          │
│                                                                             │
│  [ PRIORITAS 3: HOST VARIABLES ]                                            │
│  Lokasi: `host_vars/sre-k3s-cp-01.yml`                                      │
│  Fungsi: Parameter unik per mesin (misal: IP statis unik, disk path).       │
│  Contoh: `k3s_node_ip: "192.168.10.11"`                                     │
│                                     │                                       │
│                                     ▼ Ditimpa oleh                          │
│                                                                             │
│  [ PRIORITAS 4: EXTRA VARS VIA CLI ] (Paling Kuat / Tak Terkalahkan)        │
│  Sintaks: `ansible-playbook ... -e "force_reboot=true"`                     │
│  Fungsi: Variabel darurat atau parameter injeksi dinamis dari pipeline CI. │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Praktik Terbaik Penamaan Variabel (Namespace Rule)
Jangan pernah memberi nama variabel generik seperti `port: 80` atau `user: admin`. Selalu gunakan prefix nama role/domain untuk mencegah bentrokan (*variable collision*):
```yaml
# BENAR (Scoped & Explicit):
k3s_server_apiserver_port: 6443
k3s_cluster_cidr: "10.42.0.0/16"
chrony_service_enabled: true

# SALAH (Beresiko tinggi saling menimpa):
port: 6443
cidr: "10.42.0.0/16"
enabled: true
```

---

## 4. Menggali Fakta Sistem (Ansible Facts & Setup Module)

Saat sebuah Playbook mulai berjalan, langkah pertama yang otomatis dieksekusi adalah `Gathering Facts`. Ansible mengeksekusi modul `ansible.builtin.setup` untuk memindai seluruh data hardware, sistem operasi, dan antarmuka jaringan pada managed node.

### 4.1 Fakta-fakta Kunci bagi SRE On-Premises

Fakta tersimpan dalam struktur dictionary `ansible_facts`:

```yaml
# 1. Mendeteksi Keluarga Sistem Operasi (Debian vs RedHat)
when: ansible_facts.os_family == "Debian"

# 2. Mendeteksi Arsitektur Hardware CPU (ARM64 Apple Silicon / Pi vs AMD64 Server)
when: ansible_facts.architecture == "aarch64" # atau "x86_64"

# 3. Mendeteksi Alamat IP Default Gateway
k3s_advertise_address: "{{ ansible_facts.default_ipv4.address }}"

# 4. Mendeteksi Alokasi Memori Fisik (RAM) untuk Sizing Pod
when: ansible_facts.memtotal_mb < 2048 # Peringatkan jika RAM kurang dari 2GB!

# 5. Mendeteksi Sistem Inisialisasi
when: ansible_facts.service_mgr == "systemd"
```

### 4.2 Mengoptimalkan Kinerja: Fact Gathering Subset
Di data center dengan ribuan node atau server dengan puluhan NFS mount, `gather_facts` bisa memakan waktu bermenit-menit. Optimalkan dengan mengambil subset yang diperlukan saja:

```yaml
- name: Playbook ringkas berkinerja tinggi
  hosts: all
  gather_facts: true
  # HANYA kumpulkan fakta jaringan dan hardware dasar, abaikan fakta virtualisasi/mount
  gather_subset:
    - "!all"
    - "!min"
    - "network"
    - "hardware"
```
Jika playbook Anda sama sekali tidak membutuhkan fakta sistem, matikan total untuk eksekusi instan: `gather_facts: false`.

---

## 5. Siklus Hidup Handlers: Mencegah Restart Storms

Salah satu fitur paling elegan pada Ansible adalah pemisahan antara **deteksi perubahan** dan **eksekusi aksi sampingan** (*side effects*).

### 5.1 Masalah Badai Restart (Restart Storms)
Bayangkan Anda memiliki 4 task berturut-turut yang mengubah konfigurasi NGINX:
1. Mengubah file `nginx.conf`
2. Mengubah SSL Certificate
3. Mengubah MIME Types
4. Menambahkan Upstream backend

Jika setiap task langsung mengeksekusi perintah `systemctl restart nginx`, server NGINX akan direstart **4 KALI BERTURUT-TURUT** dalam rentang 10 detik! Ini memicu *outage* sesaat bagi ribuan pengguna aktif.

Dengan **Handlers**, Ansible mencatat bahwa ada notifikasi perubahan, namun menahan eksekusinya sampai seluruh task dalam Play selesai. NGINX hanya akan direstart **TEPAT 1 KALI DI AKHIR PLAY**!

```text
Task 1 (Edit nginx.conf)      ──> Berubah? YA  ──> Catat sinyal: notify: "Reload Nginx"
Task 2 (Update SSL cert)      ──> Berubah? YA  ──> Sinyal sama: "Reload Nginx" (Deduplikasi!)
Task 3 (Update MIME types)    ──> Berubah? TIDAK
Task 4 (Update Upstream)      ──> Berubah? YA  ──> Sinyal sama: "Reload Nginx" (Deduplikasi!)
                                       │
                                       ▼ Play Selesai
                     EKSEKUSI HANDLER: "Reload Nginx" (Hanya 1x!)
```

### 5.2 Teknik SRE: Memaksa Eksekusi Handler Seketika (`flush_handlers`)
Terkadang, sebuah service (misal: Chrony/NTP) harus segera aktif sebelum task berikutnya berjalan (karena task berikutnya adalah inisialisasi sertifikat TLS yang butuh jam tersinkron). Gunakan meta task:

```yaml
    - name: Konfigurasikan Chrony NTP server
      ansible.builtin.template:
        src: chrony.conf.j2
        dest: /etc/chrony/chrony.conf
      notify: Restart Chrony Service

    - name: Paksa eksekusi handler SEKARANG JUGA tanpa menunggu akhir play!
      ansible.builtin.meta: flush_handlers

    - name: Task berikutnya berjalan dengan jam yang sudah 100% akurat
      ansible.builtin.command: /opt/scripts/generate-tls-certs.sh
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA IDEMPOTENCY & HANDLERS                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: SKRIP BASH MERUSAK IDEMPOTENCY (ALWAYS CHANGED) ]               │
│  Gejala: Playbook dijalankan kedua kali, namun hasil masih `changed=2`.      │
│  Akar Masalah:                                                              │
│  - Menggunakan modul `command` atau `shell` tanpa `creates`, `removes`,     │
│    atau `changed_when`.                                                     │
│  Solusi SRE:                                                                │
│  Gunakan atribut `creates` agar task dilewati jika file target sudah ada:   │
│  - name: Download k3s binary installer                                      │
│    ansible.builtin.get_url:                                                 │
│      url: https://get.k3s.io                                                │
│      dest: /tmp/install-k3s.sh                                              │
│      mode: '0755'                                                           │
│  - name: Jalankan installer HANYA jika k3s belum pernah terpasang           │
│    ansible.builtin.command: /tmp/install-k3s.sh                             │
│    args:                                                                    │
│      creates: /usr/local/bin/k3s # Idempotency Guard!                       │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: GATHER_FACTS MENGGANTUNG (HANG) / TIMEOUT ]                     │
│  Gejala: Playbook terhenti lama pada task `[Gathering Facts]`.               │
│  Akar Masalah:                                                              │
│  - Terdapat mount point NFS atau network share yang sedang *stale* / putus.  │
│    Modul setup mencoba membaca `df` pada mount point tersebut lalu hang.    │
│  Solusi SRE:                                                                │
│  1. Batasi fact gathering di ansible.cfg:                                   │
│     [defaults]                                                              │
│     gather_timeout = 10                                                     │
│  2. Abaikan pengumpulan informasi hardware disk mount:                      │
│     gather_subset: ["!hardware", "network"]                                 │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: HANDLER BATAL BERJALAN KARENA TASK LAIN GAGAL ]                 │
│  Gejala: File konfigurasi berhasil diperbarui, tetapi di task nomor 5 ada   │
│          error. Playbook berhenti dan service dibiarkan dalam kondisi lama. │
│  Akar Masalah:                                                              │
│  - Secara default, jika terjadi fatal error pada sebuah task, Ansible       │
│    TIDAK AKAN menjalankan handler apa pun.                                  │
│  Solusi SRE:                                                                │
│  Tambahkan opsi `force_handlers: true` pada tingkat Play:                   │
│  - name: Setup sistem misi kritis                                           │
│    hosts: k3s_cluster                                                       │
│    force_handlers: true # Tetap eksekusi handler yang sudah terpicu!        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menampilkan seluruh fakta mentah yang dikumpulkan dari sebuah server
ansible sre-k3s-cp-01 -m setup

# 2. Memfilter fakta spesifik (misal: hanya informasi memori RAM)
ansible sre-k3s-cp-01 -m setup -a "filter=ansible_memtotal_mb"

# 3. Memfilter fakta antarmuka jaringan IPv4
ansible sre-k3s-cp-01 -m setup -a "filter=ansible_default_ipv4"

# 4. Menguji idempotensi: Jalankan playbook 2 kali berturut-turut
# Run 1: Boleh ada changed
ansible-playbook playbooks/site.yml
# Run 2: Wajib changed=0 (Konvergen 100%)
ansible-playbook playbooks/site.yml

# 5. Menimpa variabel via Extra Vars untuk pengujian canary
ansible-playbook playbooks/site.yml -e "k3s_version=v1.31.1+k3s1 debug_mode=true"
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Dalam terminologi Site Reliability Engineering (SRE), apa makna dari status `changed: 0` pada eksekusi kedua dari sebuah Ansible Playbook?
- A. Seluruh server target telah mati dan tidak merespons koneksi.
- B. Playbook telah berhasil membawa sistem ke kondisi konvergen (*desired state* sama persis dengan *current state*), sehingga tidak ada mutasi yang perlu dilakukan.
- C. Ansible gagal mem-parsing file YAML inventaris.
- D. Seluruh aplikasi di dalam cluster k3s bebas dari memory leak.

*Kunci Jawaban:* **B**. `changed: 0` menandakan kondisi konvergen sempurna di mana infrastruktur aktual sudah sesuai dengan deklarasi kode tanpa perlu intervensi.

### Soal 2 (Pilihan Ganda)
Jika sebuah variabel didefinisikan dengan nilai berbeda pada `roles/common/defaults/main.yml`, `group_vars/all.yml`, dan argumen CLI `-e "my_var=nilai_x"`, nilai manakah yang akan dimenangkan dan digunakan oleh Ansible?
- A. Nilai dari `roles/common/defaults/main.yml`.
- B. Nilai dari `group_vars/all.yml`.
- C. Nilai dari argumen CLI `-e "my_var=nilai_x"` (*Extra Vars*).
- D. Ansible akan berhenti dengan pesan *variable conflict error*.

*Kunci Jawaban:* **C**. Extra Vars via CLI (`-e`) menduduki hierarki presedensi variabel nomor 1 (paling tinggi) dan akan menimpa seluruh variabel lainnya.

### Soal 3 (Pilihan Ganda)
Sebuah Playbook memiliki 3 task yang masing-masing memanggil `notify: Restart Kubelet`. Jika ketiga task tersebut semuanya menghasilkan status `changed: true`, berapa kali handler `Restart Kubelet` akan dieksekusi oleh Ansible?
- A. 3 kali (setiap kali task selesai).
- B. Tepat 1 kali di akhir Play, karena Ansible melakukan deduplikasi sinyal notifikasi handler.
- C. 0 kali, karena notifikasi ganda membatalkan handler.
- D. Tergantung jumlah core CPU control node.

*Kunci Jawaban:* **B**. Handler mengimplementasikan deduplikasi event sehingga hanya dijalankan sekali di akhir Play, mencegah terjadinya *restart storm*.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Anda sedang menulis task untuk mengunduh dan mengekstrak tarball biner CLI `kubectl` versi `v1.31.0` ke dalam direktori `/usr/local/bin/`. Seorang junior SRE menulis task berikut:
```yaml
- name: Install kubectl
  ansible.builtin.shell: |
    curl -LO https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
Identifikasi 2 kelemahan fatal dari task di atas terkait *idempotency* dan arsitektur hardware, lalu tulis ulang task tersebut menggunakan modul deklaratif yang idempoten dan adaptif!

*Panduan Solusi:*
1. **Identifikasi Kelemahan:**
   - **Melanggar Idempotency:** Modul `shell` akan selalu mengunduh file biner dan mengekstraknya ulang pada setiap kali playbook dijalankan (`changed: true`), memboroskan bandwidth jaringan dan waktu eksekusi.
   - **Hardcoded Arsitektur CPU:** URL memuat string statis `amd64`. Jika task dijalankan pada node ARM64 (seperti MacBook Apple Silicon atau Raspberry Pi), biner kubectl akan *crash* saat dieksekusi dengan error `Exec format error`!
2. **Penulisan Ulang Standar SRE:**
   Gunakan modul `ansible.builtin.get_url` dengan pemetaan arsitektur dinamis via *facts*:
   ```yaml
   - name: Pasang biner kubectl secara idempoten dan multi-arch
     ansible.builtin.get_url:
       url: "https://dl.k8s.io/release/v1.31.0/bin/linux/{{ 'arm64' if ansible_facts.architecture == 'aarch64' else 'amd64' }}/kubectl"
       dest: /usr/local/bin/kubectl
       owner: root
       group: root
       mode: '0755'
       checksum: "sha256:https://dl.k8s.io/release/v1.31.0/bin/linux/{{ 'arm64' if ansible_facts.architecture == 'aarch64' else 'amd64' }}/kubectl.sha256"
   ```
   *Keunggulan:* Modul `get_url` akan mengecek checksum biner yang sudah ada di `/usr/local/bin/kubectl`. Jika versinya sudah cocok, modul akan mengembalikan status `ok` tanpa mengunduh ulang!

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Arsitektur Ansible, Inventory, dan SSH](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/01-arsitektur-inventory-ssh.md) — Memahami konektivitas kontrol node ke managed node.
  - [02 — Ad-hoc Command dan Playbook YAML](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/02-ad-hoc-playbook-yaml.md) — Menulis sintaks playbook dasar.
- **Praktik Laboratorium:**
  - [LAB-01 — Inventory, SSH, dan Ad-hoc](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/lab/LAB-01-inventory-ssh-ad-hoc.md) — Praktik setup inventory dan pengujian koneksi.
  - [LAB-02 — Playbook Common Idempotent](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.1-fundamental-ansible/lab/LAB-02-playbook-common-idempotent.md) — Membuktikan konvergensi `changed: 0` pada rerun.
- **Tahap Berikutnya:**
  - [Modul 4.2 — Pola Produksi Ansible](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/) — Mengorganisasi kode ke dalam Roles, Jinja2 Templating, dan Ansible Vault.
