# 02 — Backup, Restore, Upgrade, Security, dan Disaster Recovery

> **Ringkasan:** Merancang strategi pencadangan 3-lapis (**Kubernetes Objects, Persistent Volumes, dan etcd State**), memahami kalkulasi **RPO (Recovery Point Objective)** dan **RTO (Recovery Time Objective)**, mengeksekusi prosedur **Rolling Upgrade k3s** tanpa downtime menggunakan Ansible dan *PodDisruptionBudget*, memperkuat keamanan rantai pasok container menggunakan **Trivy**, serta menguji rencana pemulihan bencana (**Disaster Recovery Drill**).

---

## 1. Tiga Lapisan Batas Pencadangan (3-Tier Backup Boundary)

### 1.1 Analogi Dunia Nyata: "Brankas Tahan Api, Rekaman Akta Tanah, dan Surat Wasiat Kunci"
Ketika mengasuransikan sebuah gedung kantor dari bencana kebakaran:
1. **Denah Arsitektur Gedung (*Kubernetes Objects*):** Menyimpan cetak biru tata letak ruangan, letak stopkontak, dan pintu darurat. Jika gedung roboh, denah ini bisa digambar ulang dari arsip GitOps.
2. **Isi Brankas Uang & Dokumen Nasabah (*Persistent Volumes & Database Data*):** Ini adalah data nyata yang tidak bisa ditebak dari denah! Jika hard disk database PostgreSQL terbakar, data saldo nasabah akan musnah selamanya kecuali ada salinan fisik yang disimpan di brankas luar kota (**MinIO/S3 Off-site Backup**).
3. **Kunci Master & Hak Milik Notaris (*etcd State*):** Catatan internal yang menyimpan siapa pemilik setiap ruangan dan izin akses yang berlaku saat ini.

