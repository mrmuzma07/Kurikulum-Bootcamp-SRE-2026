# 03 — ApplicationSet, Secrets Boundary, Drift, dan Self-Healing: Otomasi Skala Multi-Klaster dan Manajemen Rahasia

> **Ringkasan:** Ketika organisasi berkembang dari beberapa aplikasi menjadi puluhan mikroservis yang tersebar di berbagai lingkungan (*Dev, Staging, Production*) dan multi-klaster, menulis manifest CRD `Application` secara manual satu per satu adalah bentuk pemborosan tenaga (*toil*) yang memicu inkonsistensi. **ArgoCD ApplicationSet** hadir sebagai mesin generator otomatis yang memproduksi objek `Application` secara dinamis. Di saat yang sama, paradigma GitOps menuntut solusi ketat terkait batas kerahasiaan (*Secrets Boundary*) dan pemulihan penyimpangan otomatis (*Drift & Self-Healing*). Dokumen ini membedah arsitektur generator ApplicationSet, integrasi rahasia modern (*Sealed Secrets vs External Secrets Operator*), serta mitigasi risiko penghapusan massal (*Accidental Pruning*).

---

## 1. Analogi Pemula: Manajer Armada Taksi Global & Kunci Brankas Digital

Untuk memahami skala yang diselesaikan oleh ApplicationSet dan tantangan data rahasia di Git, mari gunakan analogi armada taksi:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI MANAJER ARMADA TAKSI vs APPLICATIONSET              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PENDEKATAN MANUAL (CRD APPLICATION SATU PER SATU) ]                      │
│  - Masalah: Perusahaan memiliki 50 mikroservis dan 3 lingkungan (Staging,   │
│    Production-DC1, Production-DC2).                                         │
│  - Total ada: 50 x 3 = **150 file YAML Application terpisah**!               │
│  - Jika Anda ingin mengubah port metrik atau timeout sinkronisasi, Anda     │
│    harus mengedit 150 file satu per satu. Melelahkan dan rawan salah ketik! │
│                                                                             │
│  [ PENDEKATAN APPLICATIONSET: PABRIK PERAKITAN ROBOTIK OTOMATIS ]           │
│  - Anda cukup menulis **SATU CETAK BIRU ApplicationSet** dengan Generator!  │
│  - Generator memindai folder Git: "Oh, ada folder baru `apps/auth/`!"       │
│  - Dalam 1 detik, robotik ApplicationSet otomatis mencetak 3 objek rilis    │
│    sekaligus: `auth-staging`, `auth-prod-dc1`, dan `auth-prod-dc2`!        │
│                                                                             │
│  [ TANTANGAN KUNCI BRANKAS (SECRETS BOUNDARY DI GITOPS): ]                  │
│  Semua orang tahu GitOps mewajibkan seluruh konfigurasi disimpan di Git.    │
│  Tapi menaruh password database di Git sama saja dengan menempelkan kunci   │
│  brankas bank di kaca etalase depan toko yang bisa dilihat semua pejalan    │
│  kaki (**Git Plaintext Leak**)!                                             │
│                                                                             │
│  SOLUSI SRE: Kunci brankas harus dibungkus segel digital (*Sealed Secrets*) │
│  atau diambil langsung dari brankas pusat via kawat bawah tanah             │
│  (**External Secrets Operator / HashiCorp Vault**)!                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Arsitektur ApplicationSet & Tiga Generator Utama

