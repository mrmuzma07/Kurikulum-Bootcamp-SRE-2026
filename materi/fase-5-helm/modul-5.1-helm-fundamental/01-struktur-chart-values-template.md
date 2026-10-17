# 01 — Struktur Chart, Values, dan Go-template: Mengemas Aplikasi Kubernetes Menjadi Paket Rilis Deterministik

> **Ringkasan:** Di lingkungan Kubernetes produksi, mengelola puluhan manifest YAML mentah (*Deployment, Service, Ingress, ConfigMap, PDB*) menggunakan `kubectl apply` secara manual adalah anti-pola yang melahirkan inkonsistensi antar-lingkungan (*staging vs production*), ketiadaan riwayat versi rilis, dan kemustahilan melakukan *rollback* yang aman. **Helm** hadir sebagai *Package Manager* standar industri untuk Kubernetes. Dokumen ini membedah arsitektur dasar Helm, anatomi struktur berkas *Chart*, kekuatan mesin *Go-template*, validasi masukan menggunakan *JSON Schema*, serta teknik *rendering* manifest yang deterministik dan aman dari kebocoran rahasia.

---

## 1. Analogi Pemula: Paket Software OS & Percetakan Buku Kustom

Bagi engineer yang baru beralih dari manifest Kubernetes statis ke Helm, pertanyaan mendasar yang sering muncul adalah:
> *"Kenapa saya tidak cukup menyimpan file `deployment.yaml` dan `service.yaml` di Git lalu menjalankan `kubectl apply -f .`? Kenapa harus repot-repot belajar Helm Chart?"*

Mari gunakan analogi percetakan buku dan paket perangkat lunak:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI MANIFEST STATIS vs HELM PACKAGE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PENDEKATAN MANIFEST STATIS: KERTAS FOTOKOPI SATU PER SATU ]              │
│  - Masalah: Anda memiliki 1 aplikasi web microservice. Aplikasi ini butuh:  │
│    1 Deployment, 1 Service, 1 Ingress, 1 ConfigMap, 1 ServiceAccount, 1 PDB.│
│  - Total ada 6 file YAML terpisah.                                          │
│  - Di lingkungan Staging, Anda butuh 1 replika dan RAM 256MB.              │
│  - Di lingkungan Production, Anda butuh 5 replika dan RAM 2GB.              │
│  - Akibat: Anda terpaksa membuat 2 folder terpisah (`staging/` & `prod/`)    │
│    yang isinya 12 file duplikat! Jika developer mengubah port dari 80 ke    │
│    8080, Anda harus mengedit banyak file secara manual. Jika satu saja      │
│    terlewat, sistem akan mati (*Configuration Drift*)!                      │
│                                                                             │
│  [ PENDEKATAN HELM: CETAK BIRU PERCETAKAN BUKU KUSTOM ]                     │
│  - **Chart (Cetak Biru Universal):** Satu paket template yang mendefinisikan│
│    arsitektur aplikasi secara utuh tanpa data spesifik lingkungan.          │
│  - **Values (Form Isian Parameter):** Formulir kecil yang berisi variabel   │
│    khusus (misal: `values-staging.yaml` = 1 replica; `values-prod.yaml`     │
│    = 5 replicas).                                                           │
│  - **Mesin Percetakan (Helm Engine):** Menggabungkan Cetak Biru + Formulir  │
│    menjadi buku utuh, memberikan nomor seri rilis (*Release Revision*),     │
│    dan memasukkannya ke dalam rak (*Namespace Kubernetes*).                 │
│  - Jika rilis baru bermasalah, sang manajer percetakan bisa langsung        │
│    menarik buku tersebut dan mengembalikan edisi sebelumnya (*Rollback*)!   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Taksonomi Inti Helm: Chart, Values, Release, dan Registry

