# 01 — GitOps Source of Truth dan Struktur Repository: Menetapkan Git Sebagai Satu-Satunya Sumber Kebenaran

> **Ringkasan:** Di era komputasi awan modern, cara kita mengelola klaster Kubernetes telah berevolusi dari pola dorong tradisional (*Push-based CI deployment*) menuju paradigma **GitOps**. Dalam GitOps, repositori Git diperlakukan sebagai **Satu-Satunya Sumber Kebenaran (*Single Source of Truth*)** untuk seluruh keadaan klaster yang diinginkan (*desired state*). Tidak ada lagi engineer yang diperbolehkan mengetik `kubectl apply` atau mengedit konfigurasi secara langsung di server produksi. Dokumen ini membedah empat pilar fundamental OpenGitOps, membongkar arsitektur pemisahan repositori (*App Repo vs GitOps Repo*), merancang struktur direktori repositori GitOps skala enterprise, serta menyusun alur promosi perubahan yang transparan dan dapat diaudit.

---

## 1. Analogi Pemula: Buku Tabungan Bank Resmi vs Uang Fisik di Dompet

Untuk memahami mengapa GitOps mengubah cara kerja SRE secara fundamental, mari gunakan analogi sistem perbankan:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI BUKU TABUNGAN RESMI BANK vs KLASTER KUBERNETES      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ CATATAN BUKU TABUNGAN RESMI BANK (GITOPS REPOSITORY) ]                   │
│  - Merupakan dokumen hukum satu-satunya sumber kebenaran (*Source of Truth*)│
│  - Jika saldo tertulis Rp 10.000.000, maka itulah kekayaan sah Anda.        │
│  - Aturan Mutlak: Saldo di buku tabungan HANYA BISA BERUBAH jika ada        │
│    transaksi resmi yang ditandatangani teller (*Git Commit / Pull Request*)!│
│                                                                             │
│  [ UANG TUNAI DI DALAM DOMPET ANDA (LIVE KUBERNETES CLUSTER) ]              │
│  - Dompet hanyalah cerminan fisik sementara dari saldo buku tabungan.       │
│                                                                             │
│  [ SITUASI DRIFT & TINDAKAN PENGAWAS (ARGOCD CONTROLLER) ]                  │
│  - Suatu malam, seorang pencuri menyelipkan uang kertas monopoli palsu      │
│    ke dalam dompet Anda (**Engineer nakal melakukan `kubectl edit` manual**)!│
│  - Petugas Keamanan Bank (ArgoCD) memantau dompet Anda setiap 3 menit:     │
│    "Loh! Di buku tabungan tertulis 3 lembar uang, kenapa di dompet ada 5?"  │
│    (**DRIFT DETECTED: STATUS OutOfSync!**)                                  │
│  - Aksi Pengamanan Otomatis (**Automated Self-Healing**):                   │
│    Petugas bank seketika merampas 2 uang palsu tersebut dan membakarnya!    │
│    Isi dompet dikembalikan persis 100% sesuai catatan sah buku tabungan!    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Empat Pilar Prinsip OpenGitOps

