# 01 — Network, Storage, Hardware, dan Access On-Prem

> **Ringkasan:** Memahami realitas operasional infrastruktur **On-Premises Bare-Metal** di mana SRE mengelola seluruh lapisan fisik secara mandiri: segmentasi jaringan VLAN, alokasi IP MetalLB dan mitigasi *stale ARP*, perbandingan topologi penyimpanan data (**Local PV vs NFS/SAN**), pemantauan kesehatan perangkat keras (IPMI, SMART disk, ECC RAM), serta pengamanan akses server melalui **Bastion Jump Host** dan tata kelola *Break-Glass*.

---

## 1. Realitas On-Premises: Berpindah dari Kemudahan Cloud ke Kendali Penuh

### 1.1 Analogi Dunia Nyata: "Menyewa Apartemen Mewah (Cloud) vs Membangun Kota Mandiri Off-Grid (On-Premises)"
Bayangkan dua gaya hidup:
- **Cloud Publik (Apartemen Mewah):** Saat Anda butuh air minum, Anda tinggal memutar keran (*Create ALB*). Saat butuh gudang penyimpanan, Anda menyewa loker (*Attach EBS Volume*). Jika ada pipa air bocor di lorong, teknisi gedung apartemen memperbaikinya tanpa Anda perlu tahu merek pompanya. Kemudahan ini dibayar dengan biaya sewa bulanan yang sangat mahal.
- **On-Premises Bare-Metal (Kota Mandiri Off-Grid):** Anda memiliki lahan tanah sendiri. Tidak ada tagihan sewa bulanan pihak ketiga, dan Anda memiliki kendali penuh atas keamanan dan kinerja perangkat keras. Namun konsekuensinya: **Anda harus menggali sumur sendiri, memasang trafo listrik sendiri, membentang kabel fiber optik fisik, dan mengonfigurasi sakelar wesel jaringan sendiri**. Jika sebuah switch jaringan hang atau hard disk server fisik berbunyi mencicit di ruang server (*data center*), tidak ada tombol "Support Ticket AWS" yang bisa Anda klik. **Tim SRE-lah yang menjadi pemilik dan penanggung jawab seluruh lapisan tersebut!**

---

### 1.2 Topologi Jaringan Fisik & Segmentasi VLAN On-Premises

Dalam arsitektur bare-metal produksi, seluruh lalu lintas jaringan **DILARANG KERAS** dicampur ke dalam satu subnet datar (*flat network*). SRE memisahkan jalur komunikasi menggunakan **Virtual LAN (VLAN)**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 TOPOLOGI SEGMENTASI JARINGAN BARE-METAL                     │
│                                                                             │
│  [ Jaringan Luar / Kantor / Internet ]                                      │
│                     │                                                       │
│                     ▼ (Firewall / Edge Router)                              │
│       ┌───────────────────────────┐                                         │
│       │   BASTION / JUMP HOST     │ (Akses SSH Terisolasi + MFA)            │
│       └─────────────┬─────────────┘                                         │
│                     │                                                       │
│  ┌──────────────────┴────────────────────────────────────────────────────┐  │
│  │ TOP-OF-RACK (ToR) MANAGED SWITCH                                      │  │
│  ├───────────────────────────────────────────────────────────────────────┤  │
│  │ VLAN 10 (Management): IPMI, iDRAC, Switch Admin (Isolasi Total!)     │  │
│  ├───────────────────────────────────────────────────────────────────────┤  │
│  │ VLAN 20 (Node & Control-Plane): Alamat IP Fisik Server k3s (Host IP)  │  │
│  ├───────────────────────────────────────────────────────────────────────┤  │
│  │ VLAN 30 (Workload / MetalLB VIP): Pool IP Publik/Lokal Layanan Bisnis │  │
│  ├───────────────────────────────────────────────────────────────────────┤  │
│  │ VLAN 40 (Storage Network): Jalur Khusus NFS/iSCSI/Ceph (Jumbo Frames) │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Kegagalan Jaringan On-Premises: ARP, MTU, dan Asymmetric Routing

