# 02 — Progressive Delivery dan Rollback

> **Ringkasan:** Memahami arsitektur rilis modern (*Progressive Delivery*) menggunakan pola **Canary Deployment** dan **Blue-Green Deployment** dengan **Argo Rollouts**, mengotomatisasi evaluasi metrik keandalan (*analysis-driven promotion*), serta mengeksekusi prosedur *instant rollback* saat terjadi anomali di lingkungan produksi on-premises.

---

## 1. Konsep & Arsitektur Progressive Delivery

### 1.1 Analogi Dunia Nyata: "Uji Klinis Obat Bertahap & Sakelar Wesel Kereta Cepat"
Bayangkan dua skenario pengelolaan risiko di dunia nyata:
1. **Uji Klinis Bertahap (*Canary Deployment*):** Ketika sebuah vaksin baru dikembangkan, produsen tidak langsung menyuntikkannya ke 100% populasi dalam satu hari. Mereka memberikan vaksin kepada 5% relawan terpilih (*canary group*), lalu memantau data detak jantung dan suhu tubuh selama 48 jam (*metric analysis*). Jika tidak ada efek samping negatif, distribusi diperluas ke 20%, 50%, hingga akhirnya ke 100% populasi. Jika pada tahap 5% terdeteksi alergi berbahaya, uji coba dihentikan seketika (*abort & rollback*), sehingga 95% populasi lainnya tetap aman.
2. **Sakelar Wesel Rel (*Blue-Green Deployment*):** Masinis kereta cepat memiliki dua jalur rel paralel: Jalur Biru (Jalur Utama yang sedang aktif) dan Jalur Hijau (Jalur Baru yang baru selesai diperbaiki). Seluruh penumpang tetap melintas di Jalur Biru hingga tim inspektur menyelesaikan uji kelayakan di Jalur Hijau. Begitu izin keluar, petugas memindahkan tuas wesel (*Service Switch*) dalam hitungan detik. Jika di tengah jalan ditemukan kendala, tuas wesel ditarik kembali ke Jalur Biru tanpa perlu membongkar gerbong kereta.

Di lingkungan produksi SRE, **Progressive Delivery** adalah evolusi dari *Continuous Delivery* yang menggabungkan rilis kode bertahap dengan observabilitas otomatis (*automated telemetry gates*). Alih-alih merilis perubahan secara "semua atau tidak sama sekali" (*all-or-nothing*), sistem secara dinamis mengalirkan sebagian kecil trafik nyata ke versi baru dan secara otonom membatalkan rilis jika batas toleransi kesalahan terlampaui.

---

### 1.2 Perbandingan 3 Strategi Rilis di Kubernetes

Kubernetes menyediakan mekanisme rilis bawaan, namun memiliki keterbatasan mendasar jika dibandingkan dengan pola Progressive Delivery:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. Rolling Update (Deployment Bawaan K8s)                                   │
│    Pod v1: [●] [●] [●] ──> Ganti perlahan ──> Pod v2: [★] [★] [★]           │
│    Kelemahan: Tidak bisa mengatur persentase trafik secara presisi          │
│    (Trafik terbagi berdasarkan rasio jumlah pod, bukan bobot jaringan).     │
│    Jika pod v2 lulus Readiness Probe tapi error saat query DB, 100% user kena.│
├─────────────────────────────────────────────────────────────────────────────┤
│ 2. Blue-Green Deployment                                                    │
│    Versi Blue (Aktif):  [Pod v1] [Pod v1] <── [Service K8s Aktif] <── Trafik│
│    Versi Green (Uji):   [Pod v2] [Pod v2] <── [Service Preview]             │
│    Keunggulan: Pengalihan instan (0 downtime), rollback instan via Service. │
│    Kelemahan: Membutuhkan kapasitas CPU/RAM 2x lipat (200% headroom).       │
├─────────────────────────────────────────────────────────────────────────────┤
│ 3. Canary Deployment (Argo Rollouts)                                        │
│    Trafik Masuk ──> [Ingress / Service Router]                              │
│                          ├─ 95% Trafik ──> [Pod Stable v1]                  │
│                          └─  5% Trafik ──> [Pod Canary v2]                  │
│                                                   │                         │
│    Prometheus memantau Error Rate pod v2 ─────────┘                         │
│    - Jika Error Rate < 0.1% ──> Naikkan ke 20% ──> 50% ──> 100%             │
│    - Jika Error Rate > 1.0% ──> OTOMATIS ABORT & BALIK KE 0%!               │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Parameter Evaluasi | Rolling Update (Bawaan) | Blue-Green Deployment | Canary Deployment (Argo Rollouts) |
|---|---|---|---|
| **Alat Pengendali** | Kubernetes `Deployment` | `Rollout` / Custom Service | `Rollout` CRD + Ingress/Service Mesh |
| **Kontrol Trafik** | Berdasarkan rasio Pod | Semua-atau-tidak sama sekali (100% cutover) | Granular (misal: 2%, 5%, 20%, 50%, 100%) |
| **Kebutuhan Resource**| Sangat hemat (+25% surge) | Boros (butuh 200% resource) | Efisien (hanya butuh sedikit surge pod) |
| **Evaluasi Metrik** | Hanya `readinessProbe` (UP/DOWN)| Manual / Smoke Test eksternal | Otomatis via Prometheus/Mimir query |
| **Waktu Rollback** | Lambat (perlu spin down pod baru & spin up pod lama) | Sangat Cepat (< 1 detik, ubah selector Service)| Sangat Cepat (otomatis alihkan trafik kembali ke 0%) |
| **Dampak Insiden** | Seluruh pengguna berpotensi terdampak bug | Semua pengguna terdampak jika switch dilakukan tanpa uji matang | Sangat terisolasi (hanya 2–5% pengguna yang terkena dampak) |

