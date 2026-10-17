# 02 — Role Instalasi dan Upgrade k3s Multi-Node HA: Mengorkestrasi Control Plane dan Quorum Etcd

> **Ringkasan:** Memasang klaster Kubernetes satu node (*single-node*) sangat mudah, namun membangun arsitektur k3s *High Availability* (HA) multi-node dengan basis data terdistribusi *embedded etcd* di lingkungan *on-premises* menuntut disiplin urutan eksekusi (*deterministic sequencing*) yang sangat ketat. Pelanggaran urutan dapat memecah klaster menjadi beberapa klaster independen (*split-brain*) atau merusak kuorum basis data. Dokumen ini membedah arsitektur kuorum etcd, merancang kontrak peran (*role contract*) Ansible untuk server dan agen, menetapkan urutan bootstrap bertahap, serta menyajikan prosedur *rolling upgrade* versi Kubernetes tanpa menyebabkan *downtime* pada aplikasi.

---

## 1. Analogi Pemula: Sidang Parlemen Tiga Anggota & Para Pekerja Pabrik

Bagi pemula, perbedaan mendasar antara *k3s server* (control-plane) dan *k3s agent* (worker) sering membingungkan:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI SIDANG PARLEMEN (CONTROL PLANE) & PEKERJA           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ ANGGOTA DEWAN PARLEMEN: 3 K3S SERVERS DENGAN EMBEDDED ETCD ]             │
│  - Parlemen membutuhkan aturan kuorum mayoritas:                            │
│                         Kuorum = (N / 2) + 1                                │
│    Untuk 3 anggota dewan, kuorum minimal = (3/2) + 1 = **2 Anggota Sah**!   │
│                                                                             │
│  - HUKUM KELAHIRAN PARLEMEN (BOOTSTRAP SEQUENCING):                         │
│    1. Anggota Pertama (sre-k3s-cp-01) datang pagi-pagi, membuka pintu       │
│       ruang sidang, dan menyalakan mikrofon (`cluster-init: true`).         │
│    2. Anggota Kedua (sre-k3s-cp-02) datang, mendaftar ke Anggota Pertama,   │
│       dan duduk di kursi (`server: https://cp-01:6443`).                    │
│       SEKETIKA KUORUM TERCAPAI (2 dari 3)! Sidang resmi dibuka!             │
│    3. Anggota Ketiga (sre-k3s-cp-03) bergabung melengkapi toleransi bencana │
│       (Jika 1 anggota pingsan, 2 anggota lainnya tetap bisa bersidang!).    │
│                                                                             │
│  - BAHAYA SPLIT-BRAIN JIKA SALAH URUTAN:                                    │
│    Jika Anda menyuruh ketiga anggota berteriak `cluster-init: true` secara  │
│    bersamaan, mereka akan membentuk 3 ruang sidang terpisah yang saling     │
│    tidak kenal dan saling menolak!                                          │
│                                                                             │
│  [ PEKERJA PABRIK: K3S AGENTS (WORKER NODES) ]                              │
│  - Pekerja tidak ikut rapat dan tidak punya hak suara dalam voting etcd.   │
│  - Tugas pekerja: Mendaftar ke ketua dewan, menerima instruksi kerja        │
│    (Deploy Pod kontainer), dan melaporkan status kesehatan mesin.           │
│  - Jika pekerja mati, dewan cukup memindahkan pekerjaan ke pekerja lain.    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Topologi Produksi On-Premises & Arsitektur Komponen

Di lingkungan *on-premises*, klaster dirancang tanpa ketergantungan pada cloud provider luar:

