# 02 — Instalasi ArgoCD, Application CRD, dan Siklus Sinkronisasi: Mengoperasikan Mesin Rekonsiliasi GitOps

> **Ringkasan:** Setelah menetapkan repositori Git sebagai satu-satunya sumber kebenaran (*Source of Truth*), langkah berikutnya adalah menugaskan agen rekonsiliasi yang bertugas membaca deklarasi tersebut dan mewujudkannya ke dalam klaster Kubernetes. **ArgoCD** adalah sistem pengiriman berkelanjutan deklaratif (*Declarative GitOps Continuous Delivery*) yang paling dominan di ekosistem Kubernetes. Dokumen ini membedah arsitektur lima komponen internal ArgoCD, prosedur instalasi yang diperkuat (*hardened deployment*), perancangan Custom Resource Definition (CRD) `Application`, pemahaman matriks status sinkronisasi (*Sync vs Health*), serta strategi eksekusi pemulihan deviasi.

---

## 1. Analogi Pemula: Sistem Autopilot Pesawat Komersial

Untuk memahami bagaimana ArgoCD bekerja di dalam klaster Kubernetes, mari gunakan analogi sistem penerbangan modern:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI SISTEM AUTOPILOT vs KONTROLER ARGOCD                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ RENCANA PENERBANGAN RESMI (DESIRED STATE / GITOPS REPO) ]                │
│  - Dokumen flight plan terdaftar di menara bandara:                         │
│    "Terbang di ketinggian 35.000 kaki, arah 270 derajat barat, 3 mesin ON". │
│                                                                             │
│  [ KONDISI FISIK PESAWAT DI UDARA (LIVE STATE / K3S CLUSTER) ]              │
│  - Pesawat sedang terbang di langit terbuka.                                │
│                                                                             │
│  [ GANGGUAN TURBULENSI & DRIFT ]                                            │
│  - Tiba-tiba ada hembusan angin kencang dari samping yang membelokkan       │
│    pesawat ke arah 280 derajat (**Drift: OutOfSync!**).                     │
│                                                                             │
│  [ REAKSI KOMPUTER AUTOPILOT (ARGOCD CONTROLLER) ]                          │
│  - Sensor kompas membaca perbedaan 10 derajat antara arah nyata dan flight  │
│    plan resmi.                                                              │
│  - Komputer seketika menggerakkan sirip kemudi sayap secara otomatis        │
│    (**Automated Reconciliation / Self-Healing**)!                           │
│  - Pesawat kembali persis ke arah 270 derajat dalam hitungan detik tanpa    │
│    perlu pilot menyentuh tuas kemudi manual!                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Arsitektur Komponen Internal ArgoCD

ArgoCD diinstal di dalam klaster Kubernetes pada namespace `argocd`. Arsitekturnya terdiri dari 5 komponen utama yang bekerja secara sinergis:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ARSITEKTUR LIMA KOMPONEN UTAMA ARGOCD                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. [ ARGOCD-SERVER ] (API Gateway & Web Dashboard)                         │
│     - Melayani UI berbasis browser dan CLI via protokol gRPC / REST HTTPS.  │
│     - Menangani otentikasi pengguna, otorisasi RBAC, dan audit log sesi.    │
│                                     │                                       │
│                                     ▼                                       │
│  2. [ ARGOCD-REPO-SERVER ] (Mesin Kloning & Generator Manifest)             │
│     - Bertugas mengklon repositori Git internal / OCI Registry.             │
│     - Mengeksekusi rendering Kustomize, Helm Template, atau Jsonnet.        │
│     - Menghasilkan output manifest Kubernetes mentah dalam format JSON.     │
│                                     │                                       │
│                                     ▼                                       │
│  3. [ ARGOCD-APPLICATION-CONTROLLER ] (Otak Rekonsiliasi SRE)               │
│     - Terus-menerus membandingkan:                                          │
│       Desired State (dari Repo Server) vs Live State (dari Kube-APIServer). │
│     - Menentukan status Sync (`Synced` vs `OutOfSync`).                     │
│     - Menerapkan aksi `selfHeal` dan `prune` jika terjadi penyimpangan!     │
│                                     │                                       │
│  4. [ ARGOCD-DEX-SERVER ]           │  5. [ ARGOCD-REDIS ]                  │
│     - Penyedia integrasi identitas  │     - Basis data cache in-memory      │
│       OIDC/OAuth2 ke GitLab, Okta,  │       untuk mengurangi beban polling  │
│       Keycloak, atau GitHub SSO.    │       ke repositori Git eksternal.    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Instalasi ArgoCD Non-Root & Konfigurasi Akses di k3s On-Prem

