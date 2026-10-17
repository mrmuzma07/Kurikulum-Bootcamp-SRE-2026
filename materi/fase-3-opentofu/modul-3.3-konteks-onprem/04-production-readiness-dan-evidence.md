# 04 — Production Readiness dan Evidence: Checklist Pre-Flight, Chain of Custody, dan Manajemen Drift On-Premises

> **Ringkasan:** Di dunia *Site Reliability Engineering* (SRE), munculnya tulisan `Apply complete! Resources: 3 added, 0 changed, 0 destroyed` pada terminal **BUKANLAH** bukti bahwa infrastruktur telah siap melayani lalu lintas produksi (*traffic*). Di lingkungan *on-premises* (bare-metal atau private data center), keberhasilan pembuatan Virtual Machine hanyalah satu mata rantai kecil. Dokumen ini merumuskan metodologi kesiapan produksi (*Production Readiness Review*), menyusun rantai bukti audit (*Immutable Evidence Chain*), menetapkan kondisi henti darurat (*Stop Conditions*), serta menyajikan Standard Operating Procedure (SOP) penanganan *drift* fisik tanpa merusak kestabilan klaster k3s.

---

## 1. Analogi Pemula: Checklist Pre-Flight Pilot & Kotak Hitam (Black Box)

Bagi seorang pemula, godaan terbesar setelah menjalankan `tofu apply` adalah langsung membuka browser dan berseru: *"Sudah selesai, server sudah nyala!"*.

Mari kita bandingkan profesi SRE dengan kapten pilot pesawat terbang komersial:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI: PRE-FLIGHT CHECKLIST vs "ASAL TERBANG"             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PENDEKATAN AMATIR (ASAL JALAN) ]                                         │
│  - Masuk ke kokpit, putar kunci kontak mesin jet.                           │
│  - Mesin berbunyi "Wuuusshhh", lampu indikator menyala hijau.               │
│  - Langsung injak pedal gas di landasan pacu!                               │
│  HASILNYA:                                                                  │
│  Pesawat jatuh 10 menit kemudian karena hidrolik sayap bocor, sensor suhu   │
│  beku, atau bahan bakar bocor di tangki cadangan.                           │
│                                                                             │
│  [ PENDEKATAN SRE PROFESIONAL (CHECKLIST PRE-FLIGHT PILOT) ]                │
│  - Mesin menyala hanyalah *langkah awal*.                                   │
│  - Pilot dan Kopilot membuka buku saku Checklist:                           │
│    1. "Tekanan hidrolik elevator?" -> "Normal 3000 PSI."                    │
│    2. "Komputer navigasi GPS tersinkron dengan menara kontrol?" -> "Valid." │
│    3. "Bahan bakar cadangan cuaca buruk cukup 45 menit?" -> "Terkonfirmasi."│
│    4. "Black Box (Flight Data Recorder) aktif merekam?" -> "Merekam."       │
│  - HANYA SETELAH SELURUH CHECKLIST DICENTANG, izin lepas landas diminta!    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│  DI DUNIA SRE BARE-METAL / ON-PREMISES:                                     │
│  - `tofu apply` berhasil = Mesin jet menyala.                               │
│  - Host Readiness Gate & Network Audit = Checklist Pre-Flight.              │
│  - Immutable Evidence Chain = Black Box / Flight Data Recorder.             │
│                                                                             │
│  Jika terjadi insiden, Black Box (Evidence Chain) membuktikan bahwa         │
│  infrastruktur dirilis sesuai prosedur standar, bukan hasil spekulasi liar! │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Tiga Gerbang Kesiapan Produksi (The Three Production Readiness Gates)

