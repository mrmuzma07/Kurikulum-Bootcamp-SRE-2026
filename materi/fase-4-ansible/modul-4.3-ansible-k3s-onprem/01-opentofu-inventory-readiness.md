# 01 — OpenTofu Handoff, Dynamic Inventory, dan Host Readiness Matrix: Mengonsumsi Infrastruktur Tanpa Cacat

> **Ringkasan:** Hubungan antara penyedia infrastruktur (*OpenTofu*) dan pengelola konfigurasi sistem (*Ansible*) adalah titik temu paling kritis dalam rantai otomatisasi SRE. Di lingkungan *on-premises* (bare-metal atau private cloud), keberhasilan pembuatan Virtual Machine oleh hypervisor **TIDAK MENJAMIN** bahwa sistem operasi di dalamnya siap dipasangi klaster Kubernetes k3s. Dokumen ini membedah kontrak alih tugas (*handoff contract*) metadata non-rahasia, menyusun matriks 8 gerbang kesiapan host (*Host Readiness Matrix*) yang dieksekusi secara otomatis, serta mengisolasi penanganan penggantian host (*host replacement*) agar tidak merusak konsensus etcd.

---

## 1. Analogi Pemula: Izin Mendirikan Bangunan (IMB) & Sertifikat Laik Fungsi (SLF)

Untuk memahami mengapa Ansible tidak boleh langsung menginstal k3s setelah OpenTofu selesai, mari gunakan analogi konstruksi gedung perkantoran:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PROVISIONING vs INSPEKSI KELAYAKAN HUNIAN           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TAHAP 1: KONTRAKTOR PEMBETONAN (OPENTOFU) -> IMB KELUAR ]                │
│  - Mandor mengecor tiang beton, memasang dinding bata, dan menarik kabel    │
│    listrik utama ke gardu kota.                                             │
│  - Hasil: Gedung fisik berdiri tegak dan memiliki alamat jalan resmi.       │
│  - Apakah karyawan kantor (Aplikasi/Pod Kubernetes) boleh langsung masuk    │
│    dan bekerja di dalam gedung? **TIDAK BOLEH! SANGAT BERBAHAYA!**          │
│                                                                             │
│  [ TAHAP 2: INSPEKTUR KELAYAKAN HUNIAN (ANSIBLE READINESS GATE) -> SLF ]    │
│  - Tim Inspektur Keselamatan masuk membawa papan checklist ketat:           │
│    1. "Apakah sakelar grounding listrik terpasang?" (Swap Memory mati?)     │
│    2. "Apakah jam dinding di setiap lantai cocok?" (Chrony clock skew <100ms)│
│    3. "Apakah pipa pembuangan air lancar?" (Throughput IOPS disk etcd cukup)│
│    4. "Apakah pintu darurat tidak terkunci gembok?" (Port firewall terbuka) │
│  - JIKA ADA 1 SYARAT SAJA YANG GAGAL:                                       │
│    Gedung langsung diberi garis polisi (*SEALED / ABORT*)!                  │
│    Karyawan DILARANG masuk sampai teknisi memperbaiki cacat fisik tersebut! │
│                                                                             │
│  [ TAHAP 3: KANTOR RESMI DIBUKA (KUBERNETES K3S DEPLOYMENT) ]               │
│  - Setelah Sertifikat Laik Fungsi (SLF) terbit, barulah meja, komputer,     │
│    dan karyawan (Pod & Service) diizinkan beroperasi dengan aman!           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Kontrak Handoff OpenTofu ke Ansible (Non-Secret Contract)

Sesuai arsitektur yang dirancang pada [Modul 3.3 OpenTofu](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/03-opentofu-ansible-k3s-handoff.md), OpenTofu menyerahkan metadata infrastruktur ke Ansible dalam format JSON/YAML murni tanpa data sensitif.

### 2.1 Berkas Inventaris Hasil Ekstraksi (`inventories/onprem/hosts.yml`)