```text
                               ┌──────────────────────────┐
                               │ Alamat VIP Load Balancer │
                               │  https://192.168.10.10   │
                               │        (Port 6443)       │
                               └─────────────┬────────────┘
                                             │
                   ┌─────────────────────────┼─────────────────────────┐
                   │                         │                         │
                   ▼                         ▼                         ▼
         ┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐
         │   sre-k3s-cp-01   │◄───►│   sre-k3s-cp-02   │◄───►│   sre-k3s-cp-03   │
         │ (Bootstrap Master)│     │  (Joined Master)  │     │  (Joined Master)  │
         │   Etcd Quorum 1   │     │   Etcd Quorum 2   │     │   Etcd Quorum 3   │
         └───────────────────┘     └───────────────────┘     └───────────────────┘
                   │                                                   │
                   └─────────────────────────┬─────────────────────────┘
                                             │ Flannel VXLAN (UDP 8472)
                                             ▼
                               ┌───────────────────────────┐
                               │       K3S AGENTS          │
                               │  sre-k3s-worker-01        │
                               │  sre-k3s-worker-02        │
                               └───────────────────────────┘
```

### 2.1 Keputusan Komponen Bawaan k3s
Sesuai rancangan kurikulum Bootcamp SRE, dua komponen bawaan k3s dinonaktifkan:
- **`servicelb` dinonaktifkan:** Karena di on-prem kita menggunakan **MetalLB L2/BGP** yang memberikan IP LoadBalancer sungguhan pada subnet fisik data center.
- **`traefik` dinonaktifkan:** Karena kita akan memasang Ingress Controller pilihan industri (seperti Ingress NGINX) yang dipaketkan via Helm di Fase 5.

---

## 3. Kontrak Role Ansible `k3s_server` & `k3s_agent`

### 3.1 Berkas Variabel Bawaan (`roles/k3s_server/defaults/main.yml`)

```yaml
---
# Parameter Konfigurasi Role k3s Server (Control Plane)
k3s_version: "v1.31.1+k3s1"

# Endpoint VIP atau DNS API Server
k3s_api_vip: "192.168.10.10"
k3s_api_port: 6443

# Jaringan Pod dan Service Internal
k3s_cluster_cidr: "10.42.0.0/16"
k3s_service_cidr: "10.43.0.0/16"
k3s_cluster_dns: "10.43.0.10"

# Opsi Instalasi Ekstra
k3s_disable_components:
  - servicelb
  - traefik

# Pengaturan Hak Akses Kubeconfig
k3s_kubeconfig_mode: "0644"
```

### 3.2 Template Konfigurasi Dinamis (`roles/k3s_server/templates/config.yaml.j2`)

k3s membaca konfigurasi startup dari berkas `/etc/rancher/k3s/config.yaml`:

```yaml
{# Konfigurasi Deklaratif k3s Server #}
write-kubeconfig-mode: "{{ k3s_kubeconfig_mode }}"
node-ip: "{{ k3s_node_ip }}"
tls-san:
  - "{{ k3s_api_vip }}"
  - "{{ k3s_node_ip }}"
  - "{{ inventory_hostname }}"

cluster-cidr: "{{ k3s_cluster_cidr }}"
service-cidr: "{{ k3s_service_cidr }}"
cluster-dns: "{{ k3s_cluster_dns }}"

disable:
{% for comp in k3s_disable_components %}
  - {{ comp }}
{% endfor %}

{% if is_bootstrap_master %}
# Inisialisasi Master Pertama (Embedded Etcd)
cluster-init: true
token: "{{ vault_k3s_cluster_token }}"
{% else %}
# Master Tambahan Bergabung ke Master Pertama
server: "https://{{ hostvars[groups['k3s_servers'][0]]['k3s_node_ip'] }}:{{ k3s_api_port }}"
token: "{{ vault_k3s_cluster_token }}"
{% endif %}
```

---

## 4. Urutan Eksekusi Bertahap (Deterministic Sequencing)

Kunci sukses instalasi k3s HA dengan Ansible terletak pada penggunaan `serial: 1` pada server control-plane:

```yaml
---
# playbooks/deploy-k3s-ha.yml

# ==============================================================================
# TAHAP 1: BOOTSTRAP MASTER PERTAMA (INISIALISASI ETCD QUORUM)
# ==============================================================================
- name: 1. Inisialisasi Master Bootstrap k3s
  hosts: k3s_servers[0] # HANYA server pertama!
  become: true
  tasks:
    - name: Pasang direktori konfigurasi k3s
      ansible.builtin.file:
        path: /etc/rancher/k3s
        state: directory
        mode: "0755"

    - name: Salin template konfigurasi master bootstrap
      ansible.builtin.template:
        src: roles/k3s_server/templates/config.yaml.j2
        dest: /etc/rancher/k3s/config.yaml
        mode: "0600"
      vars:
        is_bootstrap_master: true

    - name: Jalankan instalasi binary k3s server
      ansible.builtin.shell: |
        curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="{{ k3s_version }}" sh -s - server
      args:
        creates: /usr/local/bin/k3s

    - name: Tunggu hingga API Server k3s aktif merespons
      ansible.builtin.wait_for:
        port: 6443
        delay: 5
        timeout: 60

# ==============================================================================
# TAHAP 2: GABUNGKAN MASTER TAMBAHAN SECARA BERGANTIAN (SERIAL 1)
# ==============================================================================
- name: 2. Gabungkan Master Tambahan ke Quorum Etcd
  hosts: k3s_servers[1:] # Seluruh master sisanya
  become: true
  serial: 1 # WAJIB SATU PER SATU agar konsensus etcd stabil!
  tasks:
    - name: Pasang direktori konfigurasi k3s
      ansible.builtin.file:
        path: /etc/rancher/k3s
        state: directory
        mode: "0755"

    - name: Salin template konfigurasi join master
      ansible.builtin.template:
        src: roles/k3s_server/templates/config.yaml.j2
        dest: /etc/rancher/k3s/config.yaml
        mode: "0600"
      vars:
        is_bootstrap_master: false

    - name: Jalankan instalasi binary join server
      ansible.builtin.shell: |
        curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="{{ k3s_version }}" sh -s - server
      args:
        creates: /usr/local/bin/k3s

    - name: Jeda stabilisasi konsensus etcd sebelum server berikutnya masuk
      ansible.builtin.pause:
        seconds: 15

# ==============================================================================
# TAHAP 3: GABUNGKAN SELURUH WORKER / AGENT (PARALEL)
# ==============================================================================
- name: 3. Gabungkan Worker Nodes ke Klaster
  hosts: k3s_agents
  become: true
  tasks:
    - name: Pasang direktori konfigurasi k3s agent
      ansible.builtin.file:
        path: /etc/rancher/k3s
        state: directory
        mode: "0755"

    - name: Tulis file konfigurasi agen k3s
      ansible.builtin.copy:
        dest: /etc/rancher/k3s/config.yaml
        content: |
          server: "https://{{ hostvars[groups['k3s_servers'][0]]['k3s_node_ip'] }}:6443"
          token: "{{ vault_k3s_cluster_token }}"
          node-ip: "{{ k3s_node_ip }}"
        mode: "0600"

    - name: Jalankan installer k3s mode agent
      ansible.builtin.shell: |
        curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="{{ k3s_version }}" sh -s - agent
      args:
        creates: /usr/local/bin/k3s-agent
```

---

## 5. Prosedur Rolling Upgrade Versi k3s Tanpa Downtime