Empat istilah ini adalah fondasi komunikasi tim SRE saat mengoperasikan Helm:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ALUR KERJA KOMPONEN UTAMA HELM                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   [ CHART ]                      [ VALUES ]                                 │
│   (Cetak Biru Template)          (Parameter Variabel)                       │
│   - Chart.yaml                   - values.yaml (Default aman)               │
│   - templates/*.yaml             - values-prod.yaml (Override)              │
│            │                              │                                 │
│            └──────────────┬───────────────┘                                 │
│                           │                                                 │
│                           ▼ Eksekusi: `helm template` / `helm install`      │
│                ┌────────────────────┐                                       │
│                │    HELM ENGINE     │                                       │
│                │ (Render Go-Template│                                       │
│                │   + Schema Check)  │                                       │
│                └──────────┬─────────┘                                       │
│                           │                                                 │
│                           ▼ Output: Manifest Kubernetes Utuh (Plain YAML)   │
│                ┌────────────────────┐                                       │
│                │   KUBERNETES API   │                                       │
│                │  (Kube-APIServer)  │                                       │
│                └──────────┬─────────┘                                       │
│                           │                                                 │
│                           ▼ Tercipta Objek Nyata di Cluster                 │
│                ┌────────────────────┐                                       │
│                │      RELEASE       │                                       │
│                │  (Instance Aktif   │                                       │
│                │  Rev 1, Namespace) │                                       │
│                └────────────────────┘                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Komponen Helm | Definisi SRE | Lokasi Penyimpanan |
|---|---|---|
| **Chart** | Arsip berkas atau direktori berisi deskripsi paket dan sekumpulan template deklaratif Kubernetes. | Git Repositori / OCI Registry (GitLab/Harbor). |
| **Values** | Berkas masukan parameter konfigurasi yang menyuntikkan nilai dinamis ke dalam template Chart. | Git Repositori (terpisah per lingkungan). |
| **Release** | Instansiasi berjalan dari sebuah Chart yang dideploy ke dalam *namespace* Kubernetes tertentu dengan nomor revisi pelacakan. | Disimpan sebagai Secret internal Kubernetes (`sh.helm.release.v1.*`). |
| **Chart Repository / OCI** | Layanan penyimpan dan pendistribusi paket Chart yang telah dikompresi (`.tgz`). | GitLab Package Registry / OCI compliant registries. |

> [!IMPORTANT]
> **Peringatan SRE: Perbedaan Client-Side Render vs Runtime Proof:**
> Perintah `helm template` bekerja sepenuhnya di sisi klien (*client-side*). Keberhasilan rendering manifest **BUKAN BUKTI** bahwa aplikasi akan berjalan sukses di klaster! `helm template` tidak memeriksa apakah *image container* benar-benar ada di registry, apakah CPU/RAM node mencukupi, apakah Admission Webhook mengizinkan Pod, atau apakah Service dapat merespons lalu lintas jaringan.

---

## 3. Anatomi Struktur Berkas Chart Standar Produksi

Saat Anda membuat chart kustom (misal: `internal-app`), struktur direktori wajib mengikuti standar baku komunitas Kubernetes:

```text
internal-app/
├── Chart.yaml              # Metadata resmi paket (Nama, SemVer, Deskripsi, Maintainers)
├── values.yaml             # Nilai bawaan (defaults) publik yang aman & non-sensitif
├── values.schema.json      # Skema JSON untuk validasi tipe data dan batasan nilai input
├── .helmignore             # Daftar file lokal yang dilarang ikut terbungkus ke paket .tgz
├── templates/              # Direktori berisi cetak biru manifest Kubernetes (Go-templates)
│   ├── _helpers.tpl        # Snippet template parsial & generator nama/label terstandar
│   ├── deployment.yaml     # Manifest workload Deployment Pod
│   ├── service.yaml        # Manifest abstraksi jaringan Service (ClusterIP/NodePort)
│   ├── ingress.yaml        # Manifest routing lalu lintas HTTP/HTTPS Ingress
│   ├── configmap.yaml      # Manifest konfigurasi non-rahasia aplikasi
│   ├── pdb.yaml            # PodDisruptionBudget untuk perlindungan high availability
│   ├── NOTES.txt           # Panduan instruksi pasca instalasi yang dicetak ke konsol
│   └── tests/              # Direktori smoke test otomatis (helm test)
│       └── test-connection.yaml
├── charts/                 # Sub-charts dependensi (jika aplikasi memuat database internal)
└── crds/                   # Custom Resource Definitions (jika chart membawa ekstensi API k8s)
```

### 3.1 Metadata Resmi Paket: `Chart.yaml`

```yaml
apiVersion: v2
name: internal-app
description: "Helm Chart resmi aplikasi microservice internal Bootcamp SRE 2026"
type: application

# Versi paket Chart itu sendiri (Wajib mengikuti aturan Semantic Versioning: MAJOR.MINOR.PATCH)
version: 1.2.0

# Versi aplikasi perangkat lunak yang dibungkus di dalamnya
appVersion: "2.4.1"

keywords:
  - sre
  - microservice
  - onprem

maintainers:
  - name: "SRE Platform Team"
    email: "sre-core@bootcamp.local"
```

> [!NOTE]
> **Pembedaan `version` vs `appVersion`:**
> - `version` (SemVer Chart): Wajib dinaikkan jika ada perubahan pada template YAML, penambahan port baru, atau modifikasi logika helper, meskipun kode aplikasi developer tidak berubah!
> - `appVersion` (Versi Aplikasi): Informasi versi biner/source code aplikasi developer. Mengubah `appVersion` tidak boleh menggantikan penggunaan *immutable image digest* pada kontainer produksi.

---

## 4. Nilai Bawaan yang Aman (`values.yaml`)

File `values.yaml` berfungsi sebagai dokumentasi hidup (*living documentation*) dan penyedia nilai bawaan yang aman (*safe defaults*):

```yaml
# values.yaml - Nilai Bawaan Aman (Default Fallback)

replicaCount: 2

image:
  repository: registry.internal.corp/bootcamp/payment-service
  pullPolicy: IfNotPresent
  # Tag atau Digest kontainer (Default fallback lab)
  tag: "v2.4.1"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 10001
  fsGroup: 10001

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
    - host: payment.internal.corp
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

probes:
  liveness:
    path: /healthz
    port: 8080
    initialDelaySeconds: 15
  readiness:
    path: /ready
    port: 8080
    initialDelaySeconds: 5

podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

---

## 5. Mesin Go-Template, Helpers, dan Sprig Functions

Helm memproses template menggunakan mesin bahasa Go (*Go text/template*) yang diperkaya oleh pustaka **Sprig Functions**.

### 5.1 Named Templates & Generator Label: `_helpers.tpl`
File `_helpers.tpl` bertugas mencegah duplikasi kode dengan membuat fungsi snippet yang dapat dipanggil di semua template:

```gotemplate
{{/*
Membuat nama rilis yang dipotong maksimal 63 karakter sesuai aturan RFC 1123 DNS Kubernetes.
*/}}
{{- define "internal-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Standar label resmi Kubernetes (Common Labels) untuk audit dan pemantauan
*/}}
{{- define "internal-app.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{ include "internal-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels yang bersifat immutable
*/}}
{{- define "internal-app.selectorLabels" -}}
app.kubernetes.io/name: {{ default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 5.2 Implementasi Template Deployment (`templates/deployment.yaml`)

Perhatikan penggunaan whitespace trimming (`{{-` dan `-}}`) serta fungsi `nindent` untuk menjaga struktur indentasi YAML tetap valid:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "internal-app.fullname" . }}
  labels:
    {{- include "internal-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "internal-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "internal-app.selectorLabels" . | nindent 8 }}
    spec:
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ .Values.probes.liveness.port }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: {{ .Values.probes.readiness.port }}
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 5.3 Enam Aturan Emas Go-Template Helm
1. **Aturan Titik (`.`) vs Dollar (`$`):** Titik merepresentasikan konteks *saat ini* (*current scope*). Ketika Anda masuk ke dalam blok `{{- with .Values.image }}`, tanda titik berubah menjadi objek `image`. Jika ingin mengakses variabel root dari dalam blok `with`, gunakan tanda dollar: `{{ $.Release.Name }}`!
2. **`nindent` Mengalahkan `indent`:** Selalu gunakan `nindent 4` saat mem-parsing objek menggunakan `toYaml`. `nindent` menambahkan baris baru (*new line*) terlebih dahulu sebelum memberikan spasi, menjamin struktur blok YAML rapi.
3. **Fungsi `required`:** Hentikan proses rendering jika ada parameter krusial yang tidak diisi pengguna:
   ```gotemplate
   database_host: {{ required "Nilai .Values.database.host wajib diisi!" .Values.database.host }}
   ```