Di lingkungan bare-metal k3s, kita menginstal ArgoCD versi stabil resmi menggunakan manifest non-root yang aman:

### 3.1 Langkah Instalasi via Terminal
```bash
# 1. Buat namespace terisolasi untuk komponen ArgoCD
kubectl create namespace argocd

# 2. Pasang manifest resmi rilis stabil ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.3/manifests/install.yaml

# 3. Tunggu hingga seluruh controller berstatus Running
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=300s
```

### 3.2 Mengamankan Akses Awal Admin
Secara default, password admin awal di-generate secara acak ke dalam Secret Kubernetes:
```bash
# Mengambil password awal admin (Base64 decoded)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Login pertama kali via ArgoCD CLI lokal
argocd login localhost:8080 --username admin --password <password-dari-secret>

# Ganti password admin ke password kuat baru
argocd account update-password

# HAPUS SECRET PASSWORD AWAL UNTUK MEMENUHI ATURAN KEAMANAN SRE!
kubectl -n argocd delete secret argocd-initial-admin-secret
```

### 3.3 Eksposur UI ArgoCD via MetalLB LoadBalancer VIP
Agar dashboard ArgoCD dapat diakses langsung oleh tim SRE melalui jaringan lokal tanpa port-forwarding, ubah tipe Service `argocd-server` menjadi `LoadBalancer`:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
MetalLB yang telah kita konfigurasi di Fase 2 akan secara otomatis mengalokasikan satu IP VIP dari pool jaringan fisik (misal: `192.168.10.150`). Anda dapat langsung membuka `https://192.168.10.150` di browser host Mac!

---

## 4. Anatomi Custom Resource Definition `Application`

