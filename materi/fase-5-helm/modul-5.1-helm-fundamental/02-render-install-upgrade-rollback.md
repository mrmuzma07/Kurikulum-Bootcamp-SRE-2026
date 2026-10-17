# 02 — Render dan Lifecycle Release: Mengendalikan Siklus Hidup Aplikasi dengan Guardrails Teruji

> **Ringkasan:** Mengoperasikan aplikasi di Kubernetes bukan hanya tentang kemampuan melakukan *deployment* pertama kali (*install*), melainkan bagaimana mengelola siklus perubahannya secara berkelanjutan: pembaruan versi (*upgrade*), pemantauan kesiapan (*health verification*), dan pemulihan instan saat bencana rilis terjadi (*rollback*). Di lingkungan *on-premises*, kegagalan rilis yang tidak ditangani dengan benar dapat mengunci status klaster ke dalam kondisi menggantung (*pending-upgrade deadlock*). Dokumen ini membedah siklus hidup rilis Helm (*Release Lifecycle*), menetapkan guardrail operasional berbasis `--wait`, `--timeout`, dan `--atomic`, serta mengurai realitas teknis apa yang bisa dan tidak bisa dipulihkan oleh mekanisme *rollback*.

---

## 1. Analogi Pemula: Rekam Medis Pasien & Tombol Restore Mesin Waktu

Untuk memahami bagaimana Helm melacak perubahan aplikasi di dalam klaster Kubernetes, mari gunakan analogi kartu rekam medis di rumah sakit:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI REKAM MEDIS RUMAH SAKIT vs HELM RELEASE             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ REKAM MEDIS RUMAH SAKIT ]                                                │
│  - Seorang pasien (Aplikasi Web) memiliki buku rekam medis tebal.           │
│  - Kunjungan 1 (Revisi 1): Dokter memberi obat flu dosis 10mg (Installed).  │
│  - Kunjungan 2 (Revisi 2): Dosis dinaikkan ke 20mg (Upgraded). Pasien sehat.│
│  - Kunjungan 3 (Revisi 3): Dokter mengganti obat eksperimental baru.        │
│    Seketika detak jantung pasien drop dan pingsan (**CrashLoopBackOff**)!   │
│  - Aksi Penyelamatan: Kepala Rumah Sakit membuka buku rekam medis, melihat  │
│    halaman Kunjungan 2, dan seketika menyuntikkan obat lama Revisi 2!       │
│    Pasien selamat dan stabil kembali (**ROLLBACK BERHASIL**)!               │
│                                                                             │
│  [ BATASAN REALITAS MEDIS (SRE WARNING): ]                                  │
│  Jika obat eksperimental pada Revisi 3 ternyata adalah operasi bedah yang   │
│  mengamputasi jari pasien (Migrasi skema database yang menghapus tabel),    │
│  membaca kembali resep Kunjungan 2 TIDAK BISA menumbuhkan jari yang hilang! │
│                                                                             │
│  Helm Rollback HANYA mengembalikan manifest konfigurasi Kubernetes,         │
│  BUKAN memulihkan data fisik di dalam database atau harddisk storage!       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Di Mana Helm Menyimpan Riwayat Rilis? (The Helm Secret Storage)

Banyak pemula mengira Helm membutuhkan basis data terpisah (seperti MySQL atau Redis) untuk mencatat riwayat rilis. Kenyataannya: **Helm versi 3 sepenuhnya stateless di sisi server**!

Setiap kali Anda menjalankan `helm install` atau `helm upgrade`, Helm membungkus manifest yang telah dirender, nilai values, dan status rilis menjadi sebuah **Secret Kubernetes** internal yang disimpan langsung di namespace rilis tersebut:

```bash
# Melihat Secret internal yang dikelola oleh Helm di namespace 'production'
kubectl get secrets -n production -l "owner=helm"
```

*Output yang Dihasilkan:*
```text
NAME                           TYPE                 DATA   AGE
sh.helm.release.v1.payment.v1  helm.sh/release.v1   1      3d
sh.helm.release.v1.payment.v2  helm.sh/release.v1   1      1d
sh.helm.release.v1.payment.v3  helm.sh/release.v1   1      10m
```

