# 03 — Hardening, Patching, dan Rolling Maintenance: Menjaga Imunitas dan Siklus Pemeliharaan Node k3s

> **Ringkasan:** Di lingkungan *bare-metal* dan *private data center*, sistem operasi *host* yang mendasari klaster Kubernetes adalah benteng pertahanan pertama. Kerentanan kernel (*CVE*) atau konfigurasi jaringan yang longgar dapat membahayakan seluruh beban kerja kontainer di atasnya. Namun, menambal sistem operasi tidak boleh memicu *outage* pada aplikasi pelanggan. Dokumen ini membedah matriks penguatan keamanan OS (*OS Hardening Matrix*), menyusun otomatisasi penambalan bergulir (*Rolling OS Patching*) yang terintegrasi secara anggun dengan siklus hidup *drain/uncordon* Kubernetes, serta menghormati batasan *Pod Disruption Budget* (PDB) demi menjamin ketersediaan layanan tanpa henti (*zero downtime*).

---

## 1. Analogi Pemula: Perawatan Berkala Armada Kapal Selam di Laut Lepas

Untuk memahami mengapa penambalan OS pada klaster Kubernetes memerlukan alur bergulir (*rolling*), mari gunakan analogi armada pertahanan maritim:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PERAWATAN BERKALA ARMADA KAPAL SELAM                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ CARA AMATIR: SELURUH ARMADA MASUK DOK BERSAMAAN ]                        │
│  - Komandan memerintahkan seluruh 5 kapal selam masuk ke galangan kapal     │
│    pada hari yang sama untuk ganti oli dan cat lambung.                     │
│  - Akibat: Perairan negara kosong melompong tanpa perlindungan selama       │
│    3 hari (**TOTAL OUTAGE / DOWNTIME PRODUKSI**)! Musuh bebas masuk!        │
│                                                                             │
│  [ CARA SRE: ROLLING MAINTENANCE BERGULIR DENGAN EVAKUASI ANGGUN ]          │
│  - Armada melakukan giliran perawatan satu per satu (`serial: 1`):          │
│                                                                             │
│    1. Kapal 1 bersiap masuk dok: Tandai bendera merah (**CORDON**).         │
│       Kapal 1 tidak boleh menerima tugas patroli baru!                      │
│                                                                             │
│    2. Evakuasi Muatan (**DRAIN**):                                          │
│       Awak kapal dan muatan penting dipindahkan ke Kapal 2, 3, 4, dan 5     │
│       yang sedang berpatroli secara seimbang.                               │
│                                                                             │
│    3. Pemeliharaan Lambung & Mesin (**PATCHING & REBOOT**):                 │
│       Ganti oli, tambal lambung bocor, perbarui sistem sonar, dan restart!  │
│                                                                             │
│    4. Uji Kesiapan Berlayar (**HEALTH CHECK**):                             │
│       Pastikan mesin menyala, radar aktif, dan status kapal prima.          │
│                                                                             │
│    5. Lepas Bendera Merah (**UNCORDON**):                                   │
│       Kapal 1 kembali ke laut lepas melayani patroli secara normal!         │
│                                                                             │
│    6. BARU LANJUTKAN PROSEDUR KE KAPAL 2!                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Matriks Penguatan Sistem Operasi Host (OS Hardening Matrix)