**ApplicationSet Controller** adalah ekstensi native ArgoCD yang bertugas memproduksi objek `Application` secara dinamis menggunakan parameter yang dihasilkan oleh **Generators**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR KERJA GENERATOR APPLICATIONSET                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ GENERATOR INPUT ]                                                        │
│  1. Git Directory Generator  -> Memindai direktori `apps/*` di Git.         │
│  2. List Generator           -> Membaca daftar statis lingkungan.           │
│  3. Cluster Generator        -> Membaca seluruh klaster yang terdaftar.     │
│                                     │                                       │
│                                     ▼ Parameter: {{path.basename}}, {{cluster}}
│                                                                             │
│  [ APPLICATION TEMPLATE ]                                                   │
│  - Cetak biru tunggal: metadata.name = "{{path.basename}}-{{env}}"          │
│  - destination.namespace = "{{path.basename}}"                              │
│                                     │                                       │
│                                     ▼ Dievaluasi secara otomatis            │
│                                                                             │
│  [ HASIL DI CLUSTER: N-APPLICATIONS TERBENTUK INSTAN ]                      │
│  ├── Application: payment-service-staging                                   │
│  ├── Application: payment-service-production                                │
│  ├── Application: catalog-service-staging                                   │
│  └── Application: catalog-service-production                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementasi Nyata: Git Directory Generator (`applicationset.yaml`)
Cetak biru di bawah ini akan secara otomatis membuatkan aplikasi baru setiap kali ada folder microservice baru yang ditambahkan developer ke direktori `apps/`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices-appset
  namespace: argocd
spec:
  # 1. GENERATOR: Memindai seluruh folder di dalam apps/
  generators:
    - git:
        repoURL: "https://gitlab.internal.corp/bootcamp-sre/gitops-manifests.git"
        revision: main
        directories:
          - path: "apps/*"

  # 2. TEMPLATE APLIKASI YANG AKAN DIGENERATE
  template:
    metadata:
      # Nama aplikasi dinamis: misal payment-service, catalog-service
      name: "{{path.basename}}-prod"
      namespace: argocd
      labels:
        environment: production
        managed-by: applicationset
    spec:
      project: default
      source:
        repoURL: "https://gitlab.internal.corp/bootcamp-sre/gitops-manifests.git"
        targetRevision: main
        path: "{{path}}/envs/production"
      destination:
        server: "https://kubernetes.default.svc"
        namespace: "{{path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ApplyOutOfSyncOnly=true
```

---

## 3. Batasan Rahasia di GitOps (Secrets Management Boundary)

Menyimpan Secret Kubernetes mentah (`type: Opaque` dengan Base64) di repositori Git adalah **pelanggaran keamanan mutlak**. Ingat: Base64 bukanlah enkripsi, melainkan hanya pengubahan format teks yang bisa di-decode oleh siapa saja dalam 1 detik!

Di lingkungan *on-premises*, tim SRE menerapkan dua pola pengelolaan rahasia GitOps terpopuler:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DUA STRATEGI UTAMA MANAJEMEN RAHASIA GITOPS                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ STRATEGI 1: ENKRIPSI ASIMETRIS (SEALED SECRETS / SOPS) ]                 │
│  - Developer mengenkripsi password menggunakan kunci publik klaster:        │
│    `kubeseal < secret.yaml > sealed-secret.yaml`                            │
│  - Berkas `sealed-secret.yaml` aman disimpan di Git! Siapa pun boleh baca!  │
│  - Di dalam klaster, kontroler Sealed Secrets mendekripsinya menjadi        │
│    Secret k8s biasa menggunakan **Kunci Privat** yang tersimpan di etcd.    │
│                                                                             │
│  [ STRATEGI 2: OPERATOR RAHASIA EKSTERNAL (EXTERNAL SECRETS OPERATOR - ESO)│
│  - Repositori Git HANYA menyimpan alamat rujukan (*Pointer* / Metadata):    │
│    "Tolong ambil password database dari HashiCorp Vault di path: /secret/db"│
│  - Operator ESO di dalam klaster secara berkala mengambil nilai asli dari   │
│    Vault/MinIO dan membuat Secret k8s secara dinamis di memori klaster.     │
│  - *Keunggulan SRE:* Mendukung rotasi kata sandi otomatis (*Auto-Rotation*) │
│    tanpa perlu melakukan git commit baru!                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Contoh Manifest External Secrets Operator (`externalsecret.yaml`)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-database-credentials
  namespace: production
spec:
  refreshInterval: "1h" # Sinkronisasi otomatis setiap 1 jam jika ada rotasi password!
  secretStoreRef:
    name: onprem-vault-backend
    kind: ClusterSecretStore
  target:
    name: payment-db-secret # Nama Secret asli yang akan tercipta di Kubernetes
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: "production/payment/database"
        property: password
```