Data di dalam Secret tersebut di-encode menggunakan Base64 dan dikompresi dengan Gzip. Jika Anda menjalankan `helm rollback payment 2`, Helm membaca data dari Secret `sh.helm.release.v1.payment.v2`, lalu mengirimkan perbedaannya ke Kubernetes API!

### Siklus Status Rilis (Release Status Cycle)
Sebuah rilis Helm memiliki beberapa status resmi:
- `deployed` : Rilis berhasil terpasang dan manifest telah diterima oleh Kubernetes.
- `failed` : Operasi install atau upgrade mengalami kegagalan (misal: timeout atau probe gagal).
- `pending-install` / `pending-upgrade` : Helm sedang menunggu konfirmasi status dari Kubernetes API.
- `superseded` : Revisi lama yang sudah digantikan oleh revisi yang lebih baru.
- `uninstalled` : Rilis telah dihapus dari klaster.

---

## 3. Empat Guardrail Operasional Rilis Terkendali

Jangan pernah menjalankan `helm install` atau `helm upgrade` secara membabi-buta di lingkungan produksi. Terapkan **Empat Guardrail Operasional**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 EMPAT GUARDRAIL OPERASIONAL RILIS HELM                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ GUARDRAIL 1: VALIDASI KONTEKS KUBECONFIG & NAMESPACE ]                   │
│  - Sebelum menyentuh klaster, pastikan terminal mengarah ke target benar!   │
│  - Cek: `kubectl config current-context`                                    │
│  - Jangan sampai berniat deploy ke cluster Lab, tapi terminal mengarah      │
│    ke cluster Production data center!                                       │
│                                     │                                       │
│                                     ▼ LULUS                                 │
│                                                                             │
│  [ GUARDRAIL 2: DRY-RUN & DIFF AUDIT ]                                      │
│  - Simulasikan perubahan dan bandingkan selisih teks manifest:              │
│    `helm diff upgrade payment ./internal-app -f values-prod.yaml`           │
│  - Periksa apakah ada selector yang berubah atau kapasitas resource melonjak│
│                                     │                                       │
│                                     ▼ LULUS                                 │
│                                                                             │
│  [ GUARDRAIL 3: EKSEKUSI DENGAN --wait, --timeout, DAN --atomic ]           │
│  - Parameter wajib produksi:                                                │
│    `--wait`    : Tunggu sampai seluruh Pod berstatus Ready!                 │
│    `--timeout` : Batas waktu maksimal (misal: 300 detik).                   │
│    `--atomic`  : JIKA GAGAL -> OTOMATIS ROLLBACK KE REVISI SEBELUMNYA!     │
│                                     │                                       │
│                                     ▼ LULUS                                 │
│                                                                             │
│  [ GUARDRAIL 4: POST-RELEASE HEALTH SMOKE-TEST ]                            │
│  - Verifikasi status rollout dan uji endpoint HTTP:                         │
│    `kubectl rollout status deployment/payment -n production`                │
│    `helm test payment -n production`                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Alur Perintah Operasional Langkah Demi Langkah

### 4.1 Instalasi Pertama Kali (`helm install`)
Saat pertama kali memasang aplikasi ke klaster:

```bash
# Pastikan context mengarah ke cluster yang tepat
kubectl config current-context

# Instalasi aplikasi dengan perlindungan atomic dan wait
helm install payment ./internal-app \
  --namespace production \
  --create-namespace \
  --values values.yaml \
  --values values-prod.yaml \
  --wait \
  --timeout 5m0s
```

> [!NOTE]
> **Fungsi Flag `--wait`:**
> Tanpa `--wait`, Helm akan langsung melapor `STATUS: deployed` hanya dalam 1 detik setelah manifest dikirim ke Kube-APIServer, meskipun Pod kontainer masih berstatus `ContainerCreating` atau `ImagePullBackOff`. Dengan menyertakan `--wait`, Helm akan menahan terminal sampai seluruh Pod benar-benar lolos *Readiness Probe* dan Service memiliki *Endpoints* aktif!

### 4.2 Pembaruan Aplikasi Tanpa Downtime (`helm upgrade --atomic`)
Saat developer merilis versi baru aplikasi:

```bash
# Menjalankan upgrade dengan rollback otomatis jika terjadi kegagalan
helm upgrade payment ./internal-app \
  --namespace production \
  --values values.yaml \
  --values values-prod.yaml \
  --set image.tag="v2.5.0" \
  --wait \
  --timeout 5m0s \
  --atomic
```