Di lingkungan cloud, *Virtual Private Cloud (VPC)* menyembunyikan protokol lapisan bawah. Di lingkungan on-premises bare-metal, SRE wajib menguasai 3 potensi kegagalan jaringan fisik:

### 2.1 MetalLB L2 Mode & Masalah Stale ARP Cache
Ketika MetalLB berjalan dalam mode **Layer 2 (L2)**:
- Satu node k3s dipilih menjadi pemimpin (*leader speaker*) yang merespons pertanyaan ARP: *"Siapa pemilik IP VIP `192.168.30.100`?"* dengan alamat MAC fisik kartu jaringannya.
- **Failure Mode — Stale ARP saat Node Mati:**
  Jika node leader tersebut mati mendadak, MetalLB memindahkan IP VIP ke node lain dalam hitungan detik menggunakan paket **Gratuitous ARP (GARP)**.
  Namun, jika switch jaringan ToR (*Top-of-Rack*) atau router kantor memiliki *ARP cache timeout* yang terlalu lama (misal 20 menit) dan mengabaikan paket GARP, maka trafik dari pengguna luar akan terus dikirim ke alamat MAC server yang sudah mati!
  - **Solusi SRE:** Pastikan `strictARP: true` aktif pada konfigurasi `kube-proxy`, dan atur timer timeout ARP pada switch ToR menjadi agresif (misal 60–120 detik).

---

### 2.2 Ketidakcocokan Ukuran Paket (MTU Mismatch)
- Standar kartu jaringan Ethernet menggunakan **MTU 1500 Bytes**.
- Jaringan penyimpanan data (*Storage VLAN*) sering dikonfigurasi menggunakan **Jumbo Frames (MTU 9000 Bytes)** untuk mempercepat transfer backup dan replikasi disk.
- **Bencana Silent Packet Drop:** Jika sebuah node dikonfigurasi dengan MTU 9000, tetapi switch di tengah jalan hanya mendukung MTU 1500, maka paket berukuran kecil (seperti ping ICMP atau handshake TCP awal) akan **LOLOS**, namun saat aplikasi mulai mengirim data besar (seperti query payload 4 KB), paket tersebut akan **DIBUANG SECARA SENYAP (*silent drop*)** tanpa pesan error yang jelas! Aplikasi akan mengalami *hanging* dan koneksi *timeout* misterius.

---

## 3. Pilihan Arsitektur Penyimpanan On-Premises (Storage Topologies)

Kubernetes membutuhkan `StorageClass` untuk menyediakan `PersistentVolume (PV)` bagi database PostgreSQL, MinIO, atau Loki. Di lingkungan on-prem, SRE harus memilih kompromi arsitektur yang tepat:

| Jenis Penyimpanan | Mekanisme & Contoh | Keunggulan Utama | Risiko & Tantangan SRE |
|---|---|---|---|
| **1. Local Persistent Volume (Local PV)** | Data disimpan langsung di hard disk NVMe/SSD lokal milik node tempat pod berjalan (`local-path-provisioner`). | **Latensi Ultra-Rendah & IOPS Tertinggi.** Sangat cepat untuk database transaksi. | **Terkunci pada 1 Node Fisik!** Jika server fisik tersebut mati, pod tidak bisa dipindahkan (*failover*) ke server lain tanpa prosedur restore backup manual. |
| **2. Network Attached Storage (NFS)** | Server NAS terpusat (seperti TrueNAS / Synology) yang di-mount via protokol NFSv4 melalui jaringan. | **Pod Bebas Pindah Node.** Mendukung mode `ReadWriteMany (RWX)` untuk banyak pod sekaligus. | Menjadi **Single Point of Failure (SPOF)**. Throughput terbatas oleh kapasitas kabel kartu jaringan server NFS dan rawan *file lock contention*. |
| **3. Distributed Block Storage (SAN / Ceph / Longhorn)** | Data direplikasi secara otomatis ke 3 disk fisik di 3 server berbeda melalui jaringan berkecepatan tinggi. | **High Availability Sejati.** Pod bisa pindah ke node lain dan data otomatis tersinkronisasi. | Mengonsumsi CPU, RAM, dan bandwidth jaringan cluster yang cukup besar untuk proses replikasi blok. |

