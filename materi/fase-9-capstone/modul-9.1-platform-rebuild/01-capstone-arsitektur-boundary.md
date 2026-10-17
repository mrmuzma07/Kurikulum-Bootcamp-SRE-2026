# 9.1.1 — Arsitektur Capstone dan Boundary

> **Ringkasan:** Memahami arsitektur menyeluruh proyek puncak (**Capstone Project**), memetakan pembagian batas kepemilikan teknologi (*Layer Boundary Ownership*), mengonfigurasi topologi target pada **OrbStack Apple Silicon ARM64**, serta membedakan secara tegas antara **Desired Topology, As-Built Topology, dan Evidence Topology** tanpa membocorkan kredensial rahasia.

---

## 1. Visi Capstone & Analogi Batas Kepemilikan

### 1.1 Analogi Dunia Nyata: "Ujian Sertifikasi Terbang Solo Pilot & Batas Kepemilikan Lahan Bangunan"
Bayangkan seorang kadet penerbang militer yang telah belajar teori aerodinamika, navigasi radar, dan mesin jet selama 4 bulan:
- **Fase Pembelajaran Biasa:** Instruktur duduk di samping kadet, memandu setiap tombol, dan mengambil alih kemudi jika terjadi kesalahan kecil.
- **Ujian Capstone (Penerbangan Solo Tanpa Pendamping):** Kadet dilepas sendirian di kokpit. Tugasnya adalah menyalakan mesin dari kondisi mati total (*Cold and Dark*), lepas landas, menavigasi rute berbadai, mendarat dengan selamat, dan menyerahkan rekaman kotak hitam (*Flight Data Recorder*) sebagai bukti sah kelulusan.

Di sisi lain, proyek infrastruktur skala besar membutuhkan **Batas Kepemilikan Lahan (*Boundary Ownership*)** yang sangat ketat:
- Petugas instalasi listrik tidak boleh membongkar pipa air bawah tanah.
- Tukang kayu tidak boleh mengubah fondasi beton cakar ayam.
- Jika setiap lapisan teknologi mencampuri lapisan lain (misal: pipeline CI aplikasi memegang master password database atau skrip shell mengubah konfigurasi kernel tanpa melalui Ansible), sistem akan menjadi *"Snowflake Infrastructure"* yang rapuh dan mustahil direkonstruksi saat terjadi bencana!

---

### 1.2 Topologi Target Produksi On-Premises di OrbStack (ARM64)

Seluruh ekosistem Capstone Bootcamp SRE 2026 dirancang untuk berjalan secara mandiri dan efisien di atas workstation **MacBook Apple Silicon (ARM64)** menggunakan **OrbStack**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 TOPOLOGI LINGKUNGAN EKSEKUSI ORBSTACK (ARM64)               │
│                                                                             │
│  [ MacBook Apple Silicon Host macOS ]                                       │
│  └── OrbStack Linux Engine (Native Virtualization & Container Daemon)       │
│      │                                                                      │
│      ├── 1. VM INFRASTRUCTURE (`infra-vm`)                                  │
│      │   ├─ MinIO S3 Server (Penyimpanan Remote State & Backup Velero)      │
│      │   ├─ GitLab Runner Lokal (Executor CI/CD Multi-Arch Buildx)          │
│      │   └─ Ansible Control Node (Pusat Eksekusi Playbook Otomasi)          │
│      │                                                                      │
│      ├── 2. VM KUBERNETES PRODUCTION (`servers-vm`)                         │
│      │   ├─ k3s Server (Control-Plane dengan etcd snapshot)                 │
│      │   ├─ k3s Worker-01 & Worker-02 (Node Beban Kerja Aplikasi)           │
│      │   ├─ MetalLB L2 Mode (LoadBalancer VIP Jaringan Lokal Fisik)         │
│      │   └─ ArgoCD GitOps Engine (Sinkronisasi Otomatis Pull-Based)         │
│      │                                                                      │
│      └── 3. K3D DISPOSABLE STAGING (`k3d-staging`)                          │
│          └─ Klaster k3s ringan di dalam container untuk uji coba rilis      │
│             kandidat (*Staging Gate*) sebelum promosi ke klaster produksi.  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Rantai Pengiriman End-to-End & Boundary Ownership Matrix