> [!IMPORTANT]
> **Mengapa `--atomic` adalah Parameter Wajib di Pipeline CI/CD?**
> Jika Anda menjalankan upgrade tanpa `--atomic`, dan image baru ternyata mengalami *CrashLoopBackOff*, rilis Helm akan berakhir dengan status **`FAILED`** pada Revisi baru. Klaster Anda tertinggal dalam kondisi setengah matang (*half-baked*). Dengan menambahkan `--atomic`, jika dalam waktu 5 menit Pod baru tidak berhasil `Ready`, Helm akan secara otomatis mengeksekusi pemulihan instan (*self-healing rollback*) ke Revisi lama yang stabil!

### 4.3 Menginspeksi Riwayat & Riwayat Nilai (`helm history` & `helm get`)

```bash
# 1. Menampilkan seluruh riwayat revisi rilis secara kronologis
helm history payment --namespace production
```
*Output yang Dihasilkan:*
```text
REVISION  UPDATED                   STATUS      CHART               APP VERSION  DESCRIPTION
1         Mon Oct 12 10:00:00 2026  superseded  internal-app-1.0.0  2.4.0        Initial install
2         Tue Oct 13 14:30:00 2026  superseded  internal-app-1.1.0  2.4.1        Upgrade resources
3         Sat Oct 17 09:15:00 2026  deployed    internal-app-1.2.0  2.5.0        Upgrade release
```

```bash
# 2. Melihat nilai variabel spesifik yang digunakan pada revisi tertentu
helm get values payment --revision 2 --namespace production

# 3. Melihat manifest utuh hasil render yang sedang aktif di klaster
helm get manifest payment --namespace production
```

### 4.4 Mengembalikan Versi Rilis Secara Manual (`helm rollback`)
Jika rilis versi terbaru menghasilkan peningkatan *error rate* (misal HTTP 500 melonjak), SRE dapat memicu pembalikan instan ke nomor revisi yang stabil:

```bash
# Mengembalikan rilis ke Revisi 2
helm rollback payment 2 \
  --namespace production \
  --wait \
  --timeout 3m0s
```
*Catatan:* Tindakan rollback ini akan dicatat sebagai Revisi baru (Revisi 4) dengan deskripsi `Rollback to 2`. Riwayat audit tidak pernah dihapus!

---

## 5. Hierarki & Presedensi Nilai (Values Precedence)