> [!IMPORTANT]
> **Parameter Wajib StorageClass Local PV: `WaitForFirstConsumer`**
> Saat mendefinisikan StorageClass untuk Local PV di Kubernetes, **WAJIB** menggunakan:
> ```yaml
> volumeBindingMode: WaitForFirstConsumer
> ```
> Opsi ini menunda pembuatan volume sampai Pod benar-benar dijadwalkan oleh scheduler ke sebuah node. Jika menggunakan `Immediate`, volume bisa saja dibuat di Server A, sementara Pod dijadwalkan di Server B, sehingga Pod akan mengalami status **Pending selamanya** karena tidak bisa mengakses disk Server A!

---

## 4. Pemantauan Kesehatan Perangkat Keras (Hardware Monitoring) & Kapasitas

Di lingkungan bare-metal, pod Kubernetes yang berstatus `Running` tidak membuktikan bahwa hardware fisik server dalam kondisi prima.

### 4.1 Parameter Kritis Pemantauan Server Bare-Metal
1. **Out-of-Band Management (IPMI / iDRAC / iLO):**
   - Port jaringan independen di belakang motherboard server yang memiliki prosesor kecil sendiri. Memungkinkan SRE menyalakan, mematikan, atau mengakses terminal BIOS server dari jarak jauh meskipun sistem operasi Linux utamanya mengalami *Kernel Panic* total.
2. **Kesehatan Hard Disk via SMART Metrics (`node_disk_...`):**
   - Pantau indikator kegagalan mekanik disk: `Reallocated_Sector_Ct` (sektor bad yang dipindahkan) dan `Current_Pending_Sector`. Jika angka ini mulai naik di atas 0, segera jadwalkan penggantian hard disk fisik sebelum data korup!
3. **Peringatan Memori ECC (Error-Correcting Code RAM):**
   - Server kelas enterprise menggunakan RAM ECC yang mampu memperbaiki kesalahan bit memori akibat radiasi kosmik secara otomatis (*Correctable Single-Bit Errors*). Namun jika terjadi *Uncorrectable Multi-Bit Error*, kernel Linux akan langsung membeku (*hang*) untuk mencegah kerusakan data transaksi.

---

### 4.2 Rumus Matematika Perencanaan Kapasitas Klaster (N+1 Capacity Headroom)

Klaster bare-metal produksi **TIDAK BOLEH** dijalankan pada utilisasi 95%! SRE wajib menyediakan ruang cadangan (*headroom*) dengan rumus **Toleransi Kegagalan N+1**:

$$\text{Kapasitas Aman Maksimal} = \frac{N - 1}{N} \times \text{Total Kapasitas Klaster}$$

**Contoh Perhitungan pada Klaster 4 Node Bare-Metal:**
- Anda memiliki 4 server fisik (masing-masing 32 Core CPU dan 128 GB RAM).
- Total Kapasitas = $4 \times 128 \text{ GB} = 512 \text{ GB RAM}$.
- Jika 1 server fisik terbakar atau harus dimatikan untuk upgrade kernel, tersisa 3 server ($3 \times 128 = 384 \text{ GB RAM}$).
- Artinya, beban kerja normal harian seluruh Pod di klaster **TIDAK BOLEH MELEBIHI 384 GB RAM (Maksimal 75% dari total kapasitas 4 node)!** Jika utilisasi normal mencapai 90%, maka begitu 1 server mati, sisa 3 server akan mengalami kaskade kegagalan (*cascading failure*) akibat kehabisan memori serentak!

---

## 5. Tata Kelola Akses Bastion Jump Host & Prosedur Break-Glass