Sebelum sebuah perubahan infrastruktur *on-prem* dideploy ke cluster produksi, perubahan tersebut wajib lolos dari **Tiga Gerbang Kesiapan (*Readiness Gates*)**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 TIGA GERBANG KESIAPAN PRODUKSI (READINESS GATES)             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ GERBANG 1: PROVIDER & STATE READINESS ]                                  │
│  - Version Pinning: Versi OpenTofu dan Provider terkunci di lockfile.       │
│  - State Isolation: Bucket MinIO S3 terpisah per environment (`prod/`).     │
│  - State Locking: Mekanisme distributed lock aktif mencegah split-brain.    │
│  - Recovery Point Objective (RPO): Backup state tersimpan sebelum apply.    │
│  - Commit Tracing: Commit Git SHA persis sama dengan kode yang di-review.   │
│                                     │                                       │
│                                     ▼ LULUS                                 │
│                                                                             │
│  [ GERBANG 2: PHYSICAL & NETWORK INFRASTRUCTURE READINESS ]                 │
│  - Hypervisor Headroom: Kapasitas RAM/CPU fisik hypervisor tersisa > 25%.   │
│  - Network & IPAM: Subnet VLAN, default gateway, dan IP statis terverifikasi│
│    tanpa ada tabrakan (*no ARP collision*).                                 │
│  - Split-Horizon DNS: Forward (A) dan Reverse (PTR) records terdaftar.      │
│  - MTU Consistency: Seluruh switch fisik dan vNIC menggunakan MTU seragam.  │
│  - Storage Durability: LUN SAN / NFS mount point siap dan lolos uji IOPS.   │
│                                     │                                       │
│                                     ▼ LULUS                                 │
│                                                                             │
│  [ GERBANG 3: PROCESS & GOVERNANCE READINESS ]                              │
│  - Blast Radius Control: Perubahan dibatasi (maksimal 1 node per siklus).   │
│  - Change Window: Eksekusi dilakukan di luar jam sibuk bisnis (*off-peak*). │
│  - 4-Eyes Principle: Kode HCL dan Plan di-review serta disetujui minimal    │
│    oleh satu orang Staff/Lead SRE.                                          │
│  - Fail-Closed Rollback Plan: Skenario pembatalan teruji jika terjadi macet. │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Matriks Audit: Network, IP, DNS, dan Storage On-Premises

Di cloud publik, Anda mendapatkan DHCP dinamis, Private DNS otomatis Route53, dan volume EBS dengan mudah. Di lingkungan *on-premises*, kelalaian konfigurasi sekecil apa pun akan memicu insiden sistemik.

Berikut adalah lembar kerja audit kesiapan fisik yang wajib diverifikasi sebelum eksekusi produksi:

| Lapisan Infrastruktur | Pertanyaan Audit SRE | Bukti Minimum yang Diperlukan | Sinyal Bahaya (*Reject Condition*) |
|---|---|---|---|
| **Management Network** | Apakah server CI/CD atau runner Ansible dapat menjangkau node via port 22? | Hasil `nc -zvw3 <IP> 22` bernilai `succeeded`. | Koneksi timeout, ditolak firewall, atau dialihkan ke host lain. |
| **Node Network (Inter-Pod)** | Apakah antar-node bisa saling kirim paket VXLAN (UDP 8472) dan API (TCP 6443)? | Validasi socket `nc -zvw3 -u <Node-IP> 8472`. | Paket di-drop oleh firewall fisik (Top-of-Rack Switch) atau UFW. |
| **IPAM (Alokasi IP)** | Siapa pemilik sah dari IP statis yang dialokasikan? Apakah ada duplikasi di jaringan? | Output `arping -c 2 -I <iface> <Target-IP>` nol respon sebelum VM up. | Ditemukan perangkat fisik lain (Cisco switch / printer) memakai IP yang sama! |
| **DNS Internal & Reverse** | Apakah DNS server lokal mampu me-resolve nama host secara bolak-balik? | Hasil `dig +short <FQDN>` dan `dig -x <Target-IP>` cocok sempurna. | Forward resolve sukses, tapi Reverse DNS (PTR) kosong (bikin lambat SSH & SSL)! |
| **Storage (Ceph/NFS/SAN)** | Apakah direktori `/var/lib/rancher/k3s` memiliki throughput IOPS cukup untuk write etcd? | Uji latensi fsync: `fio` atau `dd bs=4k count=1000 oflag=dsync`. | Latensi fsync disk > 10ms (Raft etcd akan mengalami timeout konstan). |
| **MTU & VLAN Bridging** | Apakah interface bridge Linux dan vNIC VM memiliki ukuran MTU yang selaras? | `ip link show` pada host hypervisor dan guest VM (standar: 1500). | Terjadi fragmentasi paket karena bridge MTU 1500 tapi vNIC VM diset 9000. |

---

## 4. Rantai Bukti Tak Terputuskan (The Immutable Evidence Chain)