Ketika parameter konfigurasi disuplai dari berbagai sumber, Helm menerapkan aturan presedensi yang ketat (dari prioritas paling rendah ke paling tinggi):

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 HIERARKI PRESEDENSI VALUES HELM (DARI LEMAH KE KUAT)        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TINGKAT 1: values.yaml BAWAAN CHART ] (Paling Lemah)                     │
│  Nilai default publik yang tersimpan di dalam paket chart.                  │
│                                     │                                       │
│                                     ▼ Ditimpa oleh                          │
│                                                                             │
│  [ TINGKAT 2: BERKAS VALUES PERTAMA (-f values-base.yaml) ]                 │
│  Berkas konfigurasi bersama (Common configurations).                       │
│                                     │                                       │
│                                     ▼ Ditimpa oleh                          │
│                                                                             │
│  [ TINGKAT 3: BERKAS VALUES KEDUA (-f values-prod.yaml) ]                   │
│  Berkas konfigurasi spesifik lingkungan produksi.                           │
│                                     │                                       │
│                                     ▼ Ditimpa oleh                          │
│                                                                             │
│  [ TINGKAT 4: ARGUMEN CLI (--set atau --set-string) ] (Paling Kuat Mutlak)  │
│  Injeksi langsung dari baris perintah terminal atau parameter CI/CD.       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!WARNING]
> **Anti-Pattern SRE: Bahaya Ketergantungan `--set`:**
> Hindari menggunakan terlalu banyak argumen `--set key=value` di terminal produksi. Nilai `--set` tidak tersimpan di repositori Git, melanggar prinsip *GitOps* dan *Infrastructure as Code*, serta menyulitkan proses reproduksi kegagalan. Selalu prioritaskan penggunaan berkas versi eksplisit: `-f values-prod.yaml`.

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN RILIS HELM                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "UPGRADE FAILED: ANOTHER OPERATION IS IN PROGRESS" ]            │
│  Gejala: `helm upgrade` ditolak karena rilis tersangkut di `pending-upgrade`.│
│  Akar Masalah:                                                              │
│  - Pipeline CI/CD terputus di tengah jalan (misal: di-cancel oleh developer)│
│    sehingga Helm Secret masih berstatus terkunci (*lock*).                  │
│  Solusi SRE:                                                                │
│  1. Cari Secret rilis terakhir yang macet:                                  │
│     kubectl get secrets -n production -l "owner=helm,name=payment"          │
│  2. Hapus Secret dengan revisi terakhir yang berstatus pending:             │
│     kubectl delete secret sh.helm.release.v1.payment.v3 -n production       │
│  3. Eksekusi ulang rilis dengan parameter `--atomic`.                       │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "UPGRADE FAILED: CANNOT BE APPLIED TO DEPLOYMENT: IMMUTABLE" ]  │
│  Gejala: Helm menolak upgrade: "field is immutable: spec.selector".         │
│  Akar Masalah:                                                              │
│  - Developer mengubah label pada `selector.matchLabels` di template.        │
│    Kubernetes API secara tegas melarang perubahan selector pada Deployment  │
│    yang sedang hidup!                                                       │
│  Solusi SRE:                                                                │
│  1. JANGAN PERNAH MENGGUNAKAN `--force`! Flag `--force` akan menghapus      │
│     Deployment dan seluruh Pod secara mendadak, menyebabkan DOWNTIME!       │
│  2. Kembalikan selector ke nilai lama di template.                          │
│  3. Jika perubahan selector mutlak diperlukan: buat Deployment baru dengan  │
│     nama berbeda, alihkan Service secara bertahap, lalu hapus Deployment    │
│     lama.                                                                   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: TIMEOUT PADA FLAG `--wait` KARENA CRASHLOOP ]                   │
│  Gejala: Helm menunggu 5 menit lalu melempar error:                         │
│          "timed out waiting for the condition".                             │
│  Akar Masalah:                                                              │
│  - Pod baru gagal startup karena kehabisan memory (OOMKilled) atau          │
│    koneksi database ditolak, sehingga *Readiness Probe* tidak pernah True.  │
│  Solusi SRE:                                                                │
│  1. Buka terminal baru dan inspeksi event pod:                              │
│     kubectl describe pod -l app.kubernetes.io/instance=payment -n prod      │
│  2. Cek log kontainer sebelumnya:                                           │
│     kubectl logs -l app.kubernetes.io/instance=payment -n prod --previous   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menampilkan seluruh daftar rilis Helm di semua namespace
helm list -A

# 2. Menampilkan status detail dari sebuah rilis (Resources, Pods, Notes)
helm status payment --namespace production

# 3. Menampilkan riwayat revisi dan status rilis
helm history payment --namespace production

# 4. Mengembalikan rilis ke nomor revisi tertentu
helm rollback payment 1 --namespace production --wait

# 5. Menghapus rilis dan seluruh objek Kubernetes terkait secara tuntas
helm uninstall payment --namespace production

# 6. Menghapus rilis tetapi tetap mempertahankan riwayat Secret di cluster
helm uninstall payment --namespace production --keep-history
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Apa yang membedakan eksekusi `helm upgrade payment ./internal-app --atomic` dibandingkan eksekusi `helm upgrade` biasa tanpa parameter tersebut?
- A. Parameter `--atomic` otomatis mengompresi gambar kontainer menjadi format OCI.
- B. Parameter `--atomic` mewajibkan `--wait`; jika ada satu saja Pod yang gagal berstatus Ready dalam batas waktu timeout, Helm akan secara otomatis membatalkan perubahan dan me-rollback rilis ke revisi stabil sebelumnya.
- C. Parameter `--atomic` menghapus seluruh namespace sebelum melakukan instalasi.
- D. Parameter `--atomic` mempercepat koneksi jaringan hingga 2 kali lipat.

*Kunci Jawaban:* **B**. Flag `--atomic` mengaktifkan mekanisme proteksi *all-or-nothing*, menjamin klaster tidak tertinggal dalam kondisi rilis yang rusak.