---

## 4. Mekanisme Pendeteksian Drift & Remediasi Self-Healing

Bagaimana ArgoCD mendeteksi adanya manipulasi ilegal pada klaster fisik?

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 SIKLUS DETEKSI DRIFT DAN REMEDIASI SELF-HEALING             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. STATUS NORMAL: Klaster Sinkron (Git: 3 Replicas <==> k8s: 3 Replicas)  │
│                                                                             │
│  2. TERJADI MUTASI ILEGAL DI SERVER:                                        │
│     Seorang teknisi menjalankan: `kubectl scale deployment app --replicas=0`│
│                                                                             │
│  3. SIKLUS DETEKSI (SETIAP 3 MENIT ATAU EVENT-BASED):                       │
│     `argocd-application-controller` membandingkan JSON Live vs Desired:     │
│     - Desired Spec: `replicas: 3`                                           │
│     - Live Spec   : `replicas: 0`                                           │
│     => DRIFT DITEMUKAN! Status rilis berubah menjadi: **OutOfSync**!        │
│                                                                             │
│  4. AKSI AUTOMATED SELF-HEALING BERJALAN:                                   │
│     ArgoCD memicu penimpaan paksa (*reconciliation apply*):                 │
│     Mengirim permintaan PATCH ke Kube-APIServer: `replicas: 3`.             │
│                                                                             │
│  5. STATUS KEMBALI SEHAT:                                                   │
│     Dalam 5 detik, jumlah Pod kembali menjadi 3 replika!                    │
│     Tindakan manual teknisi dibatalkan secara elegan (*Auto-Reverted*)!     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Bahaya Fatal: Bencana Penghapusan Massal (*Accidental Prune Catastrophe*)

Salah satu parameter paling berbahaya pada ArgoCD adalah:
```yaml
syncPolicy:
  automated:
    prune: true
```
**`prune: true`** berarti: jika sebuah manifest dihapus dari Git, ArgoCD akan **SEKETIKA MENGHAPUS** objek tersebut dari klaster Kubernetes.

### Skenario Horor:
Seorang junior engineer salah melakukan rebase Git, yang mengakibatkan seluruh berkas di folder `apps/` terhapus pada branch `main`.
Jika ArgoCD berjalan dengan `prune: true`, ArgoCD akan menyimpulkan: *"Oh, di Git kosong, berarti di klaster juga harus dikosongkan!"*.
**HASILNYA: PULUHAN DATABASE, STATEFULSET, DAN SERVICE PRODUKSI TERHAPUS BERSIH SECARA OTOMATIS DALAM 10 DETIK!**

### Perisai Keselamatan SRE (Safety Shields):
1. **Gunakan Parameter `allowEmpty: false`:**
   Mencegah ArgoCD menghapus seluruh aplikasi jika repositori Git terbaca kosong tak sengaja:
   ```yaml
   syncPolicy:
     automated:
       prune: true
       allowEmpty: false # JANGAN PERNAH PRUNE JIKA GIT KOSONG!
   ```
2. **Kunci Objek Sensitif dengan Anotasi Prune Protection:**
   Tambahkan anotasi ini pada PersistentVolumeClaim (PVC) atau Database agar kebal dari penghapusan ArgoCD:
   ```yaml
   metadata:
     annotations:
       argocd.argoproj.io/sync-options: Prune=false
   ```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA APPLICATIONSET & SECRETS                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: SEALED SECRETS GAGAL DIDISKRIPSI PASCA CLUSTER REBUILD ]        │