---

## 2. Arsitektur Komponen Argo Rollouts

Argo Rollouts adalah Custom Controller Kubernetes yang menggantikan resource standar `Deployment` dengan resource cerdas bernama `Rollout`.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ ARSITEKTUR RUNTIME ARGO ROLLOUTS DI KUBERNETES                              │
│                                                                             │
│   [ Argo Rollouts Controller ]                                              │
│        │                                                                    │
│        ├─ Mengelola ──> [ Rollout Resource ] (Replika Pod Stable & Canary) │
│        │                                                                    │
│        ├─ Menjalankan ──> [ AnalysisRun ] (Background Job Pengecekan Metrik)│
│        │                        │                                           │
│        │                        ▼ (Query PromQL setiap 30 detik)            │
│        │                  [ Prometheus / Mimir Server ]                     │
│        │                  "Berapa HTTP 5xx error rate pod canary?"          │
│        │                                                                    │
│        └─ Mengatur ──> [ Service Router / Ingress ]                         │
│                        ├─ Stable Service ──> Selector: `rollouts-pod-hash=A`│
│                        └─ Canary Service ──> Selector: `rollouts-pod-hash=B`│
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 Tiga Custom Resource Definition (CRD) Inti
1. **`Rollout`:** Pengganti `kind: Deployment`. Mendefinisikan template Pod, jumlah replika, dan strategi rilis (canary/blue-green).
2. **`AnalysisTemplate`:** Cetak biru (*blueprint*) pengujian mutu. Mendefinisikan query metrik apa yang harus dijalankan ke sistem observabilitas (misal: Prometheus), interval pengecekan, serta ambang batas kelulusan (*success criteria*).
3. **`AnalysisRun`:** Instansiasi nyata dari `AnalysisTemplate` yang dieksekusi secara otomatis oleh controller saat rollout berlangsung. Berstatus `Running`, `Successful`, atau `Failed`.

---

## 3. Implementasi Manifes: Canary dengan Evaluasi Otomatis Prometheus

Berikut adalah konfigurasi lengkap tingkat produksi (*production-grade*) yang siap diterapkan di cluster k3s on-prem.

### 3.1 Manifes Rollout (`rollout.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-api
  namespace: production
  labels:
    app: payment-api
spec:
  replicas: 10
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
        - name: payment-api
          image: registry.example.com/apps/payment-api@sha256:7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
  strategy:
    canary:
      # Referensi ke dua service terpisah
      stableService: payment-api-stable
      canaryService: payment-api-canary
      # Analisis otomatis yang berjalan di latar belakang selama rilis
      analysis:
        templates:
          - templateName: http-error-rate-check
        args:
          - name: service-name
            value: payment-api-canary
      # Tahapan peningkatan trafik bertahap (Traffic Stepping)
      steps:
        # Langkah 1: Kirim 5% trafik ke pod baru
        - setWeight: 5
        # Langkah 2: Berhenti sejenak selama 10 menit untuk observasi metrik
        - pause: { duration: 10m }
        # Langkah 3: Jika metrik aman, naikkan menjadi 20%
        - setWeight: 20
        - pause: { duration: 15m }
        # Langkah 4: Naikkan ke 50%
        - setWeight: 50
        - pause: { duration: 15m }
        # Langkah 5: Promosi penuh ke 100%
