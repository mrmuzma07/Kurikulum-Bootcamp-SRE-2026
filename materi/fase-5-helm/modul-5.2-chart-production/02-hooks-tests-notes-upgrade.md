# 02 — Hooks, Tests, NOTES, dan Siklus Upgrade: Mengotomasi Titik Transisi Rilis Tanpa Operasi Liar

> **Ringkasan:** Dalam siklus hidup aplikasi produksi, proses pembaruan jarang sekali sesederhana "ganti gambar kontainer lalu restart". Sering kali ada serangkaian operasi pra-syarat yang wajib tuntas terlebih dahulu—seperti migrasi skema basis data (*database migration*), pembersihan cache Redis, atau verifikasi dependensi pihak ketiga. **Helm Hooks** memungkinkan SRE mengintervensi titik-titik transisi rilis secara terstruktur dan aman. Dipadukan dengan **Helm Test** (pengujian kesehatan otomatis pasca-rilis) dan **NOTES.txt** (panduan operasional kontekstual), Helm menjadi platform orkestrasi rilis yang tangguh. Dokumen ini membedah arsitektur hook, strategi penghapusan pod sementara, implementasi *smoke-test*, serta mitigasi kegagalan migrasi.

---

## 1. Analogi Pemula: Peluncuran Misi Roket Luar Angkasa

Untuk memahami bagaimana Helm Hooks mengatur urutan rilis, mari gunakan analogi peluncuran roket misi luar angkasa:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PELUNCURAN ROKET vs SIKLUS HIDUP HELM HOOKS         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TAHAP 1: PRE-LAUNCH COUNTDOWN -> PRE-INSTALL / PRE-UPGRADE HOOK ]        │
│  - Sebelum mesin roket utama dinyalakan, teknisi menara kontrol menjalankan │
│    pengecekan otomatis:                                                     │
│    "Apakah tekanan tangki bahan bakar siap? Apakah cuaca mendukung?"        │
│  - Di Helm: Jalankan Job migrasi skema database (**Pre-Upgrade Hook**)!      │
│    JIKA MIGRASI GAGAL: Peluncuran roket BATAL TOTAL (*ABORT*)!              │
│    Mesin utama TIDAK AKAN PERNAH dinyalakan, mencegah aplikasi rusak!       │
│                                                                             │
│  [ TAHAP 2: ROKET MELUNCUR & KAPSUL DI ORBIT -> MAIN RELEASE MANIFESTS ]    │
│  - Mesin utama menyala: Deployment Pod aplikasi baru mulai dibuat.          │
│  - Kapsul pendorong yang sudah habis dibuang ke laut                        │
│    (**Hook Deletion Policy: hook-succeeded**).                              │
│                                                                             │
│  [ TAHAP 3: UJI SISTEM KOMUNIKASI RADIO -> HELM TEST ]                      │
│  - Setelah astronot berada di orbit stabil, menara mengirim sinyal uji:     │
│    "Astronot, apakah Anda mendengar suara kami? (Cek HTTP /healthz)".       │
│  - Jika astronot menjawab "Jelas!" -> Misi dinyatakan SUKSES RESMI!         │
│                                                                             │
│  [ TAHAP 4: BUKU PANDUAN NAVIGASI HARIAN -> NOTES.TXT ]                     │
│  - Menara membacakan instruksi pendaratan: "Untuk menghubungi stasiun bumi, │
│    arahkan antena ke koordinat X (**Alamat Ingress Publik**)".              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Arsitektur Helm Hooks: Titik Intervensi Siklus Hidup

Secara default, saat Anda menjalankan `helm install` atau `helm upgrade`, Helm mengirimkan seluruh manifest di folder `templates/` ke Kubernetes API secara bersamaan.