│  Gejala: Pod gagal start: "Secret payment-db-secret not found".             │
│          Log kontroler sealed-secrets: "cannot decrypt secret: private key" │
│  Akar Masalah:                                                              │
│  - Klaster k3s baru saja dibangun ulang dari nol (*Total Rebuild*). Kunci   │
│    privat dekripsi lama yang ada di etcd ikut musnah, sehingga file sealed  │
│    di Git tidak bisa didekripsi oleh klaster baru!                          │
│  Solusi SRE:                                                                │
│  1. Selalu backup kunci privat master Sealed Secrets ke MinIO S3 terenkripsi│
│     pada saat inisialisasi awal klaster:                                    │
│     kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-master-key.yaml│
│  2. Saat rebuild, restore kunci master tersebut sebelum ArgoCD aktif!       │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: APPLICATIONSET GENERATOR MELEMPAR NAMA KARAKTER INVALID ]      │
│  Gejala: ApplicationSet gagal men-generate aplikasi: "name invalid".        │
│  Akar Masalah:                                                              │
│  - Developer membuat folder di Git dengan huruf kapital atau underscore:    │
│    `apps/Payment_Service/`. Standar nama Kubernetes HANYA mengizinkan huruf │
│    kecil dan tanda minus (RFC 1123).                                        │
│  Solusi SRE:                                                                │
│  Wajibkan standarisasi penamaan folder di Git: huruf kecil alfanumerik.     │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: EXTERNAL SECRETS TIMEOUT MENGHUBUNGI VAULT ]                    │
│  Gejala: Status ExternalSecret berstatus `SecretSyncedError`.               │
│  Akar Masalah:                                                              │
│  - Token akses ESO ke Vault kedaluwarsa atau firewall memblokir port 8200.  │
│  Solusi SRE:                                                                │
│  1. Cek status controller: `kubectl describe externalsecret payment-secret` │
│  2. Validasi endpoint backend Vault dari dalam klaster k3s.                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menampilkan seluruh status ApplicationSet yang aktif di klaster
kubectl get applicationsets -n argocd

# 2. Mengambil detail evaluasi generator pada ApplicationSet
kubectl describe applicationset microservices-appset -n argocd

# 3. Mengenkripsi berkas Secret mentah menjadi SealedSecret menggunakan kubeseal
kubeseal --format=yaml --cert=pub-sealed-cert.pem < secret.yaml > sealed-secret.yaml

# 4. Mengecek status sinkronisasi objek ExternalSecrets Operator
kubectl get externalsecrets -A

# 5. Memeriksa daftar resource yang ditandai untuk di-prune oleh ArgoCD
argocd app get payment-service-prod --show-params
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa menyimpan berkas Secret Kubernetes mentah (`type: Opaque` berisi teks Base64) di dalam repositori GitOps publik maupun privat dianggap sebagai pelanggaran kepatuhan keamanan tingkat tinggi?
- A. Karena Kubernetes menolak membaca Secret yang berukuran di atas 10 baris.
- B. Karena format Base64 hanyalah algoritma encoding teks dua arah tanpa kata sandi (*reversible encoding*), yang dapat di-decode kembali menjadi teks kata sandi asli dalam hitungan detik oleh siapa saja yang memiliki akses baca ke repositori Git.
- C. Karena Git otomatis menghapus berkas yang memuat kata sandi.
- D. Agar ukuran memori RAM server k3s tidak habis.

*Kunci Jawaban:* **B**. Base64 bukanlah bentuk enkripsi; menyimpan plaintext secret di Git merusak seluruh postur keamanan data perusahaan.

### Soal 2 (Pilihan Ganda)
Dalam arsitektur *External Secrets Operator (ESO)*, apa yang sebenarnya disimpan oleh tim SRE di dalam repositori GitOps?
- A. Nilai kata sandi database yang dienkripsi dengan password zip.
- B. Hanyalah metadata rujukan (*Pointer Declaration* seperti nama secret store dan path rujukan di Vault eksternal), tanpa memuat sebaris pun nilai rahasia asli di dalam Git.
- C. Kunci master privat seluruh data center.
- D. Berkas ISO instalasi sistem operasi Linux.

*Kunci Jawaban:* **B**. Pendekatan ESO sepenuhnya memisahkan data rahasia dari repositori Git dan menjaganya tetap berada di brankas terpusat (*Secret Store Vault*).