```yaml
---
all:
  vars:
    environment_tier: "production"
    cluster_domain: "cluster.local"
    k3s_version_pin: "v1.31.1+k3s1"
    
    # Rujukan Provenance Audit (Metadata Pelacakan)
    iac_provisioning_commit: "8f9b2c1743a19e24f706e25776d6ec1438914619"
    iac_module_version: "v2.4.0"
    iac_backend_state: "s3://tofu-state-prod/clusters/k3s-core/terraform.tfstate"

  children:
    # ------------------------------------------------------------------
    # KELOMPOK CONTROL PLANE K3S (ETCD QUORUM MEMBERS)
    # ------------------------------------------------------------------
    k3s_servers:
      vars:
        node_role: "server"
      hosts:
        sre-k3s-cp-01:
          ansible_host: 192.168.10.11
          is_bootstrap_master: true
          k3s_node_ip: 192.168.10.11
        sre-k3s-cp-02:
          ansible_host: 192.168.10.12
          is_bootstrap_master: false
          k3s_node_ip: 192.168.10.12
        sre-k3s-cp-03:
          ansible_host: 192.168.10.13
          is_bootstrap_master: false
          k3s_node_ip: 192.168.10.13

    # ------------------------------------------------------------------
    # KELOMPOK WORKER / AGENT (COMPUTE WORKLOADS)
    # ------------------------------------------------------------------
    k3s_agents:
      vars:
        node_role: "agent"
      hosts:
        sre-k3s-worker-01:
          ansible_host: 192.168.10.21
          k3s_node_ip: 192.168.10.21
        sre-k3s-worker-02:
          ansible_host: 192.168.10.22
          k3s_node_ip: 192.168.10.22

    k3s_cluster:
      children:
        k3s_servers:
        k3s_agents:
```

> [!CAUTION]
> **Boundary Kredensial:**
> Perhatikan bahwa tidak ada `ansible_ssh_pass`, `k3s_token`, atau kunci privat di dalam file di atas. SSH Key dikelola oleh SSH-Agent pada runner CI, sedangkan join token k3s disimpan di brankas terenkripsi `group_vars/vault.yml`.

---

## 3. Matriks 8 Gerbang Kesiapan Host (Host Readiness Matrix)