Sebelum klaster k3s melayani lalu lintas produksi, sistem operasi Ubuntu 22.04 LTS pada seluruh node wajib diperkuat melalui peran Ansible `common_hardening`:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 MATRIKS PENGUATAN KEAMANAN OS (OS HARDENING MATRIX)         │
├─────┬──────────────────────┬────────────────────────────────────────────────┤
│ No  │ Lapisan Hardening    │ Konfigurasi Deklaratif SRE                     │
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 1   │ SSH Lockdown         │ - `PermitRootLogin no` (Wajib non-root user).  │
│     │                      │ - `PasswordAuthentication no` (Key-Only auth). │
│     │                      │ - `MaxAuthTries 3` & `ClientAliveInterval 300`.│
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 2   │ Kernel Sysctl        │ - `net.ipv4.ip_forward = 1` (Routing Pod).     │
│     │                      │ - `net.bridge.bridge-nf-call-iptables = 1`.    │
│     │                      │ - `vm.max_map_count = 262144` (Elastic/DB).    │
│     │                      │ - `fs.file-max = 2097152` (High socket limits).│
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 3   │ Swap Elimination     │ - `swapoff -a` seketika di memori.             │
│     │                      │ - Hapus entri swap permanen di `/etc/fstab`.   │
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 4   │ Network Firewall UFW │ - Default incoming DENY, Default outgoing ALLOW│
│     │                      │ - Port 22 (SSH) dibatasi ke IP Bastion/VPN.    │
│     │                      │ - Port k3s (6443, 2379-2380, 8472 UDP) dibuka │
│     │                      │   HANYA untuk subnet internal cluster.         │
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 5   │ Brute-Force Shield   │ - Pasang Fail2ban pada layanan OpenSSH.        │
│     │                      │ - Ban IP penyerang setelah 5 kegagalan login.  │
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 6   │ Audit & Time Sync    │ - Aktifkan auditd untuk pelacakan hak akses.   │
│     │                      │ - Kunci sinkronisasi waktu Chrony < 100ms.     │
└─────┴──────────────────────┴────────────────────────────────────────────────┘
```

---

## 3. Otomatisasi Rolling Patching Terintegrasi Kubernetes

Berikut adalah playbook produksi Ansible yang mengorkestrasi pembaruan keamanan sistem operasi secara bergulir tanpa menyebabkan pemadaman klaster:

```yaml
---
# playbooks/rolling-os-patch.yml - Siklus Pemeliharaan OS Tanpa Downtime
- name: Eksekusi Rolling OS Patching dan Kernel Maintenance pada Klaster k3s
  hosts: k3s_cluster
  become: true
  serial: 1 # EKSEKUSI WAJIB SATU PER SATU (ZERO DOWNTIME)!
  any_errors_fatal: true # Batalkan antrean jika ada node yang gagal reboot!

  vars:
    k8s_master_endpoint: "https://192.168.10.11:6443"

  tasks:
    # --------------------------------------------------------------------------
    # TAHAP 1: PREFLIGHT KESEHATAN KLASTER
    # --------------------------------------------------------------------------
    - name: 1. Pastikan node dalam kondisi Ready sebelum disentuh
      ansible.builtin.command: >
        kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml get node {{ inventory_hostname }} -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
      register: node_ready_status
      delegate_to: "{{ groups['k3s_servers'][0] }}"
      failed_when: node_ready_status.stdout != "True"
      changed_when: false

    # --------------------------------------------------------------------------
    # TAHAP 2: ISOLASI & EVAKUASI BEBAN KERJA (CORDON & DRAIN)
    # --------------------------------------------------------------------------
    - name: 2. Tandai node sebagai Unschedulable (Cordon)
      ansible.builtin.command: >
        kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml cordon {{ inventory_hostname }}
      delegate_to: "{{ groups['k3s_servers'][0] }}"
      changed_when: true

    - name: 3. Evakuasi Pod secara anggun ke node lain (Drain)
      ansible.builtin.command: >
        kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml drain {{ inventory_hostname }}
        --ignore-daemonsets
        --delete-emptydir-data
        --force
        --grace-period=60
        --timeout=300s
      delegate_to: "{{ groups['k3s_servers'][0] }}"
      changed_when: true

    # --------------------------------------------------------------------------
    # TAHAP 3: PENAMBALAN PAKET SISTEM OPERASI (APT UPGRADE)
    # --------------------------------------------------------------------------
    - name: 4. Perbarui repositori cache dan upgrade paket keamanan OS
      ansible.builtin.apt:
        upgrade: dist
        update_cache: true
        cache_valid_time: 3600

    # --------------------------------------------------------------------------
    # TAHAP 4: EVALUASI KEBUTUHAN REBOOT KERNEL
    # --------------------------------------------------------------------------
    - name: 5. Periksa apakah sistem membutuhkan reboot kernel baru
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required_file

    - name: 6. Lakukan reboot mesin secara terkendali jika diperlukan
      ansible.builtin.reboot:
        msg: "Reboot otomatis oleh Ansible pasca kernel security patching"
        connect_timeout: 10
        reboot_timeout: 300
        pre_reboot_delay: 5
        post_reboot_delay: 15
      when: reboot_required_file.stat.exists

    # --------------------------------------------------------------------------
    # TAHAP 5: VERIFIKASI PEMULIHAN LAYANAN & PENGAKTIFAN KEMBALI (UNCORDON)
    # --------------------------------------------------------------------------
    - name: 7. Tunggu hingga node kembali berstatus Ready di Kubernetes
      ansible.builtin.command: >
        kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml wait --for=condition=Ready node/{{ inventory_hostname }} --timeout=120s
      delegate_to: "{{ groups['k3s_servers'][0] }}"
      changed_when: false

    - name: 8. Buka kembali node agar siap menerima Pod baru (Uncordon)
      ansible.builtin.command: >
        kubectl --kubeconfig=/etc/rancher/k3s/k3s.yaml uncordon {{ inventory_hostname }}
      delegate_to: "{{ groups['k3s_servers'][0] }}"
      changed_when: true

    - name: 9. Jeda stabilisasi sebelum melanjutkan ke host berikutnya
      ansible.builtin.pause:
        seconds: 20