Dalam tata kelola SRE modern, kepatuhan audit (*compliance*) mensyaratkan setiap perubahan infrastruktur memiliki **Rantai Jejak Bukti (*Chain of Custody*)**. 

Jika terjadi bencana (*outage*) pada pukul 03.00 pagi, tim investigasi harus bisa melacak asal-usul konfigurasi hingga ke baris kode Git pertama:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 RANTAI JEJAK BUKTI TAK TERPUTUSKAN (EVIDENCE CHAIN)         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ 1. Git Commit SHA ]            Contoh: commit 8f9b2c1...                 │
│         │                                                                   │
│         ▼                                                                   │
│  [ 2. Module Version Lock ]       Contoh: git tag v1.2.0                    │
│         │                                                                   │
│         ▼                                                                   │
│  [ 3. Provider Checksum Lock ]    Contoh: .terraform.lock.hcl (SHA256)      │
│         │                                                                   │
│         ▼                                                                   │
│  [ 4. Backend Key & Environment ] Contoh: minio://tfstate/prod/k3s.tfstate  │
│         │                                                                   │
│         ▼                                                                   │
│  [ 5. Plan Summary Artifact ]     Contoh: plan-summary.json (No secrets!)   │
│         │                                                                   │
│         ▼                                                                   │
│  [ 6. Change Approval Ticket ]    Contoh: JIRA-SRE-2026-442 (Approved Lead) │
│         │                                                                   │
│         ▼                                                                   │
│  [ 7. Apply Execution Log ]       Contoh: gitlab-ci/jobs/948210.log         │
│         │                                                                   │
│         ▼                                                                   │
│  [ 8. Host Readiness Evidence ]   Contoh: check-readiness-passed.log        │
│         │                                                                   │
│         ▼                                                                   │
│  [ 9. Ansible Hardening Log ]     Contoh: ansible-run-ok.json               │
│         │                                                                   │
│         ▼                                                                   │
│  [ 10. Cluster Health Evidence ]  Contoh: kubectl-get-nodes-ready.txt       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Format Berkas Rangkuman Bukti Audit (`evidence-summary.json`)

Berkas bukti audit harus disimpan pada sistem penyimpanan jangka panjang (*Cold Storage*) atau repositori artefak tanpa menyertakan kredensial:

```json
{
  "evidence_version": "1.0",
  "change_id": "CR-ONPREM-2026-10-17-01",
  "timestamp_utc": "2026-10-17T08:00:00Z",
  "executor": {
    "system": "GitLab CI Runner on-prem-runner-01",
    "triggered_by": "sre-operations@bootcamp.local",
    "git_commit": "8f9b2c1743a19e24f706e25776d6ec1438914619",
    "git_branch": "main"
  },
  "opentofu_metadata": {
    "engine_version": "1.8.2",
    "provider_locks": {
      "bpg/proxmox": "0.66.2 (h1:xG9y...)"
    },
    "backend_state_key": "s3://tofu-state-prod/clusters/k3s-core/terraform.tfstate",
    "actions": {
      "create": 3,
      "update": 0,
      "delete": 0,
      "replace": 0
    }
  },
  "readiness_verification": {
    "chrony_clock_skew_max_seconds": 0.00042,
    "swap_disabled_all_hosts": true,
    "ssh_key_auth_verified": true,
    "firewall_ports_open": [6443, 2379, 2380, 8472]
  },
  "runtime_health": {
    "k3s_version": "v1.31.1+k3s1",
    "nodes_total": 3,
    "nodes_ready": 3,
    "etcd_leader": "sre-k3s-cp-01"
  }
}
```

> [!WARNING]
> **Aturan Redaksi Bukti (Data Sanitization):**
> File rekaman bukti **DILARANG KERAS** memuat:
> 1. Raw binary file state (`terraform.tfstate`) yang berisi string terenkripsi atau nilai variabel sensitif.
> 2. Isi utuh output file plan (`tfplan`) tanpa disaring via `tofu show -json`.
> 3. Token bearer, password Proxmox, atau private key SSH.

---

## 5. Manajemen Drift On-Premises & Prosedur Pemulihan (Recovery)