```

---

### 3.2 Manifes Evaluasi Metrik (`analysis-template.yaml`)

Manifes ini menginstruksikan controller untuk menanyai Prometheus: *"Apakah pod canary menghasilkan HTTP 500 lebih dari 1%?"*. Jika ya, rilis langsung digagalkan!

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: http-error-rate-check
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: error-rate
      interval: 30s
      # Evaluasi metrik selama minimal 3 kali sebelum memutuskan
      count: 5
      # Berikan toleransi maksimal 1 kali gagal sebelum membatalkan rollout
      failureLimit: 1
      # Kriteria keberhasilan: Error rate 5xx harus di bawah 1% (0.01)
      successCondition: result[0] < 0.01
      provider:
        prometheus:
          # URL internal Prometheus di cluster k3s on-prem
          address: http://prometheus-k8s.monitoring.svc.cluster.local:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status=~"5.*"}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

---

## 4. Prosedur Rollback & Peringatan Kritis SRE (The Hard Truths)

Ketika metrik menunjukkan degradasi performa atau aplikasi canary mengalami crash, SRE harus memahami mekanisme rollback serta konsekuensinya terhadap sistem penyimpanan data.

### 4.1 Prosedur Pembatalan Cepat (*Fast-Track Abort*)

Jika terjadi keadaan darurat, jalankan perintah CLI berikut untuk mengembalikan keadaan sistem secara instan:

```bash
# 1. Batalkan (Abort) proses canary seketika
# Perintah ini langsung mengembalikan seluruh bobot trafik ke Stable Service (0% ke canary)
kubectl argo rollouts abort payment-api -n production

# 2. Kembalikan spesifikasi Rollout ke versi sebelumnya yang stabil
kubectl argo rollouts undo payment-api -n production