Sebelum perintah instalasi k3s dipanggil, playbook audit kesiapan wajib memvalidasi **8 Gerbang Kesiapan Fisik**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 MATRIKS 8 GERBANG KESIAPAN HOST (SRE READINESS)             │
├─────┬──────────────┬──────────────────────────────┬─────────────────────────┤
│ No  │ Gerbang      │ Kondisi Wajib Lulus          │ Kondisi Henti (ABORT)   │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 1   │ Identity     │ Hostname cocok dengan DNS,   │ Nama host duplikat atau │
│     │              │ FQDN valid, role terdefinisi.│ tertukar antar server.  │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 2   │ Network      │ Tidak ada IP collision, ARP  │ Gateway unreachable atau│
│     │              │ respon tunggal, MTU konsisten│ ada bentrokan IP fisik. │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 3   │ DNS          │ Forward (A) dan Reverse (PTR)│ Reverse DNS gagal atau  │
│     │              │ lookup cocok secara simetris.│ timeout di atas 2 detik.│
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 4   │ Time Sync    │ Layanan Chrony aktif, clock  │ Perbedaan waktu > 100ms │
│     │              │ skew antar-node < 100ms.     │ (Raft etcd terancam!).  │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 5   │ SSH & Sudo   │ Auth berbasis Ed25519 sukses,│ Prompt password muncul, │
│     │              │ sudo tanpa password lancar.  │ izin file SSH longgar.  │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 6   │ OS & Memory  │ Ubuntu 22.04 LTS, Swap=0MB,  │ Swap masih aktif (akan  │
│     │              │ kernel cgroups v2 aktif.     │ ditolak oleh Kubelet).  │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 7   │ Storage I/O  │ Partisi `/var/lib/rancher`   │ Partisi penuh (< 20GB)  │
│     │              │ sisa > 20GB, fsync lat <10ms.│ atau latensi I/O tinggi.│
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 8   │ Cloud-Init   │ Status inisialisasi `done`,  │ dpkg/apt lock tertahan  │
│     │              │ apt-lock sudah dilepas.      │ oleh background update. │
└─────┴──────────────┴──────────────────────────────┴─────────────────────────┘
```

---

## 4. Implementasi Playbook Preflight Readiness

Berikut adalah playbook deklaratif yang mengeksekusi pemeriksaan gerbang kesiapan secara otomatis dengan kebijakan *fail-closed* (`any_errors_fatal: true`):

```yaml
---
# playbooks/preflight-readiness.yml - Gerbang Inspeksi Kelayakan Host
- name: Eksekusi 8 Gerbang Kesiapan Host Pra-Instalasi k3s
  hosts: k3s_cluster
  become: true
  gather_facts: true
  any_errors_fatal: true # Jika ada 1 server yang gagal, batalkan seluruh klaster!

  tasks:
    - name: 1. Gate OS Family & Arsitektur CPU
      ansible.builtin.assert:
        that:
          - ansible_facts.os_family == "Debian"
          - ansible_facts.distribution == "Ubuntu"
          - ansible_facts.distribution_major_version | int >= 22
          - ansible_facts.architecture in ['aarch64', 'x86_64']
        fail_msg: "Host {{ inventory_hostname }} menggunakan OS/Arsitektur yang tidak didukung!"

    - name: 2. Gate Swap Memory (Wajib 0 MB)
      ansible.builtin.assert:
        that:
          - ansible_facts.swaptotal_mb == 0
        fail_msg: "SWAP AKTIF pada {{ inventory_hostname }}! Matikan swap sebelum k3s dipasang."

    - name: 3. Gate Kapasitas Disk /var/lib
      ansible.builtin.assert:
        that:
          - (ansible_facts.mounts | selectattr('mount', 'equalto', '/') | map(attribute='size_available') | first) > (20 * 1024 * 1024 * 1024)
        fail_msg: "Ruang disk pada {{ inventory_hostname }} kurang dari 20GB!"

    - name: 4. Gate Sinkronisasi Waktu Chrony
      ansible.builtin.command: chronyc tracking
      register: chrony_status
      changed_when: false
      failed_when: "'System time' not in chrony_status.stdout"

    - name: 5. Gate Cloud-Init Selesai 100%
      ansible.builtin.command: cloud-init status --wait
      changed_when: false
      failed_when: "'status: done' not in cloud_init_res.stdout"
      register: cloud_init_res

    - name: 6. Gate Port Jaringan Tidak Bentrok
      ansible.builtin.wait_for:
        port: 6443
        state: stopped
        timeout: 3
      failed_when: false # Port 6443 harus bebas sebelum k3s aktif!

    - name: 7. Laporan Sukses Gerbang Kesiapan
      ansible.builtin.debug:
        msg: "==> [LULUS] Host {{ inventory_hostname }} ({{ ansible_host }}) 100% SIAP UNTUK K3S."