Objek utama yang menghubungkan repositori GitOps dengan klaster Kubernetes adalah CRD **`Application`** (`argoproj.io/v1alpha1`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-prod
  namespace: argocd
  # Finalizer menjamin penghapusan objek di klaster jika Application dihapus
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  # 1. SUMBER KEBENARAN (SOURCE / DESIRED STATE)
  source:
    repoURL: "https://gitlab.internal.corp/bootcamp-sre/gitops-manifests.git"
    targetRevision: main # Di produksi, wajib mengarah ke branch terproteksi atau commit SHA!
    path: apps/payment-service/envs/production
    # Jika menggunakan Helm Chart, tentukan berkas values:
    helm:
      valueFiles:
        - values-prod.yaml

  # 2. TARGET LOKASI PENERAPAN (DESTINATION / LIVE STATE)
  destination:
    server: "https://kubernetes.default.svc" # Mengarah ke API cluster lokal
    namespace: production

  # 3. KEBIJAKAN SINKRONISASI (SYNC POLICY & GUARDRAILS)
  syncPolicy:
    automated:
      prune: true     # Hapus resource di klaster jika resource tersebut dihapus dari Git!
      selfHeal: true  # Timpa kembali jika seseorang mengubah resource via kubectl manual!
      allowEmpty: false # Mencegah penghapusan massal jika repositori Git kosong tak sengaja!
    syncOptions:
      - CreateNamespace=true       # Buat namespace target jika belum ada
      - ApplyOutOfSyncOnly=true    # Hemat CPU: hanya apply resource yang benar-benar berubah
      - PruneLast=true             # Hapus resource lama HANYA setelah resource baru Ready!
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## 5. Memahami Matriks Status: Sync Status vs Health Status

Bagi seorang pemula, dua indikator status di dashboard ArgoCD sering kali membingungkan:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 MATRIKS DUA DIMENSI STATUS APLIKASI ARGOCD                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ DIMENSI 1: SYNC STATUS (Apakah Manifest Cocok dengan Git?) ]             │
│  - Synced    : 100% manifest di klaster sama persis dengan file di Git.    │
│  - OutOfSync : Ada perbedaan teks/versi antara Git dan klaster (Ada Drift!).│
│                                                                             │
│  [ DIMENSI 2: HEALTH STATUS (Apakah Aplikasi Berfungsi Hidup di Klaster?) ] │
│  - Healthy     : Seluruh Pod berstatus Running dan lolos Readiness Probe.   │
│  - Progressing : Pod sedang dalam proses rollout rolling update bertahap.   │
│  - Degraded    : Pod mengalami CrashLoopBackOff, OOMKilled, atau Error!     │
│  - Missing     : Resource yang dideklarasikan di Git belum terbuat di k8s.  │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│  CONTOH KASUS ANOMALI KRITIS:                                               │
│  Status: "SYNCED" tetapi "DEGRADED"                                         │
│  Artinya: ArgoCD sukses menerapkan manifest dari Git ke Kubernetes          │
│  (Sync sukses), TETAPI di dalam klaster Pod kontainer mengalami CRASH LOOP! │
│  SRE tidak boleh puas hanya melihat label "Synced"!                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN ARGOCD APPLICATION               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "ComparisonError: repo-server timeout after 90s" ]              │
│  Gejala: Aplikasi berwarna kuning dengan status `ComparisonError`.          │
│  Akar Masalah:                                                              │
│  - Komponen `argocd-repo-server` kehabisan memori RAM saat merender chart   │
│    Helm yang sangat besar, atau koneksi Git clone ke GitLab lokal timeout.  │
│  Solusi SRE:                                                                │
│  1. Naikkan resource limits pada deployment `argocd-repo-server`:           │
│     kubectl -n argocd set resources deploy argocd-repo-server               │
│       --limits=cpu=1,memory=1Gi                                             │
│  2. Cek konektivitas DNS dari dalam Pod repo-server:                        │
│     kubectl exec -it -n argocd deploy/argocd-repo-server -- nslookup gitlab │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "APPLICATION SYNCED TAPI HEALTH STATUS DEGRADED" ]              │
│  Gejala: Warna merah menyala di dashboard ArgoCD.                           │
│  Akar Masalah:                                                              │
│  - Manifest berhasil di-apply, tetapi *Image Digest* salah ketik sehingga   │
│    Pod berstatus `ErrImagePull`, atau PVC storage belum berhasil ter-bound. │
│  Solusi SRE:                                                                │
│  1. Buka pohon resource di UI ArgoCD, cari Pod yang bertanda silang merah.  │
│  2. Inspeksi event pod langsung dari terminal:                              │
│     kubectl describe pod -n production -l app.kubernetes.io/name=payment    │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: TARGET NAMESPACE FORBIDDEN / MISSING ]                          │
│  Gejala: Sync gagal dengan pesan: `namespaces "production" not found`.      │
│  Akar Masalah:                                                              │
│  - Namespace tujuan belum dibuat dan Anda lupa menyertakan opsi             │
│    `CreateNamespace=true` pada `syncOptions`.                               │
│  Solusi SRE:                                                                │
│  Tambahkan blok `CreateNamespace=true` pada CRD Application Anda.           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menampilkan seluruh aplikasi yang dikelola ArgoCD beserta status kesehatannya
argocd app list

# 2. Mengambil detail informasi dan riwayat sinkronisasi aplikasi
argocd app get payment-service-prod

# 3. Memicu sinkronisasi manual seketika dari terminal
argocd app sync payment-service-prod

# 4. Melakukan sinkronisasi hanya untuk resource tertentu yang OutOfSync
argocd app sync payment-service-prod --resource apps:Deployment:payment-service

# 5. Membatalkan operasi sinkronisasi yang sedang berjalan / macet
argocd app terminate-op payment-service-prod

# 6. Menghapus aplikasi dari ArgoCD tanpa menghapus objek fisiknya di Kubernetes
argocd app delete payment-service-prod --cascade=false
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Dalam konfigurasi CRD ArgoCD Application, apa konsekuensi operasional dari menyetel parameter `selfHeal: true` pada blok `syncPolicy.automated`?
- A. ArgoCD otomatis memperbesar kapasitas CPU node jika beban tinggi.
- B. Jika seorang engineer secara manual mengubah konfigurasi objek di klaster menggunakan perintah `kubectl edit` atau `kubectl scale`, ArgoCD akan seketika mendeteksi deviasi tersebut dan secara otomatis menimpa kembali perubahan manual tersebut sesuai deklarasi di repositori Git.
- C. ArgoCD otomatis menghapus namespace yang tidak terpakai.
- D. Mengubah seluruh password admin klaster menjadi acak.