### 5.1 Arsitektur Bastion Host yang Diperkeras (Hardened Bastion)
Server worker dan master Kubernetes di data center bare-metal **DILARANG** memiliki IP publik langsung. Seluruh akses administratif wajib melewati satu pintu gerbang tunggal bernama **Bastion / Jump Host**:
- **Hanya Menerima SSH Key-Based Auth:** Parameter `PasswordAuthentication no` dan `PermitRootLogin no` aktif di `/etc/ssh/sshd_config`.
- **Multi-Factor Authentication (MFA):** Wajib memasukkan kode TOTP 6-digit (Google Authenticator) atau kunci fisik YubiKey saat login.
- **Audit Logging Terekam:** Menggunakan utilitas `auditd` atau *session recording* (seperti Teleport / `tlog`) untuk merekam setiap penekanan tombol (*keystroke*) engineer selama berada di dalam server.

---

### 5.2 Aturan Baku "No Snowflake": Ansible vs Manual SSH Break-Glass

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ATURAN TATA KELOLA PERUBAHAN HOST                        │
│                                                                             │
│  [1. KONDISI NORMAL (99.9% Perubahan Sistem)]                               │
│   Developer / SRE ──> Commit Playbook di Git ──> Pipeline CI/CD ──> Ansible  │
│   (Idempoten, Tercatat di Git, Diuji Otomatis, Tanpa Akses SSH Manual!)     │
│                                                                             │
│  [2. KONDISI DARURAT P1 (Break-Glass Protocol)]                             │
│   Hanya saat insiden besar di mana Ansible runner terputus koneksinya:      │
│   1. Minta izin darurat ke Incident Commander (IC).                         │
│   2. Buka brankas kunci akses sementara (Time-bounded credential).          │
│   3. Lakukan intervensi manual minimal untuk menghidupkan jaringan/node.    │
│   4. Setelah insiden selesai: Tulis perubahan ke Ansible Playbook agar      │
│      konfigurasi server kembali sinkron dengan Desired State di Git!        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **VIP MetalLB Tidak Merespons dari Jaringan Luar** | 1. Node leader speaker mati dan switch ToR memegang *Stale ARP*.<br>2. IP pool MetalLB bentrok (*IP collision*) dengan IP yang dibagikan oleh server DHCP router kantor. | 1. Cek dari host luar: `arp -a | grep <IP_VIP>` untuk melihat MAC address aktif.<br>2. Jalankan `arping -I eth0 <IP_VIP>` dari node Kubernetes untuk memaksa pembaruan tabel ARP switch ToR. |
| **Kueri Data Database Lambat tapi CPU Server Hanya 10%** | Hard disk SSD mengalami *I/O Bottleneck* (antrean saturasi I/O) atau kabel SATA/SAS mengalami kerusakan transmisi (*CRC Error Count* naik). | Jalankan perintah `iostat -xz 1` di host dan periksa kolom `%util` serta `await`. Jika `%util` mendekati 100% dan `await > 20ms`, disk fisik mengalami saturasi parah. |
| **Pod Postgres Berstatus `Pending` dengan Event `FailedScheduling`** | Penggunaan StorageClass Local PV tanpa setting `volumeBindingMode: WaitForFirstConsumer` menyebabkan PV terikat di node yang tidak memiliki resource CPU cukup. | Ubah definisi StorageClass menjadi `volumeBindingMode: WaitForFirstConsumer`, hapus PVC yang macet, dan biarkan scheduler menempatkan pod dan disk di node yang sama. |

---

## 7. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. DIAGNOSA JARINGAN FISIK, ARP, DAN MTU DI WORKER NODE
# ==============================================================================

# Menguji apakah ada paket yang dibuang akibat ketidakcocokan MTU (Jumbo Frames)
# (-M do melarang fragmentasi paket, -s 8972 byte payload + 28 byte header = 9000 MTU)
ping -M do -s 8972 <STORAGE_SERVER_IP>

# Mengirim paket Gratuitous ARP untuk menyegarkan cache ARP switch secara paksa
arping -c 3 -U -I eth0 192.168.30.100

# Memeriksa apakah ada packet drop atau CRC error pada kartu jaringan fisik
ip -s link show eth0

# ==============================================================================
# 2. DIAGNOSA KESEHATAN HARDWARE & STORAGE LOKAL
# ==============================================================================