*Drift* adalah kondisi di mana keadaan fisik infrastruktur (*real-world state*) menyimpang dari kode HCL yang tercatat di Git. Di lingkungan *on-premises*, drift jauh lebih sering terjadi karena engineer tergoda login ke GUI Proxmox atau SSH ke server untuk perbaikan darurat saat insiden.

### 5.1 Empat Kategori Drift di Lingkungan On-Premises

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       EMPAT KATEGORI DRIFT ON-PREMISES                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Hypervisor Drift                                                        │
│     Penyebab: Administrator menambah vCPU/RAM secara manual lewat GUI       │
│     vSphere/Proxmox saat server overload, tanpa mencatatnya di OpenTofu.     │
│                                                                             │
│  2. Network / IPAM Drift                                                    │
│     Penyebab: Tim Jaringan mengubah tag VLAN pada switch Top-of-Rack atau   │
│     mematikan port interface firewall secara mendadak.                      │
│                                                                             │
│  3. Guest OS Drift                                                          │
│     Penyebab: Engineer melakukan SSH langsung ke node k3s dan mengedit      │
│     `/etc/resolv.conf`, menyalakan swap, atau mengupdate package OS manual. │
│                                                                             │
│  4. State File Drift                                                        │
│     Penyebab: Perintah `tofu apply` terputus di tengah jalan karena koneksi │
│     jaringan terputus, sehingga state MinIO tertinggal dari kondisi aktual. │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 SOP 8 Langkah Rekonsiliasi Drift yang Aman

Jangan panik dan jangan langsung menjalankan `tofu apply` saat drift terdeteksi! Ikuti alur kerja disiplin berikut:

```text
[ LANGKAH 1: BEKUKAN PERUBAHAN (FREEZE) ]
  Hentikan seluruh pipeline CI/CD dan larang tim melakukan modifikasi manual.

[ LANGKAH 2: BACKUP STATE SAAT INI ]
  Unduh snapshot state dari MinIO S3 sebagai cadangan penyelamat:
  $ tofu state pull > /backup/state-pre-reconcile-$(date +%s).json

[ LANGKAH 3: INSPEKSI DRIFT SECARA READ-ONLY ]
  Jalankan refresh tanpa mengubah apa pun:
  $ tofu plan -refresh-only -out=drift.tfplan
  $ tofu show drift.tfplan

[ LANGKAH 4: KLASIFIKASIKAN AKAR DRIFT ]
  Tentukan apakah perubahan di lapangan itu:
  a. Perubahan yang sah (misal: RAM dinaikkan memang untuk mitigasi OOM).
  b. Perubahan ilegal / kecelakaan (misal: ada yang salah pencet di Proxmox).

[ LANGKAH 5: SESUAIKAN KEBENARAN ]
  - Jika perubahan sah: Update kode HCL di Git agar mencerminkan kapasitas baru!
  - Jika perubahan ilegal: Biarkan OpenTofu mengembalikan (revert) ke spek lama.

[ LANGKAH 6: SIMULASIKAN ULANG (DRY-RUN) ]
  Jalankan plan baru dan periksa apakah ada indikasi "must be replaced":
  $ tofu plan -out=reconcile.tfplan
  PASTIKAN TIDAK ADA NODE K3S YANG AKAN DI-DESTROY!

[ LANGKAH 7: EKSEKUSI DENGAN PERSETUJUAN DUA ORANG (4-EYES) ]
  Terapkan perubahan setelah mendapatkan lampu hijau dari Lead SRE:
  $ tofu apply reconcile.tfplan

[ LANGKAH 8: VALIDASI ULANG KESEHATAN KLASTER K3S ]
  Verifikasi etcd quorum dan status Pod setelah rekonsiliasi:
  $ kubectl get nodes -o wide
```

---

## 6. Stop Conditions: Kapan Harus Segera Membatalkan Perubahan?