### 2.1 Alur Pengiriman Aplikasi End-to-End (The Golden Path)

Workload aplikasi produksi mengalir secara deterministik melalui pipa pengiriman berikut:

```text
┌─────────────────────────┐     ┌─────────────────────────┐
│ 1. Git Commit Developer │ ──> │ 2. GitLab CI Multi-Arch │
│    (Application Repo)   │     │    (Test, Scan, Buildx) │
└─────────────────────────┘     └────────────┬────────────┘
                                             │ Push Immutable Digest (`sha256:...`)
                                             ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│ 4. ArgoCD Pull Reconcile│ <── │ 3. Reviewed GitOps MR   │
│    (Sync & Health Check)│     │    (Manifests Repo)     │
└────────────┬────────────┘     └─────────────────────────┘
             │ Deploy via Helm
             ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│ 5. MetalLB L2 VIP       │ ──> │ 6. Observabilitas Penuh │
│    (Network Exposure)   │     │    (Mimir, Loki, Tempo) │
└─────────────────────────┘     └────────────┬────────────┘
                                             │ Evaluasi
                                             ▼
                                ┌─────────────────────────┐
                                │ 7. SRE SLO & Burn-Rate  │
                                │    (Decision & Alerting)│
                                ---------------------------
```

---

### 2.2 Matriks Batas Kepemilikan 8 Lapisan Teknologi (Boundary Ownership)

Setiap alat memiliki yurisdiksi tanggung jawabnya masing-masing. Pelanggaran terhadap batas ini (*boundary violation*) adalah anti-pattern yang dilarang keras dalam Capstone:

| Lapisan Teknologi | Tanggung Jawab Wajib Dimiliki (*Owns*) | Dilarang Keras Mengambil Alih (*Must NOT Own*) |
|---|---|---|
| **1. OpenTofu (IaC)** | Metadata VM, virtual disk, alokasi subnet jaringan, state locking di MinIO S3. | Tidak boleh menginstal paket aplikasi di dalam OS Linux atau mengelola deployment Pod K8s. |
| **2. Ansible** | OS hardening, sysctl tuning, firewall (ufw), patching kernel, instalasi binary k3s server & agent. | Tidak boleh menjadi alat rilis aplikasi bisnis atau menggantikan peran deklaratif GitOps ArgoCD. |
| **3. Kubernetes (k3s)** | Penjadwalan Pod (*scheduling*), Service routing, Ingress, Storage binding, etcd consensus. | Tidak boleh memodifikasi konfigurasi virtual machine atau melakukan commit ke repositori Git. |
| **4. Helm** | Templating manifes deklaratif, OCI package chart, pemisahan nilai non-secret per environment. | Tidak boleh mengambil keputusan promosi produksi atau memotong alur review manusia. |
| **5. GitLab CI / ArgoCD** | Validasi linting/security, build container multi-arch, pull-based reconciliation, drift detection. | CI dilarang memegang kredensial `kubeconfig` produksi; ArgoCD dilarang menyuntikkan raw secret plaintext. |
| **6. MetalLB & Ingress** | Alokasi IP LoadBalancer di jaringan lokal, pengumuman ARP L2, TLS termination di gerbang depan. | Tidak boleh menentukan apakah aplikasi sehat secara bisnis atau mengabaikan aturan routing CNI. |
| **7. Full Observability** | Pengumpulan metrik (Alloy/Mimir), log (Loki), trace (Tempo), visualisasi dashboard USE/RED. | Tidak boleh mengambil alih peran komando insiden manusia atau mematikan layanan secara sepihak. |
| **8. SRE Capstone Governance** | Penetapan kontrak SLO, rotasi On-Call, pelaksanaan Game Day, penulisan Blameless Postmortem. | Dilarang mengklaim sistem telah lulus tanpa bukti (*evidence*) rekaman runtime yang valid! |

---