4. **Fungsi `quote`:** Selalu apit nilai string angka atau boolean dengan `quote` agar tidak disalahpahami oleh parser YAML:
   ```gotemplate
   port_string: {{ .Values.service.port | quote }}
   ```
5. **Whitespace Trimming:** `{{-` menghapus semua karakter spasi dan baris kosong di sebelah kiri ekspresi; `-}}` menghapus di sebelah kanan.
6. **Immutable Selectors:** Nilai pada `spec.selector.matchLabels` **TIDAK BOLEH BERUBAH** sepanjang masa hidup Deployment. Mengubah selector akan memicu error fatal `field is immutable` saat upgrade!

---

## 6. Validasi Skema JSON (`values.schema.json`)

Untuk mencegah kesalahan manusia (*human error*) di mana developer mengisi `replicaCount: "banyak"` (seharusnya angka) atau melupakan parameter wajib, Helm menyediakan fitur validasi skema berbasis spesifikasi JSON Schema Draft-07:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Values Schema untuk Internal App",
  "type": "object",
  "required": ["replicaCount", "image", "resources"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 50,
      "description": "Jumlah replika Pod harus berupa angka bulat antara 1 hingga 50."
    },
    "image": {
      "type": "object",
      "required": ["repository", "tag"],
      "properties": {
        "repository": {
          "type": "string",
          "minLength": 3
        },
        "tag": {
          "type": "string",
          "minLength": 1
        }
      }
    },
    "resources": {
      "type": "object",
      "required": ["requests", "limits"],
      "properties": {
        "requests": {
          "type": "object",
          "required": ["cpu", "memory"]
        },
        "limits": {
          "type": "object",
          "required": ["cpu", "memory"]
        }
      }
    }
  }
}
```

Jika seseorang mencoba menjalankan instalasi dengan nilai `replicaCount: 0` atau tipe data salah, Helm akan menolak seketika sebelum manifest dikirim ke Kubernetes:
```text
Error: values don't meet the specifications of the schema(s) in the following chart(s):
internal-app:
- replicaCount: Must be greater than or equal to 1
```

---

## 7. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA MASALAH TEMPLATING HELM                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "can't evaluate field X in type interface {}" ]                 │
│  Gejala: `helm template` gagal merender dengan pesan evaluasi field nil.    │
│  Akar Masalah:                                                              │
│  - Anda masuk ke dalam blok `{{- with .Values.ingress }}`, lalu di dalamnya │
│    mencoba memanggil `{{ .Values.service.port }}`.                         │
│  - Di dalam blok `with`, konteks titik (`.`) sudah berpindah ke `ingress`   │
│    sehingga `.Values` tidak ditemukan!                                      │
│  Solusi SRE:                                                                │
│  Gunakan tanda dollar (`$`) untuk merujuk kembali ke root context:          │
│  {{ $.Values.service.port }}                                                │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: "error converting YAML to JSON: mapping values are not allowed"]│
│  Gejala: Template berhasil dirender sebagian, tapi gagal diparsing parser.  │
│  Akar Masalah:                                                              │
│  - Kesalahan penempatan spasi pada `nindent` atau penggunaan tab.           │
│  - Fungsi `toYaml` tidak diberi indentasi yang pas dengan induknya.         │
│  Solusi SRE:                                                                │
│  1. Render dan simpan hasil mentah untuk diinspeksi:                        │
│     helm template test ./internal-app --debug > /tmp/debug.yaml             │
│  2. Buka baris error di `/tmp/debug.yaml` dan periksa kelurusan spasinya.   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: NAMA RESOURCE MELEBIHI BATAS RFC 1123 (63 KARAKTER) ]           │
│  Gejala: `kubectl apply` menolak objek: "metadata.name: must be no more    │
│          than 63 characters".                                               │
│  Akar Masalah:                                                              │
│  - Gabungan nama release yang panjang dan nama chart melebihi 63 huruf.     │
│  Solusi SRE:                                                                │
│  Di dalam `_helpers.tpl`, selalu potong string menggunakan fungsi bawaan:   │
│  {{ printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }} │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Melakukan linting statis untuk mendeteksi pelanggaran best practice
helm lint ./internal-app

# 2. Merender template secara lokal tanpa melakukan koneksi jaringan (Dry-Run)
helm template my-release ./internal-app --values values-staging.yaml

# 3. Merender template dengan menampilkan pesan debug lengkap
helm template my-release ./internal-app --debug

# 4. Memvalidasi apakah nilai values memenuhi spesifikasi values.schema.json
helm lint ./internal-app --values values-invalid.yaml

# 5. Membungkus direktori chart menjadi arsip tarball (.tgz) terkompresi
helm package ./internal-app --destination ./dist/
```