Sebuah otomatisasi yang aman harus memiliki **Kondisi Henti (*Stop Conditions*)** atau tombol darurat (*kill-switch*). Jika salah satu kondisi di bawah ini terdeteksi pada tahapan apa pun, **BATALKAN PERUBAHAN SECARA TOTAL (FAIL-CLOSED)**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 TABEL STOP CONDITIONS (KONDISI HENTI DARURAT)               │
├───────────────────────────────┬─────────────────────────────────────────────┤
│ Kondisi yang Terdeteksi       │ Tindakan Wajib SRE                          │
├───────────────────────────────┼─────────────────────────────────────────────┤
│ 1. Plan menampilkan status    │ ABORT SEGERA! Review ulang kode HCL.        │
│    `must be replaced` pada    │ Pergantian node etcd aktif tanpa prosedur   │
│    node server k3s aktif.     │ cordon/drain memicu korupsi cluster state!  │
├───────────────────────────────┼─────────────────────────────────────────────┤
│ 2. Backend State MinIO gagal  │ STOP! Jangan jalankan `tofu apply` dengan   │
│    mengunci (*state lock*).   │ flag `-lock=false`! Ada proses lain yang    │
│                               │ sedang berjalan atau koneksi database down. │
├───────────────────────────────┼─────────────────────────────────────────────┤
│ 3. Waktu sistem (*clock*) node│ STOP! Jangan jalankan bootstrap k3s.        │
│    melompat > 100 milidetik.  │ Perbaiki layanan Chrony/NTP terlebih dahulu.│
├───────────────────────────────┼─────────────────────────────────────────────┤
│ 4. Terdeteksi IP address      │ STOP! Cari perangkat fisik yang membajak IP │
│    collision melalui `arping`.│ tersebut di switch lokal sebelum VM boot.   │
├───────────────────────────────┼─────────────────────────────────────────────┤
│ 5. Perubahan Git Commit SHA   │ ABORT! Pipeline mengeksekusi kode yang      │
│    tidak cocok dengan tiket.  │ belum di-approve di merge request!          │
└───────────────────────────────┴─────────────────────────────────────────────┘
```

---

## 7. Pola Kegagalan (Failure Modes) Lapangan & Diagnosa

Berikut adalah 4 skenario kegagalan operasional yang kerap menimpa tim SRE di private cloud:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   STUDI KASUS KEGAGALAN KESIAPAN PRODUKSI                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "PHANTOM DRIFT" KARTU JARINGAN / MAC ADDRESS ]                  │
│  Gejala: Setiap kali `tofu plan` dijalankan, OpenTofu selalu mendeteksi     │
│          perubahan pada MAC address kartu jaringan virtual VM, padahal      │
│          tidak ada baris kode yang diubah.                                  │
│  Akar Masalah:                                                              │
│  - Provider hypervisor menghasilkan MAC address acak secara dinamis jika    │
│    tidak didefinisikan secara eksplisit di HCL. Hal ini menyebabkan state   │
│    menganggap kartu jaringan diganti baru pada setiap plan!                 │
│  Solusi SRE:                                                                │
│  1. Tentukan MAC address secara deterministik di variabel IPAM, ATAU        │
│  2. Tambahkan blok pengabaian siklus hidup pada resource VM:                │
│     lifecycle { ignore_changes = [ network_interface[0].mac_address ] }     │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: ACCIDENTAL MASS REPLACEMENT KARENA GANTI SUBNET ]               │
│  Gejala: Engineer mengubah variabel `subnet_cidr` dari `/24` ke `/23`.      │
│          Hasil plan: "Plan: 3 to add, 0 to change, 3 to destroy".           │
│  Akar Masalah:                                                              │
│  - Parameter jaringan pada modul VM bersifat *immutable* (tidak bisa diubah │
│    saat VM hidup). Provider memutuskan untuk memusnahkan 3 VM lama dan      │
│    membuat 3 VM baru. Seluruh data etcd produksi terancam lenyap!           │
│  Solusi SRE:                                                                │
│  1. Wajib pasang `prevent_destroy = true` di seluruh resource VM produksi:  │
│     lifecycle { prevent_destroy = true }                                    │
│  2. Buat VM baru secara bertahap (Rolling expansion), gabungkan ke k3s,     │
│     pindahkan beban kerja, baru musnahkan VM lama secara sadar.             │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: SPLIT-HORIZON DNS GAGAL MERESOLVE PRIVATE REGISTRY ]            │
│  Gejala: Node k3s berhasil dibuat dan siap, tapi seluruh Pod crashloop      │
│          dengan status `ErrImagePull` atau `ImagePullBackOff`.              │
│  Akar Masalah:                                                              │
│  - Node VM diarahkan ke DNS publik (`8.8.8.8`) oleh DHCP, sehingga tidak    │
│    bisa mengenali FQDN GitLab Registry lokal (`registry.internal.corp`).    │
│  Solusi SRE:                                                                │
│  1. Masukkan pemeriksaan DNS internal ke dalam Host Readiness Gate:         │
│     dig @local-dns-ip registry.internal.corp +short                         │
│  2. Pastikan konfigurasi Cloud-Init menyuntikkan DNS server lokal data      │
│     center sebagai resolver prioritas utama.                                │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 4: KEBOCORAN KREDENSIAL PROXMOX DI ARTEFAK PIPELINE ]             │
│  Gejala: Token API Proxmox (`PVEAPIToken=root@pam!...`) tercetak di berkas  │
│          log pipeline karena perintah `tofu plan` dijalankan tanpa masking. │
│  Akar Masalah:                                                              │
│  - Variabel token didefinisikan di command line `-var` alih-alih melalui    │
│    variabel lingkungan `TF_VAR_` atau disamarkan oleh CI runner.            │
│  Solusi SRE:                                                                │
│  1. Cabut (*revoke*) token yang bocor di hypervisor dalam kurun 5 menit!    │
│  2. Buat token baru dengan hak akses terbatas (*Least Privilege*).          │
│  3. Gunakan mekanisme penyuntikan variabel lingkungan rahasia:              │
│     export TF_VAR_pm_api_token_secret="$CI_SECRET_PROXMOX_TOKEN"           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Ringkasan Perintah Penting (SRE Cheatsheet)

Kumpulan perintah esensial untuk memeriksa kesiapan dan jejak bukti di terminal:

```bash
# 1. Jalankan deteksi drift secara aman tanpa memodifikasi apa pun
tofu plan -refresh-only -detailed-exitcode