# Memeriksa atribut kesehatan disk fisik NVMe/SSD via SMART (Cari reallocated sector)
smartctl -H /dev/nvme0n1
smartctl -A /dev/nvme0n1

# Memeriksa antrean I/O disk secara real-time per detik
iostat -xz 1 5

# Memeriksa status sensor suhu dan voltase power supply server fisik via IPMI
ipmitool sdr type Temperature
ipmitool sensor get "Power Supply"
```

---

## 8. Evaluasi Pemahaman Mandiri

### 8.1 Pilihan Ganda (3 Soal)

#### Soal 1
Ketika sebuah server worker fisik pada klaster k3s bare-metal on-premises mati listrik secara mendadak, mengapa pengguna di luar klaster sempat tidak bisa mengakses layanan web selama beberapa menit padahal MetalLB sudah memindahkan IP VIP ke worker node lain yang masih hidup?
- A. Karena Kubernetes melarang pemindahan alamat IP pada hari kerja.
- B. Karena switch jaringan fisik (*Top-of-Rack Switch*) atau router kantor masih menyimpan pemetaan lama antara IP VIP dan MAC address server yang mati di dalam **ARP Cache (*Stale ARP Cache*)**, sehingga paket data terus nyasar ke server yang mati sampai tabel ARP tersebut kedaluwarsa atau diperbarui oleh paket *Gratuitous ARP*.
- C. Karena CPU di server yang baru membutuhkan waktu 1 jam untuk memanas.
- D. Karena container Docker harus dikompilasi ulang dari bahasa C.

#### Soal 2
Mengapa parameter `volumeBindingMode: WaitForFirstConsumer` sangat diwajibkan saat membuat `StorageClass` untuk **Local Persistent Volume (Local PV)** di Kubernetes?
- A. Agar pod otomatis mendapatkan kuota penyimpanan tak terbatas dari internet.
- B. Untuk menunda pemilihan disk fisik di node lokal sampai pod benar-benar dijadwalkan oleh scheduler ke node yang memiliki kapasitas CPU/RAM yang cukup, sehingga volume disk dan pod dijamin berada di server fisik yang sama.
- C. Untuk menghapus data secara otomatis saat pengguna logout dari aplikasi web.
- D. Karena parameter tersebut diwajibkan oleh undang-undang perbankan.

#### Soal 3
Dalam prinsip tata kelola infrastruktur SRE *"No Snowflake"*, apa yang **WAJIB** dilakukan seorang engineer setelah ia terpaksa melakukan intervensi manual via SSH (*Break-Glass*) untuk menyelamatkan server produksi yang putus jaringan saat insiden darurat?
- A. Menghapus log sistem `/var/log` agar jejak intervensi manualnya tidak diketahui pimpinan.
- B. Mematikan server tersebut secara permanen.
- C. Segera mendokumentasikan perubahan tersebut dan memasukkannya ke dalam kode **Ansible Playbook / OpenTofu** di repositori Git, sehingga *Desired State* di Git kembali konsisten dengan kondisi aktual server fisik.
- D. Mengubah kata sandi root server tanpa memberi tahu rekan tim lain.

---

### 8.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Perusahaan Anda memiliki klaster bare-metal yang terdiri dari **3 server fisik** (Node A, Node B, dan Node C). Masing-masing server memiliki spesifikasi: **64 Core CPU dan 256 GB RAM**.
Total kapasitas gabungan klaster adalah 192 Core CPU dan 768 GB RAM.

Saat ini, seluruh pod aplikasi di klaster mengonsumsi rata-rata **160 Core CPU dan 640 GB RAM** (Utilisasi rata-rata klaster berada di angka **83%**).

Pada hari Selasa pukul 14:00, catu daya listrik (*Power Supply Unit*) pada Node C meledak dan server tersebut mati seketika.

**Pertanyaan & Tugas Anda sebagai SRE:**
1. Hitunglah secara matematis: apa yang terjadi pada Node A dan Node B sesaat setelah Node C mati? Apakah sisa kapasitas Node A dan Node B sanggup menampung beban kerja dari Node C?
2. Berapakah batas kapasitas maksimal (*N+1 Capacity Headroom*) yang seharusnya diizinkan untuk klaster 3 node tersebut agar tahan terhadap matinya 1 server fisik?
3. Sebutkan 2 tindakan mitigasi darurat yang harus diambil oleh On-Call SRE untuk menyelamatkan layanan utama dari pemadaman kaskade (*Cascading OOMKilled*)!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Di jaringan fisik Layer 2, pengiriman paket mengandalkan tabel ARP (*IP-to-MAC mapping*). Jika perangkat perantara lambat memperbarui tabel ARP, lalu lintas jaringan akan terus terkirim ke kartu jaringan mesin yang sudah mati.
2. **B.** Mode binding default (`Immediate`) mengalokasikan disk sebelum pod dijadwalkan. Jika volume dialokasikan di Node A tetapi scheduler menempatkan Pod di Node B karena Node A penuh, Pod tidak akan pernah bisa berjalan.
3. **C.** Menjaga konfigurasi server agar tidak menjadi *"Server Snowflake"* (server unik yang konfigurasinya berbeda dari kode Git) adalah pilar utama Configuration as Code. Setiap perbaikan manual wajib diabadikan ke playbook Ansible.

#### Solusi Soal Kasus Nyata:
1. **Kalkulasi Kegagalan Kaskade (Cascading Failure):**
   - Sisa kapasitas Node A + Node B = $2 \times 256 \text{ GB} = \mathbf{512 \text{ GB RAM}}$.
   - Beban kerja pod yang aktif saat ini = $\mathbf{640 \text{ GB RAM}}$.
   - **Terjadi Defisit Kapasitas:** Beban kerja (640 GB) melampaui kapasitas fisik yang tersisa (512 GB) sebesar **128 GB RAM**!
   - **Dampak Kritis:** Kubernetes scheduler akan mencoba menjadwalkan ulang pod dari Node C ke Node A dan B. Akibatnya, Node A dan B akan kehabisan memori (*Memory Pressure*), kernel Linux memicu **OOM Killer massal**, dan seluruh klaster akan mengalami **Pemadaman Total (Total Cluster Outage)**!
2. **Batas Kapasitas Aman N+1:**
   - Untuk $N = 3$, kapasitas aman maksimal adalah:
     $$\frac{3 - 1}{3} = \frac{2}{3} \approx 66{,}6\% \text{ dari total kapasitas}$$
   - Maksimal pemakaian RAM aman untuk 3 node = $66{,}6\% \times 768 \text{ GB} = \mathbf{512 \text{ GB RAM}}$. Beban 640 GB jelas melanggar batas keselamatan N+1.
3. **Dua Tindakan Mitigasi Darurat SRE:**
   - **Tindakan 1 (Penurunan Beban Kerja Non-Kritis / Shedding Load):** Segera matikan atau turunkan replika pod non-esensial (*scale down to 0*) seperti pod reporting, analytics batch job, staging workloads, atau worker background yang tidak langsung melayani transaksi checkout pelanggan.
   - **Tindakan 2 (Enforce Resource Limits & PriorityClass):** Pastikan pod Tier-1 (Payment/Checkout) memiliki `PriorityClass: high-priority` dan PodDisruptionBudget yang ketat agar Kubernetes mengorbankan pod prioritas rendah saat memori menipis.

---

## 9. Langkah Selanjutnya & Kaitan Modul

Setelah memahami arsitektur jaringan, penyimpanan, perangkat keras, dan akses bastion on-premises:
- Lanjutkan ke [02 — Backup, Restore, Upgrade, Security, dan Disaster Recovery](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.2-production-onprem/02-backup-restore-upgrade-security-dr.md) untuk mempelajari strategi pencadangan 3-lapis dengan Velero dan snapshot etcd, prosedur upgrade rolling k3s tanpa downtime, dan uji pemulihan bencana (*Disaster Recovery Drill*).
- Praktikkan di [LAB-01 — Velero dan etcd Backup Restore](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.2-production-onprem/lab/LAB-01-velero-etcd-backup-restore.md).