Berdasarkan standar resmi CNCF OpenGitOps Working Group, sebuah sistem baru dapat disebut menerapkan GitOps sejati jika memenuhi 4 prinsip dasar:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       EMPAT PILAR UTAMA OPENGITOPS                          │
├─────┬──────────────────────┬────────────────────────────────────────────────┤
│ No  │ Prinsip              │ Definisi Teknis Operasional SRE                │
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 1   │ **Declarative**      │ Keadaan seluruh sistem (*desired state*)       │
│     │                      │ dideklarasikan dalam format deklaratif (YAML / │
│     │                      │ Helm / Kustomize), bukan skrip prosedural.     │
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 2   │ **Versioned and**    │ Deklarasi disimpan di Git dengan riwayat versi │
│     │ **Immutable**        │ abadi, commit SHA tak terhapus, dan pelacakan  │
│     │                      │ riwayat perubahan (*Audit Trail*).             │
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 3   │ **Pulled**           │ Agen perangkat lunak (ArgoCD) di dalam klaster │
│     │ **Automatically**    │ menarik (*pull*) deklarasi dari Git secara     │
│     │                      │ proaktif. Klaster tidak menunggu push dari luar│
├─────┼──────────────────────┼────────────────────────────────────────────────┤
│ 4   │ **Continuously**     │ Agen terus-menerus membandingkan keadaan nyata │
│     │ **Reconciled**       │ (*live state*) dengan Git (*desired state*),   │
│     │                      │ serta memperbaiki deviasi secara otomatis.     │
└─────┴──────────────────────┴────────────────────────────────────────────────┘
```

---

## 3. Pemisahan Repositori: App Repo vs GitOps Repo vs IaC Repo

Banyak pemula tergoda meletakkan source code aplikasi (Go/Java) dan manifest deployment Kubernetes di dalam satu repositori Git yang sama (*Monorepo Campur Aduk*). Di lingkungan produksi, pola ini sangat berbahaya:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 MENGAPA POLA SATU REPOSITORI CAMPUR ADUK BERBAHAYA?         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Pipa CI Terjebak Putaran Tak Berujung (*Infinite Feedback Loop*):        │
│     Developer push kode -> CI jalan -> CI update tag di deployment.yaml     │
│     -> CI commit ke repo yang sama -> Commit baru memicu CI jalan lagi!     │
│                                                                             │
│  2. Pelanggaran Batas Keamanan Hak Akses (*Privilege Separation*):          │
│     Agar CI bisa deploy, Runner CI developer harus diberi file `kubeconfig` │
│     klaster produksi. Jika runner diretas, seluruh klaster jebol!           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Arsitektur Pemisahan Tiga Repositori SRE (Separation of Concerns)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PEMISAHAN TIGA REPOSITORI TINGKAT PRODUKSI                  │
├──────────────────────┬──────────────────────────────┬───────────────────────┤
│ Repositori           │ Konten / Berkas Utama        │ Hak Akses & Pemilik   │
├──────────────────────┼──────────────────────────────┼───────────────────────┤
│ **1. App Repo**      │ - Source code (Go/Python)    │ Tim Pengembang        │
│    (`payment-app`)   │ - Dockerfile & Unit Tests    │ (Developers)          │
│                      │ - Pipeline CI (`.gitlab-ci`) │ *Akses: Write Kode*   │
├──────────────────────┼──────────────────────────────┼───────────────────────┤
│ **2. GitOps Repo**   │ - Helm Values (dev/stg/prod) │ Tim SRE & Platform    │
│    (`gitops-infra`)  │ - ArgoCD Application CRDs    │ (Engineers & Leads)   │
│                      │ - Kustomize Overlays         │ *Akses: 4-Eyes Review*│
├──────────────────────┼──────────────────────────────┼───────────────────────┤
│ **3. IaC Repo**      │ - OpenTofu (.tf) Hypervisor  │ Tim Core SRE          │
│    (`iac-onprem`)    │ - Ansible Playbooks (OS)     │ (Infrastructure Team) │
│                      │ - MetalLB & k3s bootstrap    │ *Akses: Restricted*   │
└──────────────────────┴──────────────────────────────┴───────────────────────┘
```

---

## 4. Standar Layout Direktori Repositori GitOps Enterprise

Berikut adalah struktur direktori repositori GitOps (`gitops-infra`) yang memisahkan aplikasi berdasarkan domain dan lingkungan menggunakan **Kustomize Overlays** atau **Helm Values**:

```text
gitops-infra/
├── README.md                            # Dokumentasi standar repositori GitOps
├── bootstrap/                           # Titik awal inisialisasi ArgoCD
│   ├── root-application.yaml            # Pola "App of Apps" induk
│   └── argocd-projects.yaml             # Pembagian project & batas hak akses namespace
│
├── infrastructure/                      # Komponen Platform Bersama (Cluster-Wide)
│   ├── metallb/
│   │   ├── ipaddresspool.yaml
│   │   └── l2advertisement.yaml
│   ├── ingress-nginx/
│   │   └── values-prod.yaml
│   └── cert-manager/
│       └── cluster-issuer.yaml
│
└── apps/                                # Beban Kerja Aplikasi Bisnis (Microservices)
    └── payment-service/
        ├── base/                        # Konfigurasi dasar bersama
        │   ├── kustomization.yaml
        │   └── deployment.yaml
        └── envs/                        # Pelapisan nilai per lingkungan
            ├── staging/
            │   ├── kustomization.yaml   # Mengacu ke base + patch staging
            │   └── values.yaml
            └── production/
                ├── kustomization.yaml   # Mengacu ke base + patch prod
                └── values.yaml
```

### Contoh Berkas Kustomization Produksi (`apps/payment-service/envs/production/kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Rujukan ke cetak biru universal di folder base
resources:
  - ../../base

# Namespace terisolasi target produksi
namespace: production

# Menimpa gambar kontainer secara deterministik berbasis Image Digest
images:
  - name: registry.internal.corp/bootcamp/payment-service
    digest: sha256:8f9b2c1743a19e24f706e25776d6ec1438914619736c4b2b1a8d0e5124fa68b1

# Penambahan label audit otomatis
commonLabels:
  env: production
  managed-by: argocd-gitops