Dalam arsitektur Kubernetes bare-metal produksi, SRE **wajib** membedakan ketiga lapisan pencadangan ini secara tegas:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TIGA LAPISAN PENCADANGAN KUBERNETES                      │
├──────────────────────────┬──────────────────────────┬───────────────────────┤
│ Lapisan Pencadangan      │ Cakupan Data & Objek     │ Alat & Mekanisme SRE  │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│ **1. K8s Objects**       │ Deployment, Service,     │ **GitOps (ArgoCD)**   │
│    (Metadata Deklaratif) │ ConfigMap, RBAC, Ingress,│ dan **Velero Backup** │
│                          │ NetworkPolicy            │ (simpan JSON manifes) │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│ **2. Application Data**  │ Data fisik di hard disk: │ **Velero + Kopia**    │
│    (Persistent Volumes)  │ tabel PostgreSQL, bucket │ atau snapshot CSI ke  │
│                          │ MinIO, file upload user  │ MinIO S3 terpisah     │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│ **3. Cluster State**     │ State internal k3s:      │ **k3s etcd snapshot** │
│    (Control-Plane etcd)  │ token service-account,   │ (`k3s etcd-snapshot`) │
│                          │ leasing IP, node status  │ lokal & S3 remote     │
└──────────────────────────┴──────────────────────────┴───────────────────────┘
```

> [!WARNING]
> **Kebenaran Pahit SRE: Status `Backup Completed` Bukan Berarti Data Bisa Dipulihkan!**
> Melihat status backup berstatus `Completed` di Velero hanya membuktikan bahwa proses pembacaan file selesai tanpa crash. Status tersebut **BUKAN BUKTI** bahwa database di dalamnya konsisten atau bisa di-restore! Jika saat backup berjalan database sedang sibuk menulis transaksi tanpa mekanisme *quiescing / lock*, data yang ter-backup bisa berstatus **korup (*corrupted state*)**. Satu-satunya bukti validitas backup adalah **Uji Coba Pemulihan Nyata (*Restore Drill*)** ke klaster lab secara berkala!

---

## 2. Pemulihan Bencana (Disaster Recovery), RPO, dan RTO

### 2.1 Menghitung RPO dan RTO secara Matematis

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       GRAFIK METRIK RPO DAN RTO                             │
│                                                                             │
│  [ Backup Terakhir ]               [ INSIDEN PUSAT ]    [ SISTEM PULIH ]    │
│  Pukul 10:00                       Pukul 14:00          Pukul 15:30         │
│         │                                │                    │             │
│         ◄─────────── RPO ────────────────►                    │             │
│                 (Maksimal 4 Jam)                              │             │
│                                          ◄─────── RTO ────────►             │
│                                                 (90 Menit)                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

1. **RPO (Recovery Point Objective):**
   - **Berapa banyak data transaksi maksimal yang boleh hilang saat bencana terjadi?**
   - Dihitung dari selisih waktu antara **titik backup terakhir** dan **waktu insiden**. Jika Anda hanya mem-backup database sekali sehari pada pukul 00:00 malam, lalu server meledak pukul 14:00 siang, maka **seluruh transaksi bisnis selama 14 jam hilang permanen (RPO = 14 Jam)**! Untuk layanan perbankan, RPO biasanya dipatok < 5 menit menggunakan teknik *continuous WAL archiving*.
2. **RTO (Recovery Time Objective):**
   - **Berapa lama waktu maksimal yang dibutuhkan tim dari deklarasi bencana sampai layanan aktif kembali melayani transaksi pengguna?**
   - Dihitung dari waktu server mati hingga seluruh verifikasi acceptance criteria terpenuhi (misal: RTO = 2 jam).

---

### 2.2 Alur Pemulihan Total dari Nol (End-to-End Recovery Path)

Jika seluruh server fisik di data center terbakar atau terserang ransomware, rekonstruksi sistem dilakukan secara hierarkis sesuai batas modul kurikulum:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. OpenTofu (Fase 3): Provisioning VM, virtual disk, dan network underlay   │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. Ansible (Fase 4): OS hardening, instalasi k3s cluster & MetalLB L2      │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. etcd Snapshot Restore: Pulihkan state k3s control-plane jika diperlukan  │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. Velero Restore: Pulihkan data volume PVC (Postgres/MinIO) dari S3 remote │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. ArgoCD Rehydration (Fase 6): Tarik seluruh manifes aplikasi dari Git     │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. Observabilitas & Verifikasi SLO (Fase 7): Validasi metrik & Error Budget │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Prosedur Rolling Upgrade k3s & Patching OS Tanpa Downtime

Melakukan upgrade versi k3s atau pembaruan kernel Linux di lingkungan produksi bare-metal **DILARANG KERAS** mematikan seluruh server secara serentak. SRE menggunakan metode **Serial Rolling Upgrade**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   ALUR SERIAL ROLLING UPGRADE PER NODE                      │
│                                                                             │
│  [1. Pre-Flight Check] ──> Pastikan Backup etcd Baru Tersimpan di MinIO!    │
│                            Pastikan N+1 Capacity Headroom Mencukupi!        │
│                                  │                                          │
│                                  ▼                                          │
│  [2. Cordon Node]      ──> Tandai node agar Scheduler tidak mengirim pod    │
│                            baru: `kubectl cordon worker-01`                 │
│                                  │                                          │
│                                  ▼                                          │
│  [3. Drain Node]       ──> Usir pod aplikasi secara tertib ke node lain:    │
│                            `kubectl drain worker-01 --ignore-daemonsets`    │
│                            (Tunduk pada PodDisruptionBudget / PDB!)         │
│                                  │                                          │
│                                  ▼                                          │
│  [4. Upgrade via       ──> Jalankan Playbook Ansible untuk patch kernel     │
│      Ansible]              dan binary k3s pada node tersebut                │
│                                  │                                          │
│                                  ▼                                          │
│  [5. Uncordon Node]    ──> Izinkan node menerima pod kembali:               │
│                            `kubectl uncordon worker-01`                     │
│                                  │                                          │
│                                  ▼                                          │
│  [6. Verifikasi Sehat] ──> Cek status node `Ready` & Workload seimbang.     │
│                            Lanjutkan ke Node worker-02 (Ulangi Serial).     │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Perlindungan Ketersediaan Pod: `PodDisruptionBudget (PDB)`**
> Sebelum menjalankan perintah `drain`, pastikan setiap layanan kritis memiliki resource `PodDisruptionBudget`. Contoh: jika `payment-api` memiliki 3 replika dan PDB dikonfigurasi dengan `minAvailable: 2`, maka proses `drain` akan **menolak mengusir pod** jika saat itu hanya ada 2 pod yang sehat di node lain! Ini mencegah perintah drain membuat aplikasi mengalami downtime total.

---

## 4. Keamanan Pasokan Container (Supply Chain Security) & Pemindaian Trivy

Memasang container image ke klaster produksi tanpa pemindaian kerentanan (*vulnerability scanning*) sama seperti memasukkan paket barang mencurigakan ke dalam brankas bank.

### 4.1 Gerbang Keamanan Trivy di Pipeline GitLab CI/CD
SRE mengintegrasikan **Trivy** pada pipeline CI/CD aplikasi untuk memindai container image sebelum image tersebut diizinkan masuk ke registry produksi:

```yaml
# Cuplikan .gitlab-ci.yml untuk Security Scanning
scan-container-security:
  stage: package-and-scan
  image: aquasec/trivy:latest
  script:
    # 1. Buat laporan lengkap format JSON untuk audit permanen
    - trivy image --format json --output gl-container-scanning-report.json "$IMAGE_NAME:$CI_COMMIT_SHORT_SHA"
    # 2. GAGALKAN PIPELINE (Exit Code 1) jika ditemukan CVE Critical atau High!
    - trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed "$IMAGE_NAME:$CI_COMMIT_SHORT_SHA"
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
    expire_in: 30 days