Dengan menyuntikkan anotasi khusus (`"helm.sh/hook"`), Anda memerintahkan Helm untuk memisahkan manifest tersebut dari alur reguler dan mengeksekusinya pada titik transisi tertentu:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       URUTAN EKSEKUSI SIKLUS HIDUP HOOK                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Operator / CI mengeksekusi: `helm upgrade <release> ...`                │
│            │                                                                │
│            ▼                                                                │
│  2. Eksekusi PRE-UPGRADE HOOKS (Urut berdasarkan hook-weight terendah)      │
│     Contoh: Job Migrasi Database (Weight: -5) -> Job Clear Cache (Weight: 0)│
│            │                                                                │
│            ▼ Apakah seluruh Pre-Upgrade Hook SUKSES?                        │
│     [ YA ]                                    [ TIDAK (Exit Code != 0) ]    │
│       │                                                    │                │
│       │                                                    ▼                │
│       │                                        RILIS DIHENTIKAN SEKETIKA!   │
│       │                                        (Rollback / Failed State)    │
│       ▼                                                                     │
│  3. Eksekusi Manifest Reguler (Deployment, Service, Ingress baru di-apply)  │
│            │                                                                │
│            ▼                                                                │
│  4. Eksekusi POST-UPGRADE HOOKS                                             │
│     Contoh: Mengirim notifikasi webhook rilis sukses ke Slack SRE           │
│            │                                                                │
│            ▼                                                                │
│  5. Pembersihan Objek Hook sesuai `hook-delete-policy`                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tabel 6 Titik Intervensi Hook Utama

| Nama Anotasi Hook | Kapan Dieksekusi? | Contoh Penggunaan SRE |
|---|---|---|
| `pre-install` | Setelah template dirender, tetapi *sebelum* ada resource reguler dibuat di k8s. | Membuat secret enkripsi eksternal, validasi dependensi. |
| `post-install` | Setelah seluruh resource reguler berhasil dibuat di klaster. | Inisialisasi data admin awal, seeding basis data lab. |
| `pre-upgrade` | Setelah template dirender, tetapi *sebelum* resource k8s diperbarui. | **Migrasi skema database (ALTER TABLE)**, snapshot data. |
| `post-upgrade`| Setelah seluruh resource berhasil di-upgrade ke versi baru. | Membersihkan cache Redis, kirim webhook notifikasi. |
| `pre-delete` | Sebelum resource dihapus saat `helm uninstall`. | Backup data darurat sebelum objek dimusnahkan. |
| `test` | HANYA dieksekusi saat operator menjalankan perintah `helm test`. | Automated smoke test, validasi endpoint HTTP. |

---

## 3. Tiga Parameter Kunci Pengendali Hook

Untuk mengontrol perilaku hook secara presisi, gunakan 3 anotasi wajib ini:

### 3.1 Penentuan Titik Hook (`helm.sh/hook`)
Mendefinisikan di fase mana objek ini aktif. Anda dapat menggabungkan beberapa event dengan tanda koma:
```yaml
annotations:
  "helm.sh/hook": "pre-install,pre-upgrade"
```

### 3.2 Bobot Urutan Eksekusi (`helm.sh/hook-weight`)
Jika Anda memiliki 3 Job yang sama-sama berjalan di fase `pre-upgrade`, Anda dapat mengatur urutan eksekusinya menggunakan angka bobot (*integer* positif atau negatif). **Angka paling kecil dieksekusi lebih dahulu**:
```yaml
annotations:
  "helm.sh/hook": "pre-upgrade"
  "helm.sh/hook-weight": "-10" # Dijalankan pertama (Backup)
  # Hook lain dengan weight "0" dijalankan kedua (Migrasi)
  # Hook lain dengan weight "10" dijalankan ketiga (Clear cache)
```

### 3.3 Kebijakan Penghapusan Otomatis (`helm.sh/hook-delete-policy`)
Secara default, Pod/Job hook yang selesai akan tertinggal di klaster selamanya, mengotori output `kubectl get pods`. Kendalikan siklus hidupnya dengan kebijakan penghapusan:
- `before-hook-creation` : Hapus objek hook lama yang bernama sama sebelum membuat objek baru (Mencegah error *Job already exists*!).
- `hook-succeeded` : Hapus objek hook secara otomatis HANYA jika Job selesai dengan sukses (exit code 0).
- `hook-failed` : Hapus objek hook jika gagal (Jarang dipakai di SRE karena kita butuh memeriksa log error!).

```yaml
annotations:
  # Standar Emas SRE: Bersihkan yang lama sebelum jalan, dan hapus jika sukses!
  "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
```

---