# 2. Periksa apakah ada resource yang ditandai untuk dihancurkan/diganti
tofu plan -out=tfplan.binary
tofu show -json tfplan.binary | jq -r '.resource_changes[] | select(.change.actions[] | contains("delete")) | .address'

# 3. Uji duplikasi alamat IP di jaringan lokal sebelum VM dinyalakan
arping -c 3 -I eth0 192.168.10.11

# 4. Validasi forward dan reverse DNS untuk seluruh node cluster
dig +short sre-k3s-cp-01.internal.corp
dig -x 192.168.10.11 +short

# 5. Uji latensi write disk storage lokal untuk kestabilan etcd Raft
fio --name=etcd-bench --ioengine=sync --fdatasync=1 --rw=write --bs=4k --size=100m --filename=/var/lib/rancher/k3s/bench.tmp

# 6. Ekstrak ringkasan perubahan untuk dilampirkan ke tiket persetujuan
tofu show -json tfplan.binary | jq '{create: [ .resource_changes[] | select(.change.actions[0] == "create") | .address ], delete: [ .resource_changes[] | select(.change.actions[0] == "delete") | .address ]}'
```

---

## 9. Pertanyaan Reflektif & Uji Pemahaman Mandiri

Evaluasi kesiapan operasional Anda sebelum menghadapi skenario lab nyata:

### Soal 1 (Pilihan Ganda)
Mengapa output `tofu plan -refresh-only` merupakan langkah pertama yang wajib diambil saat mendeteksi adanya anomali hardware fisik di hypervisor?
- A. Karena perintah ini langsung menghapus objek yang rusak di hypervisor.
- B. Karena perintah ini membaca keadaan fisik aktual (*real-world state*) dan membandingkannya dengan state OpenTofu tanpa menerapkan perubahan destruktif pada infrastruktur.
- C. Karena perintah ini otomatis mengupdate kode HCL di Git.
- D. Karena perintah ini mengunci port SSH agar administrator lain tidak bisa login.

*Kunci Jawaban:* **B**. `tofu plan -refresh-only` adalah operasi inspeksi yang aman (*read-only*) untuk mendiagnosa perbedaan antara keadaan riil dan rekaman state tanpa memicu aksi eksekusi.

### Soal 2 (Pilihan Ganda)
Saat melakukan audit kesiapan jaringan (*Network Readiness Gate*), mengapa Reverse DNS (PTR Record) tetap wajib dikonfigurasi pada IP node Kubernetes di private data center?
- A. Agar k3s bisa mengunduh file binary tanpa koneksi internet.
- B. Karena k3s otomatis mati jika PTR record tidak ditemukan.
- C. Karena ketiadaan PTR record dapat memicu jeda waktu tunda (*timeout lag*) pada handshake SSH, kegagalan validasi sertifikat mTLS internal, dan hambatan audit jaringan.
- D. Agar IP address node bisa berubah otomatis setiap kali reboot.

*Kunci Jawaban:* **C**. Banyak daemon Linux (termasuk SSH dan komponen keamanan TLS) melakukan reverse DNS lookup saat inisialisasi koneksi. Ketiadaan PTR record memicu timeout DNS hingga 5–10 detik pada setiap sesi baru.

### Soal 3 (Pilihan Ganda)
Sebuah pipeline CI/CD produksi gagal di tahap `tofu apply` karena koneksi jaringan putus saat node kedua sedang dibuat. Tindakan paling tepat yang harus diambil oleh seorang SRE adalah:
- A. Langsung menghapus file `terraform.tfstate` dari bucket MinIO agar bisa mulai dari nol.
- B. Menjalankan `tofu destroy --auto-approve` dari laptop lokal.
- C. Mengunduh backup state terakhir, menginspeksi status lock backend, melakukan `tofu plan -refresh-only` untuk menyelaraskan status VM yang sudah terlanjur dibuat, baru merencanakan apply ulang.
- D. Login manual ke hypervisor dan mematikan seluruh VM yang ada.

*Kunci Jawaban:* **C**. Menghapus state atau melakukan destroy membabi-buta akan meninggalkan *orphaned resources* (objek liar) di hypervisor dan memperparah kekacauan kontrol state.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Perusahaan Anda memberlakukan aturan kepatuhan SOC2 / ISO 27001 di mana setiap perubahan pada cluster Kubernetes produksi harus dapat diaudit oleh tim eksternal. Jelaskan mengapa mengarsipkan commit Git saja TIDAK CUKUP sebagai bukti rilis infrastruktur, dan sebutkan 3 komponen tambahan apa yang wajib dilampirkan dalam *Immutable Evidence Chain*!

*Panduan Solusi:*
Komit Git hanya membuktikan *keinginan tertulis* (*desired state*), bukan bukti bahwa kode tersebut benar-benar dieksekusi pada target fisik yang tepat pada waktu yang sah.

Tiga komponen tambahan yang wajib dilampirkan:
1. **Lockfile & Provider Checksums (`.terraform.lock.hcl`):** Membuktikan integritas binary provider yang mengeksekusi perubahan tidak disisipi malware.
2. **Execution Plan & Approval Ticket Link:** Membuktikan bahwa tindakan penambahan/pengubahan sumber daya telah ditinjau dan disetujui oleh pihak berwenang sebelum eksekusi.
3. **Runtime Health & Verification Telemetry:** Bukti verifikasi setelah eksekusi (misal: output `kubectl get nodes` yang menunjukkan node berstatus `Ready` dan quorum etcd stabil), membuktikan bahwa sistem tidak hanya "dibuat" tetapi benar-benar berfungsi melayani trafik secara sehat.

---

## 10. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Provider On-Premises dan Boundary Ownership](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/01-provider-onprem-dan-boundary.md) — Memahami pemisahan tanggung jawab provider on-prem.
  - [02 — Simulasi Lokal Docker & Mock Provider](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/02-simulasi-lokal-docker-libvirt-mock.md) — Melatih simulasi arsitektur di workstation lokal.
  - [03 — OpenTofu, Ansible, dan k3s Handoff](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/03-opentofu-ansible-k3s-handoff.md) — Menjalankan alih tugas metadata dan 8 gerbang kesiapan host.
- **Praktik Laboratorium:**
  - [LAB-01 — Simulasi Provider dan Boundary](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/lab/LAB-01-simulasi-provider-dan-boundary.md) — Hands-on pembuktian modul pada host simulasi.
  - [LAB-02 — Handoff ke Ansible dan k3s](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/lab/LAB-02-handoff-ke-ansible-dan-k3s.md) — Menguji alur transfer metadata inventaris dan readiness gate secara nyata.
- **Tahap Berikutnya:**
  - [Fase 4 — Ansible](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/) — Memasuki fase otomatisasi konfigurasi sistem operasi dan deployment k3s.