```

---

## 5. Alur Promosi Perubahan Antar-Lingkungan (The Promotion Pipeline)

Bagaimana sebuah commit kode di repositori aplikasi berpindah secara aman hingga dideploy oleh ArgoCD ke klaster produksi?

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR KERJA PROMOSI PERUBAHAN END-TO-END                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ LANGKAH 1: APP REPO ]                                                    │
│  Developer push commit ke `payment-app`.                                    │
│  GitLab CI memvalidasi kode, menjalankan unit test, dan membuat Image:     │
│  `payment-service@sha256:8f9b...`                                           │
│            │                                                                │
│            ▼                                                                │
│  [ LANGKAH 2: AUTOMATED PULL REQUEST KE GITOPS REPO ]                       │
│  Di akhir pipeline CI, runner otomatis membuka Merge Request ke repositori  │
│  `gitops-infra` pada berkas: `apps/payment-service/envs/staging/`           │
│            │                                                                │
│            ▼                                                                │
│  [ LANGKAH 3: MERGE KE STAGING & AUTO-SYNC ARGOCD ]                         │
│  Merge Request di-approve -> Masuk branch `main` GitOps repo.               │
│  ArgoCD mendeteksi commit baru -> Mengupdate klaster Staging seketika!       │
│  Tim QA menjalankan automated integration test selama 1 jam.                │
│            │                                                                │
│            ▼ LULUS 100%                                                     │
│  [ LANGKAH 4: PROMOTION MERGE REQUEST KE PRODUCTION ]                       │
│  Developer/SRE membuka MR promosi menyalin digest ke:                       │
│  `apps/payment-service/envs/production/`                                    │
│            │                                                                │
│            ▼                                                                │
│  [ LANGKAH 5: 4-EYES PRINCIPLE PEER REVIEW ]                                │
│  Lead SRE meninjau perbedaan diff di GitLab UI.                             │
│  Lead SRE menyetujui dan mengklik **Merge**.                                │
│            │                                                                │
│            ▼                                                                │
│  [ LANGKAH 6: REKONSILIASI AKHIR ARGOCD ]                                   │
│  ArgoCD di klaster produksi mendeteksi perubahan Git -> Melakukan rekonsiliasi│
│  *Rolling Update* Pod tanpa downtime. Sistem produksi resmi terbarukan!     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA MASALAH STRUKTUR GITOPS                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "ComparisonError: Failed to load target specs: git checkout" ] │
│  Gejala: ArgoCD menampilkan status `ComparisonError` pada dashboard.       │
│  Akar Masalah:                                                              │
│  - Parameter `targetRevision` pada CRD Application mengarah ke branch atau  │
│    tag yang tidak ada di repositori GitOps.                                 │
│  - Atau SSH Deploy Key yang didaftarkan di ArgoCD tidak memiliki izin baca. │
│  Solusi SRE:                                                                │
│  1. Periksa ketersediaan branch: `git branch -r` pada repositori GitOps.    │
│  2. Uji kredensial repo di antarmuka ArgoCD: *Settings -> Repositories*.    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: MULTI-CONTROLLER CONFLICT (ARGOCD vs HPA FIGHTING) ]            │
│  Gejala: Deployment terus berganti status antara Synced dan OutOfSync       │
│          setiap beberapa detik (*flapping*).                                │
│  Akar Masalah:                                                              │
│  - Git mendefinisikan `replicas: 3`, tetapi Horizontal Pod Autoscaler (HPA) │
│    di klaster menaikkan replika menjadi `replicas: 8` karena beban trafik.  │
│    ArgoCD mendeteksi ini sebagai drift dan berusaha mengembalikan ke 3!     │
│  Solusi SRE:                                                                │
│  Instruksikan ArgoCD untuk mengabaikan perbedaan kolom `replicas`:          │
│  spec:                                                                      │
│    ignoreDifferences:                                                       │
│      - group: apps                                                          │
│        kind: Deployment                                                     │
│        jsonPointers:                                                        │
│          - /spec/replicas                                                   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: TARGET REVISION BERGERAK (FLOATING BRANCH DANGER) ]             │
│  Gejala: Rilis produksi tiba-tiba berubah tanpa ada pengumuman rilis baru.  │
│  Akar Masalah:                                                              │
│  - Konfigurasi ArgoCD mengarah ke branch dinamis `targetRevision: dev`.     │
│    Setiap kali developer push ke dev, klaster produksi ikut terupdate liar! │
│  Solusi SRE:                                                                │
│  Di lingkungan produksi, WAJIB mengunci target ke branch utama yang         │
│  terproteksi (`targetRevision: main`) atau mengunci ke Git Tag rilis!       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menguji rendering Kustomize secara lokal sebelum melakukan git commit
kubectl kustomize apps/payment-service/envs/production/

# 2. Menampilkan status sinkronisasi seluruh aplikasi di ArgoCD via CLI
argocd app list

# 3. Menampilkan perbedaan (diff) antara Git dan Klaster pada aplikasi tertentu
argocd app diff payment-service-prod

# 4. Memaksa ArgoCD menyegarkan cache Git seketika (Bypass interval polling 3 menit)
argocd app get payment-service-prod --refresh

# 5. Memeriksa status kesehatan controller ArgoCD di namespace argocd
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa dalam arsitektur GitOps skala enterprise, source code aplikasi dan manifest deployment Kubernetes **WAJIB** dipisahkan ke dalam dua repositori Git yang berbeda (*Multi-Repo Pattern*)?
- A. Karena Git tidak mampu menyimpan lebih dari 100 baris kode dalam satu repositori.
- B. Memisahkan siklus hidup rilis, memutus potensi *infinite CI build loop*, dan menerapkan pemisahan hak akses keamanan (*Principle of Least Privilege*) di mana runner CI pengembang tidak perlu memegang kredensial akses ke klaster produksi.
- C. Agar developer tidak perlu belajar Git.
- D. Karena ArgoCD menolak membaca repositori yang memuat berkas Dockerfile.

*Kunci Jawaban:* **B**. Pemisahan repositori menjamin isolasi keamanan kredensial dan mencegah putaran picu tak berujung pada pipeline CI/CD.

### Soal 2 (Pilihan Ganda)
Dalam prinsip OpenGitOps, apa yang dimaksud dengan sifat **Continuously Reconciled**?
- A. Operator wajib mengetik perintah sinkronisasi manual setiap pagi.
- B. Agen perangkat lunak (ArgoCD) secara berkala dan otomatis membandingkan keadaan nyata di dalam klaster dengan keadaan deklaratif yang tercatat di Git, serta secara otomatis mengoreksi deviasi jika terjadi perubahan liar (*Drift Remediation*).
- C. Klaster otomatis menghapus seluruh namespace setiap 24 jam.
- D. Mengubah format manifest YAML menjadi JSON.

*Kunci Jawaban:* **B**. Rekonsiliasi berkelanjutan menjamin bahwa keadaan klaster aktual selalu konvergen menuju deklarasi sah di Git.

### Soal 3 (Pilihan Ganda)
Jika sebuah klaster Kubernetes menggunakan Horizontal Pod Autoscaler (HPA) untuk menaikkan dan menurunkan jumlah replika Pod aplikasi secara otomatis, konfigurasi apa yang wajib ditambahkan pada CRD ArgoCD Application agar tidak terjadi bentrokan status (*OutOfSync flapping*)?
- A. Menghapus HPA dari klaster.
- B. Menambahkan blok `ignoreDifferences` yang menargetkan JSON pointer `/spec/replicas` pada objek Deployment, sehingga ArgoCD menyerahkan wewenang penyesuaian jumlah replika kepada kontroler HPA.
- C. Mematikan fitur self-healing pada klaster.
- D. Mengubah namespace menjadi default.

*Kunci Jawaban:* **B**. `ignoreDifferences` memberi tahu ArgoCD untuk tidak memperlakukan mutasi dinamis pada kolom tertentu sebagai *drift* konfigurasi.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Pada saat terjadi insiden darurat di produksi pada pukul 02.00 pagi, seorang engineer yang sedang on-call panik dan langsung melakukan SSH ke master k3s, lalu menjalankan `kubectl edit deployment payment-api` untuk mengubah variabel memory limit dari 512Mi ke 2Gi. Dua menit kemudian, nilai memory limit tersebut tiba-tiba kembali ke 512Mi secara misterius, dan aplikasi kembali tumbang. Jelaskan mengapa hal ini terjadi berdasarkan mekanisme kerja ArgoCD, dan bagaimana prosedur darurat yang benar dalam paradigma GitOps!

*Panduan Solusi:*
1. **Penyebab Kembalinya Konfigurasi (Automated Self-Healing):**
   Tindakan `kubectl edit` manual di klaster menciptakan kondisi *Drift* (keadaan nyata di klaster menyimpang dari apa yang tertulis di Git). Karena aplikasi dikelola oleh ArgoCD dengan fitur **Automated Self-Healing** aktif, kontroler ArgoCD yang melakukan sinkronisasi berkala mendeteksi perbedaan tersebut. ArgoCD menganggap mutasi manual tersebut sebagai anomali/serangan, dan seketika menimpa (*overwrite*) memori kembali ke 512Mi sesuai deklarasi sah di repositori Git!
2. **Prosedur Darurat yang Benar (The GitOps Hotfix Way):**
   - **Terapkan Perubahan di Git, BUKAN di Klaster:**
     Buka berkas `values-prod.yaml` pada repositori GitOps, naikkan nilai memory limit menjadi `2Gi`.
   - **Bypass Cepat melalui Emergency Commit / Fast-Track MR:**
     Lakukan commit langsung ke branch utama atau approve MR darurat dengan keterangan insiden tiket P1.
   - **Paksa Sinkronisasi Instan:**
     Jalankan `argocd app sync payment-api --force` atau klik **Sync** di UI ArgoCD.
   - Dengan cara ini, perbaikan memori berhasil diterapkan di klaster, dan perubahan tersebut tercatat resmi di riwayat Git audit untuk investigasi *Postmortem* esok pagi!

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Modul 6.1 — GitLab CI/CD](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.1-gitlab-ci-cd/) — Memahami pipeline penghasil artefak dan otomasi pengujian.
- **Topik Berikutnya:**
  - [02 — Instalasi ArgoCD, Application CRD, dan Siklus Sinkronisasi](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/02-argocd-install-application-sync.md) — Memasang kontroler ArgoCD di k3s dan menghubungkan aplikasi.
  - [03 — ApplicationSet, Secrets Boundary, Drift, dan Self-Healing](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/03-applicationset-secrets-drift-selfheal.md) — Mengelola multi-klaster dan pemulihan drift otomatis.