```

### 4.2 Kebijakan Penanganan Kerentanan (Vulnerability Governance)
1. **Critical/High dengan Patch Tersedia (*Fix Available*):** Pipeline CI otomatis gagal (*Fail-Closed*). Tim pengembang wajib meng-upgrade versi base image (misal beralih dari `alpine:3.18` ke `alpine:3.20`) sebelum kode bisa di-merge.
2. **Pengecualian Berbatas Waktu (*Vulnerability Waiver*):** Jika ditemukan CVE yang belum memiliki patch resmi dari pembuat library pihak ketiga, tim dapat mendaftarkan ID CVE tersebut ke file `.trivyignore`.
   - **Syarat Wajib Waiver:** Harus menyertakan nomor tiket audit keamanan, alasan mitigasi kompensasi (*compensating control* seperti pembatasan NetworkPolicy), dan **tanggal kedaluwarsa maksimal 30 hari**.

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Perintah `kubectl drain` Macet dan Menolak Berjalan (Timeout)** | 1. Pod dilindungi oleh `PodDisruptionBudget` yang tidak terpenuhi.<br>2. Ada pod lokal yang menggunakan `emptyDir` tanpa flag izin.<br>3. Ada pod DaemonSet yang belum diabaikan. | Jalankan drain dengan flag yang benar:<br>`kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force`. Periksa PDB yang menghalangi dengan `kubectl get pdb -A`. |
| **Restore Velero Berstatus `Completed`, tapi Pod Database `CrashLoopBackOff`** | File data database di-restore ke StorageClass yang salah atau izin kepemilikan direktori fisik (*UID/GID Linux*) berubah saat restore. | Periksa izin mount direktori data pada pod: `kubectl logs <pod-db>`. Pastikan UID `999` (postgres) memiliki hak akses penuh ke folder data PV. |
| **Upgrade k3s Menyebabkan Split-Brain pada etcd Quorum** | SRE melakukan upgrade pada 2 server control-plane sekaligus secara paralel, menyebabkan quorum terpecah. | **ATURAN MUTLAK:** Upgrade node control-plane wajib dilakukan **SATU PER SATU SECARA SERIAL**. Pastikan `k3s etcd-snapshot` sukses dibuat sebelum node pertama di-upgrade. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. PENCADANGAN & PEMULIHAN ETCD DI K3S BARE-METAL
# ==============================================================================

# Membuat snapshot etcd secara manual dan menyimpannya di disk lokal server master
sudo k3s etcd-snapshot save --name pre-upgrade-snapshot-$(date +%Y%m%d%H%M)

# Menampilkan daftar snapshot etcd yang tersedia beserta status kesehatannya
sudo k3s etcd-snapshot list

# ==============================================================================
# 2. OPERASI PENCADANGAN & PEMULIHAN VELERO
# ==============================================================================

# Membuat backup namespace production secara langsung (tanpa secret rahasia)
velero backup create prod-backup-$(date +%Y%m%d) --include-namespaces production --wait

# Melihat status detail hasil backup terakhir
velero backup describe prod-backup-$(date +%Y%m%d) --details

# Memulihkan seluruh namespace production dari backup yang tersimpan di MinIO S3
velero restore create --from-backup prod-backup-$(date +%Y%m%d) --wait

# ==============================================================================
# 3. MANAJEMEN MAINTENANCE DRAIN/CORDON PADA WORKER NODE
# ==============================================================================

# Mengunci node agar tidak menerima pod baru sebelum pemeliharaan OS
kubectl cordon worker-node-02

# Mengusir seluruh beban kerja aplikasi secara aman dari node tersebut
kubectl drain worker-node-02 --ignore-daemonsets --delete-emptydir-data --timeout=5m

# Membuka kembali node setelah proses upgrade kernel/k3s selesai
kubectl uncordon worker-node-02
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa status backup yang menunjukkan `Completed` pada sistem pencadangan Kubernetes (seperti Velero) **belum membuktikan** bahwa aplikasi dan database Anda siap dipulihkan saat terjadi bencana?
- A. Karena Velero tidak mendukung penyimpanan file di hard disk lokal.
- B. Karena status `Completed` hanya menunjukkan proses transfer file telah selesai, namun tidak menjamin bahwa transaksi database di dalam file tersebut berada dalam kondisi konsisten dan bebas korup (*consistent state*); validitas sesungguhnya hanya bisa dibuktikan melalui **Uji Coba Pemulihan Nyata (*Restore Drill*)**.
- C. Karena Velero selalu menghapus data setelah 5 menit.
- D. Karena seluruh file backup dienkripsi dengan kata sandi yang tidak diketahui siapa pun.

#### Soal 2
Dalam konteks pemulihan bencana (*Disaster Recovery*), apa perbedaan mendasar antara **RPO (Recovery Point Objective)** dan **RTO (Recovery Time Objective)**?
- A. RPO mengukur kecepatan koneksi internet, sedangkan RTO mengukur suhu prosesor server.
- B. RPO adalah jumlah maksimal kehilangan data transaksi bisnis yang ditoleransi diukur dalam satuan waktu ke belakang sejak backup terakhir, sedangkan RTO adalah durasi maksimal pemulihan sistem dari saat bencana terjadi hingga layanan kembali aktif melayani pengguna.
- C. RPO dan RTO adalah istilah yang sama persis dan bisa dipertukarkan.
- D. RPO khusus untuk sistem operasi Linux, sedangkan RTO khusus untuk sistem operasi Windows.

#### Soal 3
Sebelum mengeksekusi perintah `kubectl drain worker-01` untuk melakukan pembaruan kernel server bare-metal, konfigurasi Kubernetes apakah yang bertugas mencegah terjadinya pemadaman layanan dengan cara membatasi jumlah replika pod yang boleh mati dalam waktu bersamaan?
- A. HorizontalPodAutoscaler (HPA)
- B. **PodDisruptionBudget (PDB)**
- C. StorageClass Reclaim Policy
- D. NetworkPolicy Ingress Rule

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Perusahaan Anda memiliki klaster k3s bare-metal dengan 3 server control-plane yang menjalankan database terdistribusi internal `etcd`.
Tim keamanan jaringan mengumumkan bahwa ditemukan celah keamanan kritis (*Zero-Day Vulnerability*) pada paket kernel Ubuntu Linux yang terpasang di seluruh server, sehingga ketiga server master harus di-upgrade dan di-reboot sesegera mungkin.

Seorang engineer junior ingin menyelesaikan tugas dengan cepat. Ia membuat skrip shell sederhana:
```bash
for node in master-01 master-02 master-03; do
  ssh $node "sudo apt update && sudo apt upgrade -y linux-generic && sudo reboot" &