*Kunci Jawaban:* **B**. Fitur `selfHeal` adalah instrumen penegak kedaulatan Git yang membasmi *manual configuration drift* di sisi klaster fisik.

### Soal 2 (Pilihan Ganda)
Sebuah aplikasi di dashboard ArgoCD menampilkan status ganda: **`Synced`** dan **`Degraded`**. Bagaimana seorang SRE menginterpretasikan situasi ini?
- A. Dashboard ArgoCD sedang mengalami bug tampilan dan harus di-refresh.
- B. Seluruh manifest yang tertulis di Git telah sukses diterapkan 100% ke Kube-APIServer (Synced), namun di tingkat runtime klaster, Pod kontainer mengalami kegagalan fungsi (seperti CrashLoopBackOff atau gagal Readiness Probe) sehingga aplikasi tidak melayani trafik secara sehat (Degraded).
- C. Klaster kehabisan alamat IP LoadBalancer.
- D. Koneksi internet router data center terputus total.

*Kunci Jawaban:* **B**. `Synced` mengukur kecocokan deklarasi manifest teks, sedangkan `Health` mengukur kesiapan operasional beban kerja kontainer secara nyata di klaster.

### Soal 3 (Pilihan Ganda)
Komponen internal ArgoCD manakah yang bertugas melakukan proses kloning repositori Git, mengurai berkas Helm/Kustomize, dan merender manifest teks mentah untuk diserahkan ke Application Controller?
- A. `argocd-server`
- B. `argocd-repo-server`
- C. `argocd-dex-server`
- D. `argocd-redis`

*Kunci Jawaban:* **B**. `argocd-repo-server` adalah mesin pemroses (*manifest generation engine*) yang bertugas menerjemahkan format GitOps menjadi manifest Kubernetes murni.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Di lingkungan staging, tim QA meminta agar setiap kali ada commit baru di GitOps repo, ArgoCD langsung melakukan sinkronisasi otomatis (*Full Automated Sync*). Namun di lingkungan produksi, Lead SRE melarang sinkronisasi otomatis 24 jam karena dikhawatirkan perubahan kode masuk saat jam sibuk transaksi perbankan (pukul 09.00 - 17.00 WIB). Rancang strategi konfigurasi ArgoCD yang dapat membatasi sinkronisasi otomatis produksi hanya di luar jam sibuk bisnis!

*Panduan Solusi:*
1. **Analisis Solusi:**
   ArgoCD menyediakan fitur resmi bernama **Sync Windows** yang didefinisikan pada objek `AppProject`. Sync Windows dapat memblokir (*deny*) atau mengizinkan (*allow*) sinkronisasi otomatis berdasarkan jadwal ekspresi cron.
2. **Konfigurasi `AppProject` dengan Sync Windows:**
   Terapkan konfigurasi berikut pada Project produksi di namespace `argocd`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AppProject
   metadata:
     name: production-project
     namespace: argocd
   spec:
     syncWindows:
       # BLOKIR SELURUH SINKRONISASI OTOMATIS SAAT JAM SIBUK BISNIS!
       - kind: deny
         schedule: "0 9 * * 1-5" # Setiap Senin - Jumat pukul 09.00 WIB
         duration: 8h            # Berlangsung selama 8 jam (hingga pukul 17.00 WIB)
         applications:
           - "*-prod"
         manualSync: true         # Izinkan jika operator darurat menekan tombol manual!
   ```
   Dengan arsitektur ini, jika ada perubahan di Git pada pukul 11.00 siang, ArgoCD akan menahan sinkronisasi otomatis (*Sync Blocked by Window*). Begitu jam kantor berakhir pada pukul 17.01 WIB, jendela sinkronisasi terbuka dan ArgoCD memperbarui klaster secara otomatis dan aman!

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — GitOps Source of Truth dan Struktur Repository](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/01-gitops-source-of-truth-repo-structure.md) — Prinsip dasar dan pemisahan repositori GitOps.
- **Topik Berikutnya:**
  - [03 — ApplicationSet, Secrets Boundary, Drift, dan Self-Healing](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/03-applicationset-secrets-drift-selfheal.md) — Mengorkestrasi puluhan aplikasi secara dinamis dengan generator ApplicationSet.
  - [LAB-01 — ArgoCD Application Sync](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/lab/LAB-01-argocd-application-sync.md) — Praktik instalasi dan menghubungkan aplikasi di klaster lab.