```

---

## 4. Pengendalian Pod Disruption Budget (PDB) & Kesiapan Aplikasi

Mengapa proses `drain` pada task nomor 3 di atas bisa menunggu dengan anggun alih-alih mematikan Pod secara brutal?

Jawabannya adalah **Pod Disruption Budget (PDB)** yang didefinisikan oleh developer aplikasi:

```yaml
# Contoh Manifest PDB Aplikasi Pembayaran (Payment Gateway)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api-pdb
spec:
  minAvailable: 2 # Wajib minimal 2 Pod hidup setiap detik!
  selector:
    matchLabels:
      app: payment-api
```

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 INTERAKSI ANSIBLE DRAIN DENGAN KUBERNETES PDB               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Ansible mengeksekusi: `kubectl drain sre-k3s-worker-01`                 │
│  2. Kubernetes API memeriksa PDB aplikasi `payment-api`:                    │
│     - Total Pod saat ini = 3.                                               │
│     - Batas minimum Pod hidup (`minAvailable`) = 2.                         │
│     - Evakuasi 1 Pod diizinkan!                                             │
│  3. Pod baru dibuat di `sre-k3s-worker-02` dan ditunggu hingga `Ready`.     │
│  4. Setelah Pod baru aktif, barulah Pod di `worker-01` dimatikan secara     │
│     aman tanpa ada request pengguna yang terputus (*Zero HTTP 502*)!        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA ROLLING MAINTENANCE & DRAIN                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: DRAIN HANG TAK BERUJUNG (TIMEOUT SETELAH 300 DETIK) ]           │
│  Gejala: Playbook terhenti pada task drain dengan error:                    │
│          "Cannot evict pod as it would violate the pod's disruption budget" │
│  Akar Masalah:                                                              │
│  - Aplikasi hanya memiliki 1 replika, tapi memiliki PDB `minAvailable: 1`. │
│    Kubernetes menolak mematikan Pod tersebut karena tidak ada penggantinya! │
│  Solusi SRE:                                                                │
│  1. Koordinasikan dengan tim pengembang untuk menaikkan replika aplikasi:   │
│     kubectl scale deployment payment-api --replicas=3                       │
│  2. Jika dalam situasi darurat bencana fisik host: gunakan override batas:  │
│     kubectl drain <node> --disable-eviction                                 │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: NODE TIDAK KEMBALI "READY" PASCA REBOOT ]                       │
│  Gejala: Mesin berhasil hidup via SSH, tapi status Kubernetes tetap NotReady│
│  Akar Masalah:                                                              │
│  - Layanan `k3s.service` gagal startup karena swap memory menyala kembali   │
│    akibat lupa menghapus baris swap di `/etc/fstab`!                        │
│  Solusi SRE:                                                                │
│  1. Matikan swap seketika: `swapoff -a`                                     │
│  2. Hapus entri swap secara permanen dari berkas fstab:                     │
│     sed -i '/swap/d' /etc/fstab                                             │
│  3. Restart layanan k3s: `systemctl restart k3s`                            │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: NODE TERJEBAK DALAM STATUS "SchedulingDisabled" ]               │
│  Gejala: Pemeliharaan selesai, tapi tidak ada Pod yang dijadwalkan ke node. │
│  Akar Masalah:                                                              │
│  - Playbook gagal di tengah jalan sebelum mencapai task `uncordon`, atau    │
│    operator lupa mengeksekusi uncordon manual.                              │
│  Solusi SRE:                                                                │
│  Buka kembali penjadwalan secara eksplisit:                                 │
│  kubectl uncordon <node-name>                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menjalankan playbook rolling patching satu per satu pada cluster produksi
ansible-playbook -i inventories/onprem/hosts.yml playbooks/rolling-os-patch.yml

# 2. Mengisolasi node secara manual dari jadwal penempatan Pod
kubectl cordon sre-k3s-worker-01

# 3. Mengosongkan node dari seluruh Pod aplikasi secara manual
kubectl drain sre-k3s-worker-01 --ignore-daemonsets --delete-emptydir-data --force

# 4. Mengembalikan node agar siap menerima Pod kembali
kubectl uncordon sre-k3s-worker-01

# 5. Mengecek daftar Pod yang memiliki aturan PDB di dalam cluster
kubectl get pdb -A

# 6. Memeriksa apakah ada file penanda reboot sistem operasi
ansible k3s_cluster -m command -a "ls -l /var/run/reboot-required"
```

---

## 7. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa dalam prosedur pemeliharaan sistem operasi host Kubernetes, perintah `kubectl drain` wajib menyertakan flag `--ignore-daemonsets`?
- A. Agar seluruh DaemonSet dihapus permanen dari cluster.
- B. Karena DaemonSet (seperti kube-proxy, MetalLB speaker, atau agen logging) didesain untuk berjalan di *setiap* node secara otomatis; tanpa flag ini, perintah drain akan menolak berjalan karena DaemonSet tidak bisa dipindahkan secara biasa.
- C. Untuk mematikan koneksi internet pada node.
- D. Agar proses download kernel berjalan lebih cepat.