done
```
Skrip tersebut menjalankan upgrade dan reboot pada ketiga server master **secara bersamaan di latar belakang (*background parallel execution `&`*)**.

**Pertanyaan & Tugas Anda sebagai Lead SRE:**
1. Mengapa skrip paralel di atas akan memicu **Bencana Fatal Pemadaman Klaster (Loss of etcd Quorum)**?
2. Jelaskan konsep **Quorum pada etcd 3-Node**, dan berapa jumlah node minimal yang wajib hidup agar klaster Kubernetes tetap bisa menerima perintah operasional!
3. Tuliskan langkah-langkah prosedur upgrade rolling yang benar dan aman untuk ketiga server master tersebut!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Integritas data pada database transaksional memerlukan sinkronisasi memori ke disk (*flushing WAL*). Backup fisik tanpa koordinasi aplikasi sering menghasilkan snapshot setengah jadi yang gagal dibuka saat proses booting database baru.
2. **B.** RPO berbicara mengenai toleransi kehilangan data (*loss of data*), sedangkan RTO berbicara mengenai toleransi durasi pemadaman (*time to recover*).
3. **B.** **PodDisruptionBudget (PDB)** adalah kontrak resmi yang menjaga ketersediaan minimum aplikasi selama operasi pemeliharaan terencana (*voluntary disruptions*).

#### Solusi Soal Kasus Nyata:
1. **Penyebab Bencana Fatal (Loss of Quorum):**
   - Menjalankan reboot secara paralel pada ketiga node master akan mematikan seluruh proses `etcd` secara serentak.
   - Ketika server menyala kembali, konsensus *Raft* di dalam etcd bisa gagal menentukan pemimpin (*no elected leader*) jika terjadi *split-brain* atau penulisan data parsial, menyebabkan API Server Kubernetes lumpuh total (*Cluster-Wide Outage*).
2. **Konsep Quorum pada etcd 3-Node:**
   - Rumus Kuorum mayoritas: $\text{Quorum} = \text{floor}(N / 2) + 1$.
   - Untuk 3 node master, kuorum minimal adalah $\text{floor}(3 / 2) + 1 = \mathbf{2 \text{ node hidup}}$.
   - Jika 2 atau 3 node mati dalam waktu yang sama, etcd kehilangan kuorum dan langsung mengunci database menjadi *read-only* atau menolak seluruh request API Kubernetes.
3. **Prosedur Upgrade Rolling yang Benar & Aman:**
   - **Langkah 1 (Pencadangan Wajib):** Buat snapshot etcd manual pada master-01 dan pastikan tersimpan di penyimpanan eksternal MinIO S3:
     `sudo k3s etcd-snapshot save --name pre-kernel-upgrade`.
   - **Langkah 2 (Eksekusi Serial pada Master-01):**
     1. Kunci dan drain workload master-01: `kubectl cordon master-01 && kubectl drain master-01 --ignore-daemonsets`.
     2. Lakukan upgrade kernel dan reboot **hanya pada master-01**: `ssh master-01 "sudo apt upgrade -y && sudo reboot"`.
     3. Tunggu hingga master-01 menyala kembali, berstatus `Ready`, dan periksa kesehatan etcd: `sudo k3s etcd-snapshot list`.
     4. Buka kembali node: `kubectl uncordon master-01`.
   - **Langkah 3 (Verifikasi Kuorum Sebelum Melanjutkan):** Pastikan ketiga anggota etcd kembali sinkron dan sehat sebelum mengulangi langkah yang sama secara serial pada **master-02**, lalu terakhir pada **master-03**.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai strategi pencadangan, pemulihan bencana, dan rolling upgrade:
- Praktikkan di [LAB-02 — On-Prem Readiness, Patching, dan Trivy Security](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.2-production-onprem/lab/LAB-02-onprem-readiness-patching-trivy.md).
- Lanjutkan ke modul terakhir di Fase 8: [Modul 8.3 — Runbook & Dokumentasi Operasional](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.3-runbook-dokumentasi/README.md) untuk mempelajari penyusunan Runbook 4 skenario kritis dan penulisan dokumen arsitektur *as-built*.