```

---

## 5. Penanganan Penggantian Host (Host Replacement & Drift)

Apa yang terjadi jika suatu hari salah satu VM server mengalami kerusakan hardware dan OpenTofu me-*recreate* VM tersebut dengan UUID baru?

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 SOP PENANGANAN PENGGANTIAN HOST CONTROL PLANE               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BAHAYA: Jangan langsung jalankan playbook install k3s ke server baru!     │
│  Anggota cluster etcd lama masih terdaftar di database quorum.             │
│                                                                             │
│  [ LANGKAH 1: ISOLASI DARI KLASTER K3S AKTIF ]                              │
│  Dari salah satu master k3s yang masih sehat:                               │
│  $ kubectl get nodes                                                        │
│  $ kubectl cordon sre-k3s-cp-02                                             │
│  $ kubectl drain sre-k3s-cp-02 --delete-emptydir-data --force --ignore-daemonsets│
│  $ kubectl delete node sre-k3s-cp-02                                        │
│                                                                             │
│  [ LANGKAH 2: HAPUS DARI ANGGOTA ETCD RAFT ]                                │
│  $ k3s etcd-ctl member list                                                 │
│  $ k3s etcd-ctl member remove <MEMBER_ID_LAMA>                              │
│                                                                             │
│  [ LANGKAH 3: VERIFIKASI QUORUM TERSISA TETAP SEHAT ]                       │
│  $ k3s etcd-ctl endpoint health                                             │
│                                                                             │
│  [ LANGKAH 4: JALANKAN PROVISIONING & READINESS GATE PADA HOST BARU ]       │
│  $ tofu apply -target=module.compute_hosts.proxmox_vm_qemu.cp2              │
│  $ ansible-playbook playbooks/preflight-readiness.yml --limit sre-k3s-cp-02 │
│                                                                             │
│  [ LANGKAH 5: GABUNGKAN SEBAGAI ANGGOTA FRESH SERVER JOIN ]                 │
│  $ ansible-playbook playbooks/deploy-k3s.yml --limit sre-k3s-cp-02          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN HOST READINESS                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: CLOCK SKEW MELAMPAUI AMBANG BATAS (> 100ms) ]                   │
│  Gejala: Playbook assertion gagal: "Max error 0.428412 seconds exceeds 0.1s"│
│  Akar Masalah:                                                              │
│  - VM baru bangun dari mode sleep/suspend, atau hypervisor host mengalami   │
│    kelebihan beban CPU sehingga waktu di dalam VM berjalan lambat.         │
│  Solusi SRE:                                                                │
│  1. Paksa sinkronisasi seketika: `ansible all -m command -a "chronyc makestep"`│
│  2. Jika hypervisor Proxmox/vSphere: aktifkan integrasi waktu guest agent:  │
│     `qemu-guest-agent` time sync daemon.                                    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: APT-LOCK TERKUNCI OLEH UNATTENDED-UPGRADES ]                    │
│  Gejala: Preflight macet di task apt: "Could not get lock /var/lib/dpkg/lock"│
│  Akar Masalah:                                                              │
│  - Cloud-Init baru saja selesai, tetapi daemon `unattended-upgrades` Ubuntu │
│    sedang mengunduh pembaruan security patch di latar belakang.             │
│  Solusi SRE:                                                                │
│  Tunggu proses selesai dengan graceful timeout:                             │
│  ansible all -m shell -a "while fuser /var/lib/dpkg/lock >/dev/null 2>&1;   │
│  do sleep 3; done"                                                          │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: REVERSE DNS (PTR) GAGAL DI LINGKUNGAN ON-PREM ]                 │
│  Gejala: Koneksi SSH antar-node membutuhkan waktu 10-15 detik (sangat lambat)│
│  Akar Masalah:                                                              │
│  - Server DNS internal data center tidak memiliki catatan PTR untuk IP node │
│    k3s, sehingga daemon sshd mengalami DNS timeout saat reverse lookup.     │
│  Solusi SRE:                                                                │
│  1. Daftarkan PTR Record pada server DNS lokal, ATAU                        │
│  2. Nonaktifkan lookup di sshd_config via role common:                      │
│     UseDNS no                                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menjalankan pemeriksaan kesiapan seluruh node sebelum instalasi k3s
ansible-playbook -i inventories/onprem/hosts.yml playbooks/preflight-readiness.yml

# 2. Memeriksa status sinkronisasi waktu Chrony di seluruh server cluster
ansible k3s_cluster -m command -a "chronyc tracking"

# 3. Memverifikasi bahwa swap memory 100% mati di seluruh server
ansible k3s_cluster -m command -a "free -h"

# 4. Memeriksa apakah ada port k3s (6443) yang sudah terpakai sebelum instalasi
ansible k3s_servers -m command -a "ss -tulpn | grep 6443"

# 5. Memeriksa status Cloud-Init pada node yang baru saja dibuat
ansible k3s_cluster -m command -a "cloud-init status"
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa dalam gerbang kesiapan (*readiness gate*) k3s, toleransi perbedaan waktu (*clock skew*) antar server control-plane dibatasi sangat ketat (maksimal di bawah 100 milidetik)?
- A. Agar tampilan log Grafana memiliki warna yang seragam.
- B. Karena algoritma konsensus Raft pada etcd sangat bergantung pada interval heartbeat waktu yang presisi untuk pemilihan ketua (*leader election*); deviasi waktu memicu *split-brain* dan kegagalan enkripsi TLS.
- C. Karena k3s hanya bisa diinstal pada zona waktu UTC.
- D. Agar cron job Linux berjalan tepat waktu.

*Kunci Jawaban:* **B**. Toleransi waktu yang longgar akan merusak kestabilan konsensus etcd dan menggagalkan validasi sertifikat x509 antar-komponen Kubernetes.

### Soal 2 (Pilihan Ganda)
Sebelum menjalankan playbook k3s pada VM baru yang baru saja di-*provisioning* oleh OpenTofu, task preflight menjalankan perintah `cloud-init status --wait`. Apa tujuan mendasar dari langkah ini?
- A. Mengunduh wallpaper desktop Ubuntu.
- B. Memastikan bahwa seluruh konfigurasi awal sistem operasi (seperti penanaman SSH key, hostname, dan inisialisasi jaringan) telah tuntas 100% dan kunci `dpkg/apt` telah dilepas.
- C. Memformat ulang harddisk server.
- D. Mengubah IP address server menjadi dinamis.

*Kunci Jawaban:* **B**. `cloud-init status --wait` mencegah tabrakan eksekusi dengan proses inisialisasi latar belakang bawaan sistem operasi.

### Soal 3 (Pilihan Ganda)
Jika sebuah node master k3s mengalami kerusakan fisik dan OpenTofu membuatkan VM baru dengan IP yang sama, apa langkah pertama yang wajib dilakukan pada klaster Kubernetes sebelum memasukkan node baru tersebut?
- A. Langsung menginstall k3s dengan opsi `--force`.
- B. Menghapus cluster Kubernetes lama dan membuat baru dari nol.
- C. Melakukan `kubectl cordon`, `drain`, dan menghapus ID anggota etcd lama dari database quorum via `k3s etcd-ctl member remove`.
- D. Mematikan seluruh firewall di server master lainnya.

*Kunci Jawaban:* **C**. Menghapus identitas lama dari database etcd wajib dilakukan agar node baru dapat mendaftar dengan identitas dan status TLS fresh tanpa konflik quorum.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Di sebuah private data center, tim Anda menjalankan pipeline `preflight-readiness.yml`. Pada task pengujian partisi disk, node `sre-k3s-worker-01` melempar pesan kegagalan:
`GERBANG KESIAPAN GAGAL: Ruang disk pada sre-k3s-worker-01 kurang dari 20GB!`
Setelah diinvestigasi, ternyata teknisi salah memilih template VM di Proxmox yang hanya memiliki disk virtual 10GB. Jelaskan tindakan SRE yang benar sesuai batas tanggung jawab (*boundary ownership*): apakah Anda memperbaikinya lewat task Ansible tambahan atau mengembalikannya ke OpenTofu?

*Panduan Solusi:*
1. **Analisis Batas Tanggung Jawab (*Boundary Ownership*):**
   Alokasi ukuran disk virtual (*virtual disk size*) adalah tanggung jawab lapisan **Infrastruktur / Provisioning (OpenTofu)**, bukan tanggung jawab Ansible. Mencoba mempartisi ulang atau memperbesar disk secara paksa lewat skrip Ansible mentah akan memicu *configuration drift* dan inkonsistensi terhadap state OpenTofu.
2. **Langkah Perbaikan yang Benar:**
   - **Tolak Eksekusi Ansible:** Hentikan pipeline Ansible pada node tersebut (*Fail-Closed*).
   - **Perbaiki di OpenTofu:** Buka modul OpenTofu dan sesuaikan variabel kapasitas disk:
     ```hcl
     disk_size = "50G"
     ```
   - **Terapkan di Hypervisor:** Jalankan `tofu apply` untuk memperbesar disk fisik di Proxmox.
   - **Verifikasi Ulang:** Jalankan kembali `preflight-readiness.yml` via Ansible untuk memastikan partisi telah terbaca > 20GB dan lolos inspeksi kesiapan.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Fase 3: 03 — OpenTofu, Ansible, dan k3s Handoff](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/03-opentofu-ansible-k3s-handoff.md) — Memahami kontrak data non-rahasia dari OpenTofu.
  - [Modul 4.2: 03 — Check, Diff, Limit, Error Handling, dan CI](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.2-pola-produksi-ansible/03-check-diff-limit-error-handling-ci.md) — Menggunakan defensive assertion dan guardrail eksekusi.
- **Topik Berikutnya:**
  - [02 — Role Instalasi dan Upgrade k3s](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/02-k3s-install-upgrade-role.md) — Mengorkestrasi bootstrap dan penambahan node k3s multi-server HA.
  - [LAB-01 — Handoff k3s Multi-node](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/lab/LAB-01-handoff-k3s-multinode.md) — Praktik hands-on pengujian gerbang kesiapan dan deployment k3s.