## 4. Implementasi Nyata: Database Migration Job via Pre-Upgrade Hook

Berikut adalah berkas manifest produksi untuk Job migrasi database (`templates/db-migration-job.yaml`):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "internal-app.fullname" . }}-db-migrate
  labels:
    {{- include "internal-app.labels" . | nindent 4 }}
  annotations:
    # Aktif pada saat install pertama kali maupun upgrade berkala
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  # Batas toleransi percobaan jika skrip migrasi gagal (Maksimal 2x coba)
  backoffLimit: 2
  # Timeout maksimal jika migrasi database hang (Maksimal 5 menit)
  activeDeadlineSeconds: 300
  template:
    metadata:
      labels:
        app.kubernetes.io/component: db-migration
    spec:
      restartPolicy: Never
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
      containers:
        - name: db-migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          command:
            - "/app/bin/migrate"
            - "up"
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.secretName }}
                  key: database-url
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

> [!IMPORTANT]
> **Aturan Emas SRE: Desain Migrasi Database yang Kompatibel (Expand-Contract):**
> Jika Anda menjalankan `pre-upgrade` hook untuk migrasi database, pastikan perubahan skema database bersifat **Backward-Compatible**! Artinya, skema baru harus tetap bisa dibaca oleh Pod versi lama. Mengapa? Karena saat migrasi selesai, Pod versi lama masih melayani trafik pengguna selama proses *rolling update* berlangsung. Jika migrasi menghapus kolom tabel secara mendadak, Pod lama akan langsung *crash* sebelum Pod baru selesai dibuat!

---

## 5. Pengujian Otomatis Pasca-Rilis (`helm test`)

Setelah aplikasi selesai dideploy, SRE tidak boleh berasumsi bahwa sistem sehat hanya karena statusnya `deployed`. Kita menguji fungsionalitasnya menggunakan **Helm Test**.