*Kunci Jawaban:* **B**. Pod DaemonSet terikat pada node secara khusus dan dikelola oleh kontroler DaemonSet, sehingga perintah `drain` mewajibkan konfirmasi eksplisit via `--ignore-daemonsets`.

### Soal 2 (Pilihan Ganda)
Apa risiko fatal yang terjadi jika seorang administrator sistem menjalankan perintah `apt-get dist-upgrade && reboot` secara serentak ke seluruh server master k3s menggunakan modul ad-hoc tanpa koordinasi `serial: 1`?
- A. Kuota harddisk server master akan berkurang drastis.
- B. Seluruh node etcd akan reboot pada detik yang sama, memusnahkan kuorum konsensus etcd secara instan (*Total Control Plane Outage*) dan berisiko merusak basis data transaksi klaster.
- C. Versi Linux otomatis diturunkan (*downgrade*) ke Ubuntu 18.04.
- D. Seluruh sertifikat TLS akan otomatis terhapus.

*Kunci Jawaban:* **B**. Reboot serentak pada seluruh master menghancurkan kuorum mayoritas etcd, melumpuhkan Kubernetes API Server, dan membuat klaster tidak dapat diakses.

### Soal 3 (Pilihan Ganda)
Bagaimana mekanisme *Pod Disruption Budget* (PDB) melindungi ketersediaan aplikasi saat Ansible menjalankan perintah `kubectl drain`?
- A. PDB secara otomatis memblokir koneksi SSH dari Ansible.
- B. PDB membatasi jumlah Pod yang boleh mati secara bersamaan selama penggusuran (*voluntary disruptions*), memaksa perintah drain menunggu hingga Pod pengganti di node lain berstatus sehat sebelum menghancurkan Pod lama.
- C. PDB mengonversi Pod menjadi file zip cadangan.
- D. PDB mematikan seluruh firewall di server worker.

*Kunci Jawaban:* **B**. PDB bertindak sebagai kontrak batas toleransi pemadaman antara pengembang aplikasi dan operator infrastruktur (*SRE*).

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Sebuah kerentanan keamanan kritis pada Linux Kernel (tingkat keparahan CVSS 9.8) diumumkan hari ini. Anda diwajibkan menambal 20 worker node k3s produksi hari ini juga. Namun, manajer produk memperingatkan bahwa aplikasi pembayaran tidak boleh mengalami downtime sedetik pun. Jelaskan langkah-langkah mitigasi dan parameter Ansible yang wajib Anda konfigurasikan untuk memenuhi kedua tuntutan tersebut!

*Panduan Solusi:*
1. **Parameter Ansible Wajib:**
   - Gunakan `serial: 1` pada playbook patching agar pemeliharaan dieksekusi tepat satu mesin dalam satu waktu.
   - Gunakan `any_errors_fatal: true` sebagai *Circuit Breaker*: jika mesin pertama gagal reboot atau tidak kembali `Ready`, seluruh antrean 19 mesin lainnya langsung dibatalkan secara otomatis (*Fail-Closed*).
2. **Langkah-Langkah Mitigasi Zero-Downtime:**
   - **Audit PDB:** Pastikan aplikasi pembayaran memiliki manifest `PodDisruptionBudget` dengan `minAvailable: 2` (dan jumlah replika Pod minimal 3).
   - **Prosedur Cordon & Drain:** Sebelum mengeksekusi `apt-get upgrade`, jalankan `kubectl cordon` dan `kubectl drain` dengan `--grace-period=60`.
   - **Reboot & Health Verification:** Gunakan modul `ansible.builtin.reboot` dan tindak lanjuti dengan `kubectl wait --for=condition=Ready node/<node-name>`.
   - **Uncordon:** Kembalikan node ke jadwal penjadwalan sebelum berpindah ke node berikutnya.

---

## 8. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — OpenTofu Handoff dan Host Readiness](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/01-opentofu-inventory-readiness.md) — Mengonfirmasi baseline sebelum patching.
  - [02 — Role Instalasi dan Upgrade k3s Multi-Node HA](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/02-k3s-install-upgrade-role.md) — Memahami siklus hidup control plane k3s.
- **Topik Berikutnya:**
  - [04 — Backup, Health, Evidence, dan Rebuild](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/04-backup-health-evidence-rebuild.md) — Menyiapkan snapshot etcd, pemulihan bencana (*disaster recovery*), dan rekonstruksi total infrastruktur.
  - [LAB-02 — Rolling Patching dan Readiness](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/lab/LAB-02-rolling-patching-readiness.md) — Praktik hands-on rolling maintenance di terminal.