# 3. Verifikasi status bahwa versi stabil telah aktif penuh
kubectl argo rollouts status payment-api -n production
```

---

### 4.2 Tiga Peringatan Kritis SRE Mengenai Rollback

Banyak pemula berasumsi bahwa *"Rollback di GitOps akan membatalkan semua masalah"*. Dalam realitas produksi SRE, terdapat tiga jebakan besar yang tidak bisa diselesaikan hanya dengan `git revert` atau `rollout undo`:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ REALITAS OPERASIONAL SRE: BATASAN TEKNIS SEBUAH ROLLBACK                    │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Database Schema Migration TIDAK Pernah Ter-Rollback Otomatis!            │
│    Jika pod canary versi baru menjalankan `ALTER TABLE DROP COLUMN`, pod    │
│    lama (v1) akan langsung rusak seketika karena kolom yang dicarinya hilang!│
│    Aturan SRE: Selalu gunakan pola "Expand-Contract Pattern" (Parallel-Run).│
│    Jangan pernah menghapus kolom di versi rilis yang sama dengan migrasi!   │
├─────────────────────────────────────────────────────────────────────────────┤
│ 2. Data Kerusakan di PersistentVolume (PVC) Bersifat Permanen               │
│    Jika aplikasi versi baru menulis format file JSON yang rusak ke storage  │
│    bersama (NFS/Longhorn), rollback kode aplikasi tidak memperbaiki file    │
│    yang sudah terlanjur korup di hard disk fisik.                           │
├─────────────────────────────────────────────────────────────────────────────┤
│ 3. Efek Samping ke Sistem Eksternal (External Side Effects)                 │
│    Jika versi canary sempat mengirim 50 email notifikasi keliru ke nasabah  │
│    atau memicu webhook potong saldo di payment gateway pihak ketiga,        │
│    rollback aplikasi tidak akan membatalkan email atau saldo yang terpotong.│
│    Dibutuhkan *Compensating Transaction* (tindakan pemulihan bisnis manual).│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Failure Modes & Troubleshooting Guide

Berikut adalah matriks diagnosa cepat saat mengoperasikan Argo Rollouts di cluster on-prem:

| Gejala Masalah | Kemungkinan Penyebab Utama | Langkah Investigasi & Solusi SRE |
|---|---|---|
| **Canary Tidak Menerima Trafik** | - Selector Service salah.<br>- Ingress controller tidak mendukung traffic split.<br>- Pod canary belum berstatus `Ready`. | 1. Cek endpoint: `kubectl get ep payment-api-canary`<br>2. Pastikan `readinessProbe` lolos.<br>3. Cek apakah ingress controller (Traefik/NGINX) telah diaktifkan fitur weight-nya. |
| **Analysis `NoData` Error** | - Prometheus server down.<br>- Label query PromQL tidak cocok dengan service.<br>- Tidak ada trafik masuk ke canary (pembagian nol). | 1. Uji query PromQL langsung di web browser Prometheus UI.<br>2. Pastikan pod Prometheus sehat.<br>3. Konfigurasi `missingData: ignore` pada template jika trafik awal sangat rendah. |
| **Node Cluster Kehabisan RAM/CPU** | - Pola Blue-Green menggandakan jumlah pod melampaui kapasitas mesin fisik Mac/Server.<br>- OOMKilled (Exit Code 137). | 1. Cek beban node: `kubectl top nodes`<br>2. Kurangi jumlah replika atau turunkan nilai `resources.requests` di manifes.<br>3. Gunakan Canary bertahap alih-alih Blue-Green murni pada cluster lab kecil. |
| **Rollout Terjebak di Status `Paused`** | - Langkah `pause: {}` tanpa durasi menunggu persetujuan manual manusia.<br>- Step analysis gagal. | 1. Cek status: `kubectl argo rollouts get rollout payment-api`<br>2. Lakukan promote manual jika sudah disetujui: `kubectl argo rollouts promote payment-api`. |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. MEMANTAU STATUS PROGRESIF DENGAN PLUGIN KUBECTL ARGO ROLLOUTS
# ==============================================================================

# Menampilkan tampilan visual interaktif (TUI) progres rilis canary
kubectl argo rollouts get rollout payment-api -n production --watch

# Memeriksa detail riwayat revisi dan status analysis run
kubectl argo rollouts status rollout payment-api -n production

# ==============================================================================
# 2. OPERASI PENGENDALIAN MANUAL (LIFECYCLE MANAGEMENT)
# ==============================================================================

# Melanjutkan langkah rilis yang sedang di-pause manual
kubectl argo rollouts promote payment-api -n production

# Memaksa promosi langsung ke 100% (melewati sisa langkah canary)
kubectl argo rollouts promote payment-api -n production --full

# Menjeda rilis secara manual untuk investigasi
kubectl argo rollouts pause payment-api -n production

# Melanjutkan kembali rilis yang dijeda
kubectl argo rollouts resume payment-api -n production

# Membatalkan rilis dan mengembalikan trafik ke versi stabil
kubectl argo rollouts abort payment-api -n production

# Mengulang analisis metrik yang gagal setelah perbaikan
kubectl argo rollouts retry rollout payment-api -n production
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa strategi **Canary Deployment** jauh lebih aman diterapkan di lingkungan produksi dibandingkan dengan strategi **Rolling Update** bawaan Kubernetes?
- A. Karena Canary Deployment tidak membutuhkan container runtime seperti containerd.
- B. Karena Canary memungkinkan pengalihan persentase trafik secara bertahap (misal 5%) dan dapat membatalkan rilis secara otomatis jika metrik error rate melebihi ambang batas sebelum seluruh pengguna terkena dampak.
- C. Karena Canary Deployment secara otomatis memperbaiki bug logika di dalam kode aplikasi Java/Python.
- D. Karena Rolling Update membutuhkan downtime server minimal 15 menit setiap kali deploy.

#### Soal 2
Sebuah `AnalysisTemplate` pada Argo Rollouts dikonfigurasi untuk memeriksa error rate setiap 30 detik. Apa yang akan terjadi jika nilai hasil query Prometheus menunjukkan error rate sebesar 2.5% sementara syarat kelulusan (*successCondition*) adalah `result[0] < 0.01` (di bawah 1%)?
- A. Argo Rollouts akan mengabaikan hasil tersebut dan langsung mempromosikan ke 100%.
- B. Cluster k3s akan langsung di-restart secara otomatis.
- C. Status `AnalysisRun` akan menjadi `Failed`, controller menandai Rollout sebagai `Degraded`, dan proses rilis langsung di-*abort* serta mengembalikan seluruh trafik ke versi stabil.
- D. Seluruh database di cluster akan otomatis di-rollback ke snapshot kemarin malam.

#### Soal 3
Seorang junior engineer mengeksekusi `kubectl argo rollouts abort` setelah rilis versi v2.0 mengalami kegagalan. Namun, pengguna masih mengeluh bahwa aplikasi versi v1.0 menampilkan pesan error "Database Column Not Found". Mengapa hal ini bisa terjadi?
- A. Karena perintah abort gagal berjalan di cluster Kubernetes.
- B. Karena rollback kode aplikasi tidak secara otomatis membatalkan (*rollback*) migrasi skema database yang sudah terlanjur dieksekusi di server basis data.
- C. Karena cache browser seluruh pengguna internet harus dibersihkan secara manual oleh SRE.
- D. Karena ArgoCD menolak melakukan rekonsiliasi jika ada pod yang mati.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Tim Anda sedang merilis versi baru dari microservice e-commerce menggunakan pola Canary Deployment (5% -> 20% -> 50% -> 100%). Pada langkah ke-2 (20% traffic weight), query metrik pada `AnalysisTemplate` menghasilkan status `NoData` (tidak ada metrik yang kembali dari server observabilitas) karena pod Prometheus kehabisan memori (*OOMKilled*).

Secara *default*, jika tidak dikonfigurasi dengan hati-hati, ada sistem yang memperlakukan `NoData` sebagai "sukses" karena tidak ada error 500 yang tercatat.

**Pertanyaan & Tugas SRE:**
1. Mengapa menganggap ketiadaan metrik (*NoData*) sebagai kondisi aman (*pass*) merupakan anti-pattern berbahaya dalam prinsip SRE?
2. Bagaimana cara mengonfigurasi perilaku *missing-data* pada `AnalysisTemplate` agar sistem bersikap defensif (*fail-safe*)?
3. Sebutkan tindakan operasional yang harus diambil oleh On-Call SRE saat insiden ini terjadi!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Rolling update hanya membagi risiko berdasarkan rasio jumlah pod dan tidak mengintegrasikan metrik bisnis/telemetri ke dalam alur rilis. Canary deployment membatasi paparan risiko pada segmen trafik kecil dan terintegrasi dengan analisis metrik otomatis.
2. **C.** Sesuai kontrak `AnalysisTemplate`, ketika kriteria keberhasilan dilanggar melampaui `failureLimit`, controller secara otonom membatalkan promosi (*abort*) dan memulihkan trafik ke versi stabil tanpa menunggu tindakan manual manusia.
3. **B.** Rollback komputasi (Pod/Deployment) terisolasi dari lapisan persistensi (Database). Jika migrasi database bersifat *destructive* (merusak backward compatibility), rollback aplikasi lama akan tetap gagal membaca skema baru.

#### Solusi Soal Kasus Nyata:
1. **Bahaya Menganggap NoData sebagai Sukses:**
   - Ketiadaan metrik tidak sama dengan ketiadaan error (*absence of evidence is not evidence of absence*).
   - Jika sistem monitoring mati, kita berada dalam kondisi "buta" (*flying blind*). Aplikasi baru bisa saja sedang crash total atau mencuri data nasabah tanpa kita sadari.
2. **Konfigurasi Defensif (Fail-Safe Strategy):**
   - Pada `AnalysisTemplate`, atur parameter toleransi error jaringan/monitoring secara ketat.
   - Jangan izinkan promosi berlanjut jika `consecutiveErrorLimit` tercapai. Default aman adalah membuat status analisis menjadi `Inconclusive` atau `Paused`, sehingga rilis tertahan dan membutuhkan persetujuan/verifikasi manual dari manusia sebelum melanjutkan.
3. **Tindakan Operasional On-Call SRE:**
   - **Tahan Rilis:** Jalankan `kubectl argo rollouts pause` agar rilis tidak naik ke tahap 50%.
   - **Pulihkan Observabilitas:** Investigasi pod Prometheus (`kubectl describe pod -n monitoring`), naikkan memory limit jika terjadi OOMKilled, dan pastikan scrape target kembali UP.
   - **Verifikasi Data Ulang:** Jalankan `kubectl argo rollouts retry` untuk mengeksekusi ulang `AnalysisRun` setelah Prometheus pulih. Jika metrik valid dan error rate di bawah 1%, barulah promosi dilanjutkan.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Sekarang Anda telah menguasai alur pengiriman kode tingkat lanjut dengan jaminan observabilitas dan mitigasi risiko.
- Lanjutkan ke [LAB-02 — Canary dan Blue-Green Hands-On Lab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.3-end-to-end-flow/lab/LAB-02-canary-blue-green-introduction.md) untuk mempraktikkan langsung deployment Argo Rollouts pada cluster k3s lokal Anda.
- Sambungkan metrik analisis ini secara mendalam di [Fase 7 Observability](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/README.md) dengan Prometheus, Grafana Alloy, Mimir, dan Loki.