### Soal 3 (Pilihan Ganda)
Apa tujuan utama menyertakan anotasi `argocd.argoproj.io/sync-options: Prune=false` pada manifest PersistentVolumeClaim (PVC) aplikasi database produksi?
- A. Agar harddisk storage otomatis digandakan menjadi 2 unit.
- B. Mengunci objek storage PVC agar tidak pernah dihapus secara otomatis oleh fitur *pruning* ArgoCD meskipun berkas manifest PVC tersebut terhapus atau dipindahkan di repositori Git, melindungi data fisik dari kehancuran tak sengaja.
- C. Memaksa database berjalan tanpa menggunakan ruang disk.
- D. Mengubah PVC menjadi RAM disk in-memory.

*Kunci Jawaban:* **B**. Anotasi perlindungan prune adalah jaring pengaman wajib untuk melindungi data persisten stateful dari bencana *accidental pruning*.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Perusahaan Anda baru saja mengalami insiden kebakaran fisik data center dan seluruh 3 server master k3s musnah. Anda membangun klaster k3s baru dari nol menggunakan OpenTofu dan Ansible. Ketika ArgoCD di-deploy dan menarik repositori GitOps yang memuat puluhan manifest `SealedSecret`, seluruh Pod aplikasi gagal start dengan pesan error `Secret Not Found`, dan log kontroler sealed-secrets melaporkan `cannot decrypt: private key mismatch`. Jelaskan mengapa hal ini terjadi dan bagaimana strategi pemulihan bencana (*Disaster Recovery*) yang seharusnya disiapkan sejak awal!

*Panduan Solusi:*
1. **Penyebab Kegagalan Dekripsi:**
   Kontroler Bitnami *Sealed Secrets* bekerja menggunakan kriptografi kunci asimetris. Berkas manifest di Git dienkripsi menggunakan *Kunci Publik*. Pada saat instalasi klaster pertama kali, kontroler secara otomatis men-generate pasangan *Kunci Privat* yang disimpan sebagai Secret internal di namespace `kube-system`. Ketika klaster lama terbakar dan klaster baru dibangun, kontroler baru membuat pasangan kunci privat baru yang berbeda. Akibatnya, manifest di Git yang terenkripsi dengan kunci lama mustahil didekripsi oleh kunci baru!
2. **Strategi Pemulihan Bencana Baku SRE (Sealed Secrets Backup Runbook):**
   - **Pencadangan Kunci Master Berkala ke Off-site MinIO S3:**
     Sejak hari pertama instalasi klaster, backup kunci master privat Sealed Secrets secara terenkripsi:
     ```bash
     kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > /secure-backup/sealed-secrets-key.yaml
     ```
     Kunci ini disimpan di penyimpanan dingin aman (*Off-site Cold Storage*) yang memiliki retensi multi-tahun.
   - **Prosedur Restore Pra-ArgoCD:**
     Saat klaster baru dibangun dari nol, SEBELUM ArgoCD atau Sealed Secrets Controller diizinkan melakukan sinkronisasi, suntikkan kembali berkas kunci master lama tersebut ke dalam klaster:
     ```bash
     kubectl apply -f /secure-backup/sealed-secrets-key.yaml -n kube-system
     ```
   - Dengan langkah ini, kontroler baru akan mengadopsi kunci privat lama yang sah, dan seketika seluruh puluhan `SealedSecret` di repositori GitOps dapat didekripsi kembali dengan sempurna tanpa perlu mengenkripsi ulang satu pun berkas!

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — GitOps Source of Truth dan Struktur Repository](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/01-gitops-source-of-truth-repo-structure.md) — Memahami pemisahan repositori GitOps.
  - [02 — Instalasi ArgoCD, Application CRD, dan Siklus Sinkronisasi](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/02-argocd-install-application-sync.md) — Mengoperasikan mesin sinkronisasi dasar.
- **Tahap Berikutnya:**
  - [Modul 6.3 — End-to-End Promotion Flow](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.3-end-to-end-flow/) — Menggabungkan GitLab CI dan ArgoCD ke dalam alur pengiriman progresif (*Canary & Blue-Green*).
  - [LAB-02 — Drift, Self-Healing, dan ApplicationSet](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.2-argocd/lab/LAB-02-drift-selfheal-applicationset.md) — Praktik hands-on pembuktian self-heal dan pengujian ApplicationSet.