### 5.1 Manifest Smoke Test Pod (`templates/tests/test-connection.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "internal-app.fullname" . }}-test-connection"
  labels:
    {{- include "internal-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: wget-test
      image: curlimages/curl:8.5.0
      command:
        - "curl"
        - "--fail"
        - "--connect-timeout"
        - "5"
        - "--max-time"
        - "10"
        - "http://{{ include "internal-app.fullname" . }}:{{ .Values.service.port }}/ready"
```

### 5.2 Mengeksekusi Uji Rilis di Terminal / CI Pipeline
```bash
# Menjalankan pengujian dan mencetak log hasil pengujian ke konsol
helm test payment --namespace production --logs
```
*Output jika Sukses:*
```text
RUNNING: payment-internal-app-test-connection
PASSED: payment-internal-app-test-connection
TEST SUITE: 1 passed, 0 failed
```

---

## 6. Menyusun Catatan Operasional yang Aman (`templates/NOTES.txt`)

File `templates/NOTES.txt` adalah teks informasi yang otomatis tercetak di konsol operator setelah perintah `helm install` atau `helm upgrade` selesai. Gunakan berkas ini untuk memberikan petunjuk verifikasi yang berguna:

```gotemplate
===============================================================================
  APLIKASI: {{ .Chart.Name }} (Versi: {{ .Chart.AppVersion }})
  STATUS  : RILIS {{ .Release.Name }} SUKSES DIPASANG PADA NAMESPACE {{ .Release.Namespace }}
===============================================================================

1. Dapatkan Status Kesiapan Pod Aplikasi:
   $ kubectl get pods -n {{ .Release.Namespace }} -l "app.kubernetes.io/instance={{ .Release.Name }}"

2. Alamat Akses Layanan:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
   URL Publik: https://{{ $host.host }}
{{- end }}
{{- else }}
   Akses Internal Cluster:
   $ kubectl run test-curl --rm -it --image=curlimages/curl -- curl http://{{ include "internal-app.fullname" . }}.{{ .Release.Namespace }}.svc.cluster.local:{{ .Values.service.port }}/healthz
{{- end }}

3. Jalankan Pengujian Smoke-Test Otomatis:
   $ helm test {{ .Release.Name }} -n {{ .Release.Namespace }}

CATATAN SRE:
- Pantau metrik latensi dan error rate rilis ini pada Dashboard Grafana.
- Jika terjadi anomali kritis, lakukan rollback:
  $ helm rollback {{ .Release.Name }} <revision-sebelumnya> -n {{ .Release.Namespace }}
===============================================================================
```

> [!CAUTION]
> **Strict Secrets Hygiene pada NOTES.txt:**
> **DILARANG KERAS** mencetak password, string koneksi database, atau token API di dalam `NOTES.txt`! Setiap teks yang ada di berkas ini akan dicetak ke konsol dan tersimpan di log pipeline CI/CD yang dapat dibaca oleh banyak orang.

---

## 7. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN HOOKS & TESTS                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "UPGRADE FAILED: JOBS.BATCH ALREADY EXISTS" ]                   │
│  Gejala: `helm upgrade` gagal: "rendered manifests contain a resource that  │
│          already exists: jobs.batch/payment-db-migrate".                    │
│  Akar Masalah:                                                              │
│  - Job migrasi dari rilis sebelumnya masih tertinggal di klaster, dan       │
│    anotasi `hook-delete-policy` tidak menyertakan `before-hook-creation`.   │
│  Solusi SRE:                                                                │
│  1. Tambahkan anotasi pada Job template:                                    │
│     "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"     │
│  2. Hapus Job lama yang menggantung secara manual:                          │
│     kubectl delete job payment-db-migrate -n production                     │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: HOOK JOB HANG KARENA DATABASE DOWN ]                            │
│  Gejala: Helm upgrade menunggu hingga 5 menit lalu melempar timeout error.  │
│  Akar Masalah:                                                              │
│  - Job migrasi mencoba menghubungi database Postgres, tetapi database       │
│    sedang down atau kredensial password salah, sehingga Job terus me-retry. │
│  Solusi SRE:                                                                │
│  1. Selalu batasi durasi Job hook dengan `activeDeadlineSeconds: 120`.      │
│  2. Inspeksi log Job yang gagal:                                            │
│     kubectl logs -n production job/payment-db-migrate                       │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: HELM TEST FAILED: "CONNECTION REFUSED" ]                        │
│  Gejala: `helm test` gagal padahal Pod aplikasi dilaporkan Running.         │
│  Akar Masalah:                                                              │
│  - Pod aplikasi membutuhkan waktu 20 detik untuk inisialisasi framework,    │
│    tetapi test Pod langsung dijalankan seketika sebelum port siap melayani. │
│  Solusi SRE:                                                                │
│  1. Pastikan *Readiness Probe* pada Pod aplikasi sudah dikonfigurasi benar. │
│  2. Di dalam test script, gunakan curl dengan opsi retry:                   │
│     curl --retry 5 --retry-delay 3 http://.../ready                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Menjalankan seluruh test suite rilis yang terdaftar di templates/tests/
helm test payment --namespace production

# 2. Menjalankan test dan langsung mencetak log output Pod test ke konsol
helm test payment --namespace production --logs

# 3. Menghapus Pod test yang tertinggal setelah pengujian selesai
kubectl delete pod -n production -l "helm.sh/hook=test"

# 4. Menampilkan pesan panduan operasional (NOTES.txt) dari rilis yang aktif
helm get notes payment --namespace production

# 5. Menguji rendering template hook dan memeriksa bobot weight secara lokal
helm template payment ./internal-app -s templates/db-migration-job.yaml
```

---

## 9. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Sebuah Job migrasi database memiliki anotasi `"helm.sh/hook": "pre-upgrade"`. Jika Job tersebut gagal mengeksekusi migrasi skema (melempar exit code 1), apa yang akan dilakukan oleh Helm?
- A. Helm otomatis mengabaikan kegagalan tersebut dan melanjutkan pembuatan Pod aplikasi baru.
- B. Helm akan langsung membatalkan proses rilis (*aborted*), dan tidak akan pernah menerapkan manifest Deployment versi baru ke dalam klaster Kubernetes.
- C. Helm otomatis menghapus basis data produksi.
- D. Helm mengubah Service menjadi NodePort.

*Kunci Jawaban:* **B**. Pre-upgrade hook bertindak sebagai gerbang pengaman mutlak; kegagalan pada tahap ini menggagalkan rilis sebelum beban kerja baru dideploy.

### Soal 2 (Pilihan Ganda)
Mengapa kombinasi anotasi `"helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"` dianggap sebagai praktik terbaik SRE dalam mendefinisikan Job hook?
- A. Karena dapat mempercepat koneksi Wi-Fi data center.
- B. Memastikan objek Job lama yang bernama sama dibersihkan sebelum rilis baru dimulai (mencegah error *already exists*), dan membersihkan objek Job yang sukses secara otomatis agar klaster tetap rapi, sambil tetap mempertahankan Job jika gagal untuk keperluan audit/forensik log.
- C. Agar Job berjalan tanpa menggunakan kuota memori RAM.
- D. Mengharuskan operator mengetik password root setiap kali hook jalan.

*Kunci Jawaban:* **B**. Kebijakan ini mencegah kegagalan tabrakan nama objek pada rilis berikutnya sekaligus menyisakan jejak forensik jika terjadi kegagalan (*failed state preservation*).

### Soal 3 (Pilihan Ganda)
Kapan manifest yang memiliki anotasi `"helm.sh/hook": "test"` akan dieksekusi oleh Helm di dalam klaster?
- A. Setiap 5 menit sekali secara terjadwal.
- B. HANYA ketika operator atau skrip pipeline CI/CD mengeksekusi perintah `helm test <release-name>`.
- C. Bersamaan saat perintah `helm template` dijalankan di laptop lokal.
- D. Saat node fisik server mengalami reboot.

*Kunci Jawaban:* **B**. Objek bertipe hook `test` tidak dieksekusi saat install atau upgrade biasa, melainkan dipicu secara on-demand via perintah `helm test`.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Di sebuah rilis pembaruan microservice pembayaran, developer menyertakan pre-upgrade hook yang menjalankan perintah SQL:
`ALTER TABLE payment_transactions DROP COLUMN legacy_account_number;`
Saat rilis dijalankan, Pod baru ternyata mengalami bug logika pada modul enkripsi dan SRE terpaksa menjalankan `helm rollback`. Begitu rollback selesai, seluruh Pod versi lama yang aktif seketika tumbang dan mengeluarkan error `Database Column legacy_account_number Not Found`! Analisis akar kegagalan arsitektur ini dan rancang solusi migrasi yang benar!

*Panduan Solusi:*
1. **Analisis Akar Kegagalan Arsitektur:**
   Developer melanggar hukum *Backward-Compatibility* pada skema database. Ketika kolom langsung dihapus di tahap pre-upgrade, basis data tidak lagi kompatibel dengan aplikasi versi lama. Ketika terjadi masalah dan SRE melakukan *rollback*, kode aplikasi versi lama yang bergantung pada kolom tersebut menjadi rusak total (*Catastrophic Downtime*). Ingat: `helm rollback` hanya membalikkan versi manifest Kubernetes, bukan membalikkan struktur tabel database!
2. **Solusi Arsitektur Migrasi yang Benar (Pola Expand-Contract):**
   - **Fase 1 (Expand / Rilis N):**
     Tambahkan kolom baru tanpa menghapus kolom lama. Aplikasi baru mulai menulis ke kedua kolom, tapi tetap bisa membaca kolom lama.
   - **Fase 2 (Migrate Data / Background Job):**
     Salin data historis dari kolom lama ke kolom baru.
   - **Fase 3 (Contract / Rilis N+1):**
     Setelah rilis N terbukti stabil di produksi selama beberapa hari/minggu dan tidak ada lagi kebutuhan rollback, barulah pada rilis N+1 kolom lama dihapus secara aman.

---

## 10. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Custom Chart dan Values per Environment](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/01-custom-chart-environment-values.md) — Memahami pemisahan nilai per lingkungan.
- **Topik Berikutnya:**
  - [03 — Security, Reliability, dan Observability Preparation](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/03-production-chart-security-reliability.md) — Memperkuat postur keamanan PodSecurityContext dan kesiapan telemetri.
  - [LAB-02 — Observability, OCI, Test, dan Rollback](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.2-chart-production/lab/LAB-02-observability-oci-test-rollback.md) — Praktik hands-on pengujian hook dan test di klaster.