---

## 9. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Dalam templating Helm, apa perbedaan fungsional mendasar antara konteks tanda titik (`.`) dan tanda dollar (`$`) saat berada di dalam perulangan `{{- range .Values.ports }}`?
- A. Tidak ada perbedaan sama sekali.
- B. Tanda titik (`.`) merepresentasikan elemen saat ini di dalam array iterasi, sedangkan tanda dollar (`$`) selalu merepresentasikan root context (puncak hierarki global), memungkinkan akses ke `.Values` atau `.Release`.
- C. Tanda dollar hanya digunakan untuk operasi matematika penambahan.
- D. Tanda titik otomatis menghapus spasi di sebelah kanan.

*Kunci Jawaban:* **B**. Di dalam blok *scope-shifting* seperti `range` atau `with`, tanda `.` berubah cakupannya, sehingga variabel root global wajib diakses menggunakan `$`.

### Soal 2 (Pilihan Ganda)
Mengapa berkas `values.schema.json` sangat direkomendasikan untuk disertakan dalam setiap Helm Chart tingkat produksi?
- A. Untuk mengenkripsi kata sandi secara otomatis.
- B. Bertindak sebagai *Contract Guardrail* yang memvalidasi tipe data, batas minimum/maksimum nilai, dan keberadaan parameter wajib secara otomatis sebelum rilis dikirim ke klaster Kubernetes.
- C. Mempercepat proses compile binary Go.
- D. Agar chart dapat berjalan di sistem operasi Windows.