## 3. Tiga Dimensi Topologi Arsitektur: Desired vs As-Built vs Evidence

Dalam dunia rekayasa SRE profesional, diagram arsitektur tidak boleh hanya ada dalam bentuk gambar statis di dokumen presentasi. SRE memelihara 3 bentuk representasi arsitektur:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 TIGA DIMENSI DOKUMENTASI ARSITEKTUR SRE                     │
├──────────────────────────┬──────────────────────────┬───────────────────────┤
│ Dimensi Topologi         │ Karakteristik & Sumber   │ Tujuan Operasional    │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│ **1. Desired Topology**  │ Kode deklaratif di Git   │ Sumber kebenaran satu-│
│    (Cita-Cita Ideal)     │ (OpenTofu `.tf`, Ansible │ satunya (*Single      │
│                          │ Playbooks, Helm values). │ Source of Truth*).    │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│ **2. As-Built Topology** │ Kondisi nyata di runtime │ Memvalidasi fakta     │
│    (Realitas Lapangan)   │ hasil perintah discovery │ fisik dan mendeteksi  │
│                          │ (`ip a`, `kubectl get`). │ adanya penyimpangan.  │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│ **3. Evidence Topology** │ Rekaman bukti berstempel │ Bukti kelulusan audit │
│    (Rantai Bukti Sah)    │ waktu UTC yang membukti- │ dan verifikasi pasca- │
│                          │ kan transisi Desired ➔   │ rekonstruksi bencana  │
│                          │ As-Built berjalan sukses.│ (*Disaster Recovery*).│
└──────────────────────────┴──────────────────────────┴───────────────────────┘
```

### 3.1 Deteksi dan Penanganan Architectural Drift
- **Apa itu Drift?** Ketika seseorang melakukan perubahan manual via SSH di server fisik (misal: menaikkan ukuran swap atau mengubah port firewall) tanpa mencatatnya di kode Ansible.
- **Konsekuensi Fatal:** Saat klaster direkonstruksi dari nol (*rebuild*), perubahan manual tersebut akan hilang dan sistem baru akan gagal berfungsi!
- **Aturan Capstone:** Setiap kali terdeteksi perbedaan antara *As-Built* dan *Desired State*, engineer wajib memilih: kembalikan server ke spesifikasi Git (*self-heal*), atau jika perubahan darurat tersebut benar, segera buat Merge Request di Git agar *Desired State* diperbarui secara resmi.

---

## 4. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Pelanggaran Batas (*Boundary Violation*) | Solusi & Tindakan Perbaikan SRE |
|---|---|---|
| **Pipeline CI Gagal karena Kubeconfig Produksi Kedaluwarsa** | **Pelanggaran Batas Lapisan 5:** Pipeline CI mencoba menjalankan `kubectl apply` langsung ke cluster produksi (*Push-based CI anti-pattern*). | Cabut seluruh akses kubeconfig dari GitLab CI! Konfigurasikan alur GitOps: CI hanya membuat MR ke repositori manifes, dan biarkan ArgoCD yang menarik (*pull*) perubahan dari dalam klaster. |
| **Klaster k3s Gagal Rebuild karena Script Shell Tergantung pada File Lokal Mac** | **Pelanggaran Batas Lapisan 1 & 2:** Menggunakan script ad-hoc yang menyalin file dari desktop laptop tanpa deklarasi di OpenTofu/Ansible. | Masukkan seluruh file konfigurasi ke dalam template Jinja2 Ansible atau ConfigMap GitOps agar klaster bisa dibangun ulang secara deterministik oleh siapa pun. |
| **IP MetalLB Bentrok dengan IP Router Kantor** | **Pelanggaran Batas Lapisan 6:** Alokasi pool IP MetalLB tidak dikunci di level jaringan underlay dan bertabrakan dengan pool DHCP router. | Koordinasikan alokasi subnet khusus dengan tim jaringan; kunci pool MetalLB pada rentang IP statis terisolasi yang tidak tumpang tindih (*dedicated non-overlapping CIDR*). |

---

## 5. Cheatsheet & Ringkasan Perintah Verifikasi Arsitektur

```bash
# ==============================================================================
# 1. VERIFIKASI SUMBER DAYA ORBSTACK DI WORKSTATION MAC (HOST CLI)
# ==============================================================================