Salah satu keunggulan terbesar mengelola k3s dengan Ansible adalah kemampuan melakukan **Rolling Upgrade** versi Kubernetes tanpa memutus lalu lintas pengguna:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 SIKLUS HIDUP ROLLING UPGRADE SATU NODE (SRE RUNBOOK)        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ LANGKAH 1: CORDON NODE ]                                                 │
│  Tandai node agar scheduler tidak menaruh Pod baru di server ini:           │
│  $ kubectl cordon <node-name>                                               │
│                                                                             │
│  [ LANGKAH 2: DRAIN NODE ]                                                  │
│  Pindahkan seluruh Pod aplikasi secara anggun ke node lain:                 │
│  $ kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data     │
│                                                                             │
│  [ LANGKAH 3: UPGRADE BINARY & RESTART SYSTEMD ]                            │
│  Ansible memperbarui versi biner k3s dan me-restart layanan `k3s.service`:  │
│  $ curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.31.2" sh -s - ... │
│  $ systemctl restart k3s                                                    │
│                                                                             │
│  [ LANGKAH 4: VERIFIKASI KESEHATAN KLASTER ]                                │
│  Tunggu hingga status node kembali `Ready` dan versi baru terlaporkan:      │
│  $ kubectl wait --for=condition=Ready node/<node-name> --timeout=60s        │
│                                                                             │
│  [ LANGKAH 5: UNCORDON NODE ]                                               │
│  Buka kembali node agar siap menerima beban kerja Pod:                      │
│  $ kubectl uncordon <node-name>                                             │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│  Ulangi langkah 1-5 untuk node berikutnya hingga seluruh klaster selesai!   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN K3S HA MULTI-NODE                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "x509: certificate is valid for IP A, not IP B" ]               │
│  Gejala: Worker node atau `kubectl` gagal menghubungi master dengan pesan   │
│          sertifikat TLS tidak valid.                                        │
│  Akar Masalah:                                                              │
│  - Parameter `tls-san` pada template `config.yaml` tidak menyertakan alamat │
│    VIP Load Balancer atau IP node yang digunakan oleh klien.                │
│  Solusi SRE:                                                                │
│  1. Tambahkan IP/FQDN yang hilang ke dalam list `tls-san` pada config.yaml. │
│  2. Restart layanan k3s untuk memperbarui sertifikat otomatis:              │
│     systemctl restart k3s                                                   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: MASTER KEDUA GAGAL JOIN DENGAN TIMEOUT KONSENSUS ]              │
│  Gejala: Instalasi di node 2 macet dan melempar error koneksi etcd.         │
│  Akar Masalah:                                                              │
│  - Port internal komunikasi peer etcd (Port 2379 dan 2380) terblokir oleh   │
│    firewall UFW pada master pertama!                                        │
│  Solusi SRE:                                                                │
│  Pastikan port etcd dibuka khusus untuk subnet cluster:                     │
│  ufw allow from 192.168.10.0/24 to any port 2379,2380 proto tcp            │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: POD TIDAK BISA SALING PING ANTAR-NODE (FLANNEL RUSAK) ]         │
│  Gejala: Pod di Node A tidak bisa berkomunikasi dengan Pod di Node B.       │
│  Akar Masalah:                                                              │
│  - Paket overlay jaringan VXLAN (Port UDP 8472) di-drop oleh firewall fisik │
│    atau UFW.                                                                │
│  Solusi SRE:                                                                │
│  Buka port VXLAN Flannel antar seluruh node:                                │
│  ufw allow from 192.168.10.0/24 to any port 8472 proto udp                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menjalankan playbook instalasi klaster k3s HA
ansible-playbook -i inventories/onprem/hosts.yml playbooks/deploy-k3s-ha.yml

# 2. Mengecek status seluruh node dan arsitektur CPU dari master k3s
kubectl get nodes -o wide

# 3. Mengecek status kesehatan etcd quorum internal k3s
k3s etcd-ctl endpoint health \
  --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/client.key

# 4. Melihat daftar anggota etcd yang aktif
k3s etcd-ctl member list \
  --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/client.key

# 5. Menguji API server readiness endpoint secara detail
kubectl get --raw='/readyz?verbose'
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa dalam playbook Ansible untuk instalasi control-plane k3s multi-server dengan basis data embedded etcd, task penggabungan server master kedua dan ketiga **WAJIB** menggunakan parameter `serial: 1`?
- A. Karena prosesor server master hanya memiliki satu core CPU.
- B. Untuk menjamin keanggotaan etcd bertambah satu demi satu secara teratur sehingga konsensus kuorum Raft tetap stabil dan tidak terjadi perebutan ketua (*split-brain race condition*).
- C. Agar kuota bandwidth internet tidak cepat habis.
- D. Karena k3s tidak mendukung sistem operasi 64-bit jika di-run paralel.

*Kunci Jawaban:* **B**. Penambahan anggota etcd wajib deterministik satu per satu agar kuorum cluster dapat menyesuaikan status keanggotaan basis data secara aman.