*Kunci Jawaban:* **B**. Skema JSON menggagalkan eksekusi di tahap awal (*fail-fast*) jika input konfigurasi cacat atau melanggar aturan keamanan organisasi.

### Soal 3 (Pilihan Ganda)
Seorang engineer menjalankan `helm template my-app ./internal-app` dan hasilnya tidak mengeluarkan error sama sekali. Apakah hasil ini menjadi jaminan 100% bahwa aplikasi akan sukses berjalan di klaster k3s produksi?
- A. Ya, karena Helm otomatis menguji pod di klaster saat proses templating.
- B. Tidak, karena `helm template` adalah operasi *client-side rendering* yang hanya menguji kecocokan sintaks teks; perintah ini tidak memvalidasi ketersediaan image di registry, kecukupan memori node, atau validitas API admission webhook.
- C. Ya, asalkan versi Helm yang digunakan adalah versi 3.
- D. Tidak, kecuali jika file values ditulis dalam format JSON.

*Kunci Jawaban:* **B**. `helm template` murni merupakan generator teks lokal tanpa verifikasi *runtime state* di sisi klaster.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Anda sedang meninjau template `deployment.yaml` yang dibuat oleh junior engineer:
```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:latest"
          resources:
            {{ toYaml .Values.resources }}
```
Identifikasi 2 kesalahan fatal (satu terkait keandalan deployment dan satu terkait sintaks Go-template), serta tuliskan perbaikannya yang sesuai standar SRE!

*Panduan Solusi:*
1. **Identifikasi Kesalahan:**
   - **Kelemahan Keandalan (Image Tag `latest`):** Menggunakan tag `:latest` melanggar prinsip *reproducibility* dan *immutability*. Jika image di registry diupdate, Pod yang reboot akan menarik versi baru yang belum diuji, membuat *rollback* Helm mustahil dilakukan secara deterministik.
   - **Kesalahan Sintaks Templating (`toYaml` tanpa `nindent`):** Fungsi `{{ toYaml .Values.resources }}` tidak menyertakan `nindent`. Hasil render teks resources akan menempel di tepi kiri baris tanpa indentasi spasi yang sesuai dengan blok `containers`, memicu `YAML parse error` seketika.
2. **Penulisan Ulang Standar SRE:**
   ```yaml
   spec:
     template:
       spec:
         containers:
           - name: app
             image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
             resources:
               {{- toYaml .Values.resources | nindent 12 }}
   ```

---

## 10. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [Fase 2: Modul 2.1 — Konsep Inti Kubernetes](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.1-konsep-k3d/) — Memahami objek dasar Deployment, Service, dan Ingress.
- **Topik Berikutnya:**
  - [02 — Render dan Lifecycle Release](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/02-render-install-upgrade-rollback.md) — Menguasai siklus hidup rilis: install, upgrade, atomic, dan rollback.
  - [03 — Chart Repository dan OCI di GitLab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/modul-5.1-helm-fundamental/03-repository-chart-oci-gitlab.md) — Mendistribusikan chart ke GitLab OCI Registry.