# Menampilkan daftar seluruh VM Linux yang sedang berjalan di OrbStack
orb list

# Memeriksa batas alokasi memori CPU/RAM yang digunakan oleh mesin OrbStack
orb info

# ==============================================================================
# 2. VERIFIKASI AS-BUILT STATUS DARI LAPISAN KUBERNETES & METALLB
# ==============================================================================

# Memeriksa apakah seluruh node k3s terdeteksi dengan arsitektur ARM64
kubectl get nodes -o wide -L kubernetes.io/arch

# Memeriksa apakah alokasi IP VIP MetalLB aktif pada Service produksi
kubectl get svc -A -l app.kubernetes.io/managed-by=Helm

# ==============================================================================
# 3. VERIFIKASI STATUS DESIRED VS LIVE GITOPS ARGOCD
# ==============================================================================

# Memeriksa apakah seluruh aplikasi berada dalam kondisi Synced dan Healthy (No Drift!)
argocd app list
```

---

## 6. Evaluasi Pemahaman Mandiri

### 6.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa dalam arsitektur modern SRE, pipeline **GitLab CI/CD pada repositori kode aplikasi** dilarang keras menyimpan kredensial `kubeconfig` cluster produksi untuk menjalankan perintah `kubectl apply` secara langsung?
- A. Karena `kubectl apply` tidak kompatibel dengan prosesor Apple Silicon ARM64.
- B. Untuk menjaga batas keamanan (*security boundary*) dan menerapkan prinsip *Least Privilege*; jika server CI disusupi oleh penyerang melalui dependency injection, penyerang tidak akan mendapatkan kendali administratif ke klaster produksi karena akses ditarik dari dalam oleh **ArgoCD (Pull-based GitOps)**.
- C. Karena Kubernetes hanya mengizinkan 1 koneksi admin per hari.
- D. Agar kuota bandwidth jaringan internet kantor tidak cepat habis.

#### Soal 2
Dalam matriks batas kepemilikan (*Boundary Ownership Matrix*), lapisan teknologi manakah yang memiliki kewenangan penuh untuk mengelola siklus hidup mesin virtual, disk virtual, dan alokasi CIDR jaringan dasar sebelum sistem operasi diinstal?
- A. Helm Chart
- B. **OpenTofu (Infrastructure as Code)**
- C. ArgoCD
- D. Grafana Tempo

#### Soal 3
Apa perbedaan mendasar antara dokumen **Desired Topology** dengan **As-Built Topology** dalam pengelolaan infrastruktur bare-metal on-premises?
- A. Desired Topology berisi konfigurasi ideal yang dideklarasikan di repositori Git, sedangkan As-Built Topology adalah kondisi fisik dan logis nyata yang benar-benar sedang berjalan di data center saat ini.
- B. Desired Topology ditulis dalam bahasa Python, sedangkan As-Built Topology ditulis dalam bahasa C++.
- C. As-Built Topology dibuat sebelum server dibeli, sedangkan Desired Topology dibuat setelah server rusak.
- D. Tidak ada perbedaan; keduanya adalah dokumen yang sama persis.

---

### 6.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Seorang kandidat insinyur SRE sedang mempersiapkan ujian Capstone di MacBook miliknya. Ia telah berhasil menginstal klaster k3s di OrbStack. Namun, agar pekerjaannya cepat selesai, kandidat tersebut melakukan hal-hal berikut:
1. Menjalankan perintah `ssh root@servers-vm` lalu mengetik perintah `apt install` dan mengedit file konfigurasi jaringan `/etc/network/interfaces` secara manual tanpa membuat Ansible Playbook.
2. Menyimpan token admin k3s secara terang-terangan di dalam file `README.md` repositori Git publik.
3. Menjalankan deployment aplikasi pembayaran dengan mengetik perintah manual `kubectl run payment --image=my-app:latest` langsung di terminal.

Saat sesi **Readiness Review**, dewan penguji menolak meluluskan kandidat tersebut (*Result: NOT READY*).

**Pertanyaan & Tugas Anda sebagai Mentor SRE:**
1. Sebutkan **3 pelanggaran batas (*boundary violations*) dan aturan tata kelola SRE** fatal yang dilakukan oleh kandidat tersebut!
2. Jelaskan apa yang akan terjadi jika laptop kandidat tersebut mengalami kerusakan hard disk dan ia diminta membangun kembali sistemnya dari nol (*Disaster Recovery*) berdasarkan repositori Git miliknya!
3. Tuliskan perbaikan langkah demi langkah yang harus dilakukan kandidat tersebut agar memenuhi kriteria kelulusan Capstone!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Pola Push-based CI membuka celah keamanan fatal di mana runner CI memegang kunci master klaster. Pola Pull-based GitOps membatasi hak runner hanya untuk mem-push artefak image ke registry dan membuat MR konfigurasi.
2. **B.** **OpenTofu** adalah pemilik lapisan fondasi infrastruktur (VM, network boundary, storage LUN).
3. **A.** Pemisahan Desired vs As-Built adalah esensi dari *Drift Management*. Tanpa membandingkan keduanya, SRE tidak tahu apakah sistem di lapangan masih sesuai dengan cetak biru yang disetujui.

#### Solusi Soal Kasus Nyata:
1. **Tiga Pelanggaran Fatal Kandidat:**
   - **Pelanggaran Prinsip No Snowflake (Manual SSH Edit):** Mengedit file konfigurasi OS secara langsung via SSH tanpa playbook Ansible membuat server menjadi unik dan mustahil direproduksi (*unrepeatable snowflake*).
   - **Pelanggaran Higienitas Rahasia (Secrets Hygiene Breach):** Menuliskan k3s token secara plaintext di Git publik melanggar standar kepatuhan keamanan dan membuka akses klaster ke publik.
   - **Bypass GitOps & Immutable Digest:** Menggunakan `kubectl run` dengan tag mutable `:latest` memotong alur kendali ArgoCD dan melanggar prinsip *Immutable Artifact Provenance*.
2. **Dampak Saat Bencana Rebuild Terjadi:**
   - Seluruh konfigurasi OS manual akan hilang. Script OpenTofu yang baru hanya akan menghasilkan VM kosong tanpa setting jaringan `/etc/network/interfaces` yang benar.
   - Perintah `kubectl run` tidak tercatat di repositori GitOps, sehingga klaster baru akan kosong tanpa ada aplikasi yang berjalan (*Total Data and Workload Loss*).
3. **Langkah Perbaikan Standar Capstone:**
   - **Langkah 1:** Pindahkan seluruh perintah konfigurasi OS dan jaringan ke dalam **Ansible Roles & Playbooks** yang idempoten di repositori `infra-ansible`.
   - **Langkah 2:** Hapus token k3s dari Git; gunakan mekanisme penyuntikan rahasia terenkripsi via `ansible-vault` atau variabel environment aman.
   - **Langkah 3:** Bungkus aplikasi ke dalam **Helm Chart** di repositori GitOps, gunakan image digest tetap (`sha256:...`), dan biarkan **ArgoCD** yang melakukan sinkronisasi otomatis ke klaster.

---

## 7. Langkah Selanjutnya & Kaitan Modul

Setelah memahami arsitektur dan batas kepemilikan proyek Capstone:
- Lanjutkan ke [9.1.2 — Contract OpenTofu → Ansible → k3s Rebuild](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-9-capstone/modul-9.1-platform-rebuild/02-opentofu-ansible-k3s-rebuild.md) untuk mempelajari kontrak data serah terima (*handoff contract*) antar alat dan protokol rekonstruksi platform dari kondisi hancur total (*bare-metal destroy-and-rebuild*).
- Praktikkan di [LAB-01 — Infrastructure Rebuild Lab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-9-capstone/modul-9.1-platform-rebuild/lab/LAB-01-infrastructure-rebuild.md).