### Soal 2 (Pilihan Ganda)
Dalam berkas template `config.yaml.j2`, mengapa kita menyertakan daftar `tls-san` yang memuat alamat IP VIP Load Balancer dan alamat IP seluruh master?
- A. Agar tampilan sertifikat di browser terlihat menarik.
- B. Karena sertifikat TLS/x509 k3s API server secara default hanya memuat `localhost` dan IP internal k3s; ketiadaan IP eksternal/VIP di dalam Subject Alternative Name (SAN) akan memicu error `x509 certificate validation failed` saat diakses oleh klien luar.
- C. Untuk mempercepat enkripsi paket data VXLAN.
- D. Agar k3s otomatis mendapatkan sertifikat gratis dari Let's Encrypt tanpa internet.

*Kunci Jawaban:* **B**. Parameter `tls-san` menyuntikkan daftar IP dan nama domain yang sah ke dalam sertifikat TLS yang diterbitkan oleh CA internal k3s.

### Soal 3 (Pilihan Ganda)
Saat melakukan rolling upgrade versi k3s pada salah satu node worker, mengapa kita harus menjalankan `kubectl drain <node>` sebelum mematikan atau memperbarui biner k3s di node tersebut?
- A. Untuk menghapus seluruh file log sistem di `/var/log`.
- B. Memastikan seluruh Pod aplikasi dievakuasi (*evicted*) secara anggun (*graceful*) ke node lain yang sehat sesuai aturan Pod Disruption Budget (PDB), sehingga pengguna tidak mengalami pemutusan layanan (*zero downtime*).
- C. Untuk memformat ulang partisi swap memory.
- D. Mengubah status node menjadi Master.

*Kunci Jawaban:* **B**. `kubectl drain` adalah prosedur standar SRE untuk mengevakuasi beban kerja kontainer secara tertib sebelum pemeliharaan host fisik/VM dilakukan.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Anda adalah SRE yang memelihara klaster k3s 3-node HA di private data center. Node `sre-k3s-cp-02` mengalami kegagalan listrik mendadak dan mati total. Apakah klaster Kubernetes k3s Anda masih bisa melayani request deployment Pod baru dari developer saat node tersebut mati? Jelaskan alasannya berdasarkan teori kuorum Raft etcd!

*Panduan Solusi:*
1. **Status Ketersediaan Klaster:**
   **YA, KLASTER MASIH TETAP BERFUNGSI SEPENUHNYA!**
2. **Penjelasan Teknis Kuorum Raft:**
   - Rumus kuorum etcd adalah $(N/2) + 1$, di mana $N$ adalah jumlah total anggota master.
   - Untuk klaster 3-node ($N = 3$), kuorum minimum yang dibutuhkan adalah:
     $$(3 / 2) + 1 = 1 + 1 = 2 \text{ Node Aktif}$$
   - Ketika 1 node mati, masih terdapat 2 node master yang hidup (`sre-k3s-cp-01` dan `sre-k3s-cp-03`). Karena 2 node masih memenuhi batas kuorum mayoritas (2 dari 3), etcd tetap memiliki hak voting penuh untuk menyetujui transaksi tulis (*write requests*), menerima pembuatan Pod baru, dan mempertahankan state klaster tanpa hambatan (*Fault Tolerance = 1 Node*).
   - Namun, SRE harus segera memperbaiki node yang mati karena jika 1 node lagi mati (sisa 1), kuorum akan hilang total dan klaster akan terkunci (*read-only*).

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Fase 2: 02 — k3s Multi-Node High Availability](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.2-k3s-production/02-k3s-multi-node-ha.md) — Teori arsitektur etcd quorum.
  - [Modul 4.3: 01 — OpenTofu Handoff dan Host Readiness](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/01-opentofu-inventory-readiness.md) — Validasi kesiapan sebelum k3s dipasang.
- **Topik Berikutnya:**
  - [03 — Hardening, Patching, dan Rolling Maintenance](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/03-hardening-patching-rolling.md) — Memperkuat keamanan OS host k3s dan prosedur penambalan berkala.
  - [04 — Backup, Health, Evidence, dan Rebuild](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/04-backup-health-evidence-rebuild.md) — Pemulihan bencana dan rantai bukti audit.