### Soal 2 (Pilihan Ganda)
Di manakah Helm versi 3 menyimpan catatan metadata riwayat rilis (*Release Revisions*) secara internal di dalam klaster Kubernetes?
- A. Di dalam file `/etc/helm/history.db` pada node master k3s.
- B. Di dalam etcd database eksternal yang di-host di MinIO S3.
- C. Disimpan sebagai objek Secret Kubernetes terenkripsi/terkompresi langsung di dalam *namespace* tempat rilis tersebut dideploy (`sh.helm.release.v1.*`).
- D. Di dalam direktori `.git` laptop operator.

*Kunci Jawaban:* **C**. Helm 3 memanfaatkan API Kubernetes secara native dengan menyimpan status rilis sebagai Secret bertipe `helm.sh/release.v1`.

### Soal 3 (Pilihan Ganda)
Seorang SRE menjalankan `helm rollback my-app 2` setelah rilis versi 3 mengalami kegagalan. Apa yang sebenarnya terjadi pada data di dalam database MySQL yang tersimpan pada PersistentVolumeClaim (PVC)?
- A. Data di dalam tabel database otomatis kembali ke kondisi waktu Revisi 2 dibuat.
- B. Seluruh data di harddisk storage otomatis terhapus bersih.
- C. Data fisik di dalam database TIDAK BERUBAH SAMA SEKALI, karena Helm rollback hanya mengembalikan manifest deklaratif Kubernetes (Deployment/ConfigMap), bukan memulihkan data storage stateful.
- D. Database MySQL otomatis melakukan restart 5 kali.

*Kunci Jawaban:* **C**. Helm bukanlah pengelola data stateful; pemulihan data aplikasi database wajib dilakukan menggunakan mekanisme *database restore/snapshot*, bukan Helm rollback.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Pada saat rilis terjadwal di pipeline GitLab CI, sebuah job `helm upgrade --wait` tiba-tiba mengalami *timeout* dan dibatalkan paksa (*aborted*) oleh pengguna. Saat pipeline berikutnya dijalankan, perintah Helm melempar error:
`Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress`
Jelaskan mengapa kondisi *deadlock* ini terjadi dan susun langkah-langkah resolusi teknis untuk memulihkan rilis agar pipeline CI/CD dapat berjalan kembali secara normal!

*Panduan Solusi:*
1. **Akar Masalah (Release Lock Deadlock):**
   Saat Helm menjalankan operasi upgrade, ia menandai Secret revisi aktif dengan status `pending-upgrade`. Jika proses dihentikan mendadak dari luar (misal: tombol Cancel di CI/CD atau kill sinyal), Helm tidak sempat menyelesaikan siklus untuk mengubah status menjadi `deployed` atau `failed`. Akibatnya, rilis terkunci (*locked*) untuk mencegah modifikasi ganda secara bersamaan.
2. **Langkah Resolusi Pemulihan:**
   - **Identifikasi Secret Terkunci:**
     Cari Secret rilis terakhir yang berstatus `pending-upgrade`:
     ```bash
     kubectl get secret -n <namespace> -l "owner=helm,name=<release-name>" --sort-by=.metadata.creationTimestamp
     ```
   - **Hapus Secret yang Menggantung:**
     Hapus Secret revisi paling terakhir yang macet (misal revisi 5):
     ```bash
     kubectl delete secret sh.helm.release.v1.<release-name>.v5 -n <namespace>
     ```
   - **Verifikasi Status Rilis:**
     Jalankan `helm history <release-name> -n <namespace>` untuk memastikan rilis kembali mengacu pada revisi stabil sebelumnya (Revisi 4 berstatus `deployed`).
   - **Pencegahan Masa Depan:**
     Selalu sertakan flag `--atomic` pada skrip pipeline agar Helm menangani kegagalan atau pembatalan secara otomatis tanpa meninggalkan status menggantung.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Struktur Chart, Values, dan Go-template](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/01-struktur-chart-values-template.md) — Mengemas aplikasi ke dalam Chart deklaratif.
- **Topik Berikutnya:**
  - [03 — Chart Repository dan OCI di GitLab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/03-repository-chart-oci-gitlab.md) — Memaketkan dan mempublikasikan chart ke Open Container Initiative (OCI) Registry.
  - [LAB-02 — Release Lifecycle dan Values Multi-Environment](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/lab/LAB-02-release-lifecycle-values.md) — Praktik simulasi rilis dan rollback di klaster.
