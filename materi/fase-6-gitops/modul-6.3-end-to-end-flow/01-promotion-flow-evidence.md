# 01 — Promotion Flow dan Immutable Evidence Chain

> **Ringkasan:** Memahami bagaimana kode aplikasi berpindah dari commit developer hingga berjalan aman di cluster production melalui pipeline GitOps, serta menyusun rantai bukti (*evidence chain*) yang tidak dapat dimanipulasi (*immutable*) untuk audit kepatuhan, keandalan, dan keamanan operasional SRE.

---

## 1. Konsep & Arsitektur Rantai Promosi (Promotion Flow)

### 1.1 Analogi Dunia Nyata: "Paspor Diplomatik & Surat Pengantar Bersegel (*Batch Release Certificate*)"
Bayangkan sebuah pabrik farmasi yang memproduksi obat injeksi steril:
1. **Fase Laboratorium (CI Build & Test):** Bahan baku diuji di lab. Jika lulus uji kimia dan sterilitas, batch obat dikemas ke dalam vial steril dan diberi nomor seri batch yang dicetak permanen (*immutable digest*). Batch ini tidak boleh diubah atau dicampur lagi.
2. **Karantina & Uji Klinis Terbatas (Staging Environment):** Batch obat dikirim ke klinik uji coba terbatas. Dokter memantau efek samping dan reaksi pasien selama periode observasi tertentu (*telemetry window*).
3. **Surat Pengantar Bersegel (*Immutable Evidence Chain*):** Petugas lab, dokter penguji, dan auditor mutu menandatangani dokumen resmi berisi: siapa pembuatnya, nomor batch vial, hasil tes lab, dan rekaman reaksi pasien di klinik uji coba.
4. **Izin Edar Massal & 4-Eyes Approval (Production Promotion):** Sebelum obat didistribusikan ke rumah sakit umum di seluruh negeri (*production*), kepala badan pengawas dan direktur medis memeriksa kelengkapan surat pengantar bersegel tersebut. Tanpa tanda tangan kedua pihak (*4-eyes principle*) dan tanpa nomor batch yang cocok persis, obat dilarang keras keluar dari gudang karantina.

Di dunia SRE modern, **Promotion Flow** adalah mekanisme perpindahan versi aplikasi dari lingkungan non-produksi (Development/Staging) menuju Production. Sedangkan **Immutable Evidence Chain** adalah jejak digital lengkap yang membuktikan bahwa artefak yang berjalan di production adalah artefak yang sama persis dengan yang diuji di staging, tanpa modifikasi, tanpa jalan pintas manual, dan telah disetujui oleh pihak yang berwenang.

---

### 1.2 Diagram Alur End-to-End Delivery Pipeline

Berikut adalah arsitektur lengkap alur promosi dari repositori kode aplikasi (*Application Repo*) hingga cluster Kubernetes yang direkonsiliasi oleh ArgoCD:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. REPOSITORI APLIKASI (Developer Space)                                    │
│    Developer Commit ──> GitLab CI Pipeline:                                 │
│                         ├─ Lint & Unit Test                                 │
│                         ├─ Multi-Arch Build (ARM64 & AMD64 via buildx)      │
│                         ├─ Security Scan (Trivy Vulnerability & SBOM)       │
│                         └─ Push Container Image ke OCI Registry             │
│                            Digest: sha256:7f83b1... (IMMUTABLE)             │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼ (Automated Bot / CI Script)
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. REPOSITORI GITOPS (Infrastructure & Manifest Space)                      │
│                                                                             │
│    [Stage 1: Staging Promotion]                                             │
│    Bot membuat Git Commit / Merge Request di branch `main`:                 │
│    - Mengubah file `apps/payment/environments/staging/values.yaml`          │
│    - image.digest = sha256:7f83b1...                                        │
│                                      │                                      │
│                                      ▼                                      │
│    ArgoCD Staging Controller mendeteksi perubahan:                          │
│    - GitOps Pull & Reconcile ──> Pod Staging di-update                     │
│    - Jalankan Automated Smoke Test & Kumpulkan Metrik Latensi (Fase 7)      │
│    - Status: Staging Verified (Lulus Uji)                                  │
│                                      │                                      │
│    [Stage 2: Production Promotion Gate]                                     │
│    Reviewer / SRE membuat MR ke `apps/payment/environments/prod/values.yaml`│
│    - Melampirkan `evidence-summary.json` (Hasil tes staging & scan)         │
│    - Memerlukan "4-Eyes Approval" (Senior SRE + Tech Lead)                  │
│    - MR di-merge ke branch `main`                                           │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼ (ArgoCD Pull-Based Reconcile)
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. RUNTIME KUBERNETES PRODUCTION (k3s On-Premises)                          │
│                                                                             │
│    ArgoCD Production Controller mendeteksi commit baru di GitOps repo:      │
│    - Sync Status: Synced                                                    │
│    - Health Status: Progressing ──> Healthy                                 │
│    - Pod Production ditarik menggunakan digest `sha256:7f83b1...`           │
│    - Observability Engine (Prometheus/Grafana) memantau SLO Burn Rate       │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Prinsip Isolasi Kredensial (Zero Production Kubeconfig in CI):**
> Pipeline CI pada repositori aplikasi **TIDAK BOLEH** memegang kredensial `kubeconfig` cluster production! Pipeline CI hanya bertugas mempublikasikan image ke registry dan membuat Pull/Merge Request ke repositori GitOps. ArgoCD di dalam internal network cluster yang bertugas menarik (*pull*) perubahan tersebut. Ini mencegah kebocoran akses cluster apabila pipeline developer disusupi.

---

## 2. Anatomi Evidence Chain & Matriks Batasan Bukti

Dalam audit kepatuhan (seperti ISO 27001, SOC 2, atau audit internal perbankan), pernyataan *"Aplikasi sudah dites dan aman"* tidak bernilai tanpa adanya bukti digital yang terverifikasi. SRE wajib menyusun rantai bukti (*chain of custody*) dari ujung ke ujung.

### 2.1 Matriks Batasan Bukti per Tahapan Pipeline

Setiap tahapan menghasilkan bukti (*evidence*) spesifik, namun masing-masing memiliki batasan yang tidak boleh diasumsikan mewakili tahapan lain:

| Tahapan Pipeline | Bukti Minimum (*Minimum Evidence*) | Batasan & Yang **Bukan** Bukti |
|---|---|---|
| **1. Source Code** | - Commit SHA lengkap (40 karakter)<br>- Identitas Author & Reviewer<br>- Signed Git Commit (GPG/SSH signature) | Mengetahui kode di-commit **bukan** bukti bahwa kode tersebut bisa dikompilasi atau bebas error. |
| **2. CI Build & Test** | - Pipeline ID & Job ID unik<br>- Laporan Unit Test & Code Coverage<br>- Status Exit Code = 0 | Pipeline sukses **bukan** bukti bahwa artefak sudah ter-deploy ke server atau aman dari celah keamanan pihak ketiga. |
| **3. Security & Artifact** | - Container Image Digest (`sha256:...`)<br>- Laporan Trivy scan (0 Critical/High)<br>- Software Bill of Materials (SBOM) | Image digest **bukan** bukti bahwa pod berhasil dijalankan di Kubernetes tanpa CrashLoopBackOff. |
| **4. GitOps Change** | - Git Commit SHA di repositori GitOps<br>- Diff manifes (`values.yaml` diff)<br>- Approval log dari Merge Request | Konfigurasi tercatat di Git **bukan** bukti bahwa cluster sudah menerapkan perubahan tersebut. |
| **5. ArgoCD Sync** | - Revision SHA yang di-sync<br>- Sync Status: `Synced`<br>- Timestamp sinkronisasi | `Synced` hanya membuktikan manifes dikirim ke API server, **bukan** bukti bahwa pod healthy dan siap melayani trafik. |
| **6. Kubernetes Rollout**| - Deployment generation & replica count<br>- Health Status: `Healthy`<br>- Hasil `livenessProbe` & `readinessProbe` | Pod running **bukan** bukti bahwa fungsi bisnis aplikasi berjalan normal (misal: koneksi database lambat atau query gagal). |
| **7. Smoke Test & App** | - HTTP status code 200 pada endpoint kritis<br>- Log transaksi sukses tanpa error 5xx<br>- Latensi p95 memenuhi batas toleransi | Smoke test lolos **bukan** bukti stabilitas jangka panjang di bawah beban trafik puncak (*peak load*). |
| **8. Production SLO** | - Metrik Prometheus error rate & burn rate<br>- Evaluasi jendela observasi (15–60 menit)<br>- Dashboard Grafana & Alertmanager silence status | Membutuhkan integrasi penuh dengan telemetry stack (Mimir/Prometheus di Fase 7). |

---

### 2.2 Struktur Dokumen Evidence JSON (Immutable Audit Record)

Berikut adalah contoh skema rekaman audit digital (`gitops-evidence.json`) yang dihasilkan secara otomatis oleh pipeline promosi dan disimpan sebagai artefak audit permanen:

```json
{
  "schema_version": "v1.2.0",
  "application": "payment-service",
  "promotion_id": "prom-20261017-084512",
  "timestamp_utc": "2026-10-17T08:45:12Z",
  "source_repository": {
    "url": "https://gitlab.example.com/apps/payment-service.git",
    "commit_sha": "a1b2c3d4e5f67890123456789abcdef012345678",
    "branch": "main",
    "committer": "developer@example.com"
  },
  "ci_verification": {
    "pipeline_id": 98234,
    "pipeline_url": "https://gitlab.example.com/apps/payment-service/-/pipelines/98234",
    "unit_test_status": "PASSED",
    "security_scan": {
      "scanner": "trivy-0.50.0",
      "critical_vulnerabilities": 0,
      "high_vulnerabilities": 0,
      "scan_report_artifact": "gl-container-scanning-report.json"
    }
  },
  "immutable_artifacts": {
    "container_image": {
      "registry": "registry.gitlab.example.com/apps/payment-service",
      "tag": "v1.4.2",
      "digest": "sha256:7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069",
      "architectures": ["linux/arm64", "linux/amd64"]
    },
    "helm_chart": {
      "oci_url": "oci://registry.gitlab.example.com/charts/payment-service",
      "version": "1.4.2",
      "digest": "sha256:d8e9f0123456789abcdef0123456789abcdef0123456789abcdef0123456789a"
    }
  },
  "gitops_promotion": {
    "repository_url": "https://gitlab.example.com/gitops/onprem-clusters.git",
    "target_environment": "production",
    "merge_request_id": 412,
    "merge_request_url": "https://gitlab.example.com/gitops/onprem-clusters/-/merge_requests/412",
    "gitops_commit_sha": "f9e8d7c6b5a432109876543210fedcba98765432",
    "approvers": [
      "lead-sre@example.com",
      "security-officer@example.com"
    ]
  },
  "argocd_sync_verification": {
    "application_name": "prod-payment-service",
    "cluster_target": "https://k3s-prod-api.internal:6443",
    "sync_status": "Synced",
    "health_status": "Healthy",
    "observed_revision": "f9e8d7c6b5a432109876543210fedcba98765432"
  },
  "telemetry_verification": {
    "staging_observation_window_minutes": 60,
    "smoke_test_status": "PASSED",
    "error_rate_percentage": 0.002,
    "p95_latency_ms": 42
  }
}
```

> [!CAUTION]
> **Aturan Higienitas Rahasia (*Secrets Hygiene*) pada Evidence Record:**
> Dokumen audit seperti di atas sering kali diarsipkan ke sistem log terpusat atau bucket MinIO jangka panjang. **DILARANG KERAS** menyertakan data sensitif berikut ke dalam file evidence:
> 1. Private SSH Keys atau Kubeconfig cluster.
> 2. Password database, token API, atau GitLab Personal Access Token (PAT).
> 3. Nilai manifes Kubernetes `Secret` yang sudah di-render atau plain text environment variables.
> Selalu pastikan nilai rahasia di-redact atau hanya direferensikan melalui *Key Vault Secret Path* (misal: `vault://secret/prod/db#password`).

---

## 3. Alur Promosi Bertingkat: Staging ke Production

Di lingkungan enterprise on-premises, mempromosikan perubahan dari Staging ke Production tidak boleh dilakukan dengan menyalin tag `latest` atau mengubah konfigurasi secara acak. Terdapat 7 gerbang verifikasi (*promotion gates*) yang wajib dipenuhi:

```text
┌────────────────┐      ┌────────────────┐      ┌────────────────┐
│ 1. Artifact &  │ ──>  │ 2. Helm Chart  │ ──>  │ 3. GitOps Diff │
│ Compatibility  │      │ & DB Migration │      │ Verification   │
└────────────────┘      └────────────────┘      └────────────────┘
                                                         │
┌────────────────┐      ┌────────────────┐               ▼
│ 6. Owner &     │ <──  │ 5. Approval &  │ <──  ┌────────────────┐
│ Comm Plan      │      │ Freeze Check   │      │ 4. Staging     │
└────────────────┘      └────────────────┘      │ Telemetry Gate │
                                                └────────────────┘
```

### 3.1 Tujuh Gerbang Verifikasi Promosi (The 7 Gates)

1. **Artifact & Architecture Compatibility Gate:**
   - Memastikan container image memiliki digest `sha256` yang tetap.
   - Memastikan image memiliki multi-architecture manifest (`linux/arm64` untuk dev/staging di Mac/OrbStack, dan `linux/amd64` untuk server bare-metal production di data center).
2. **Database Migration & Backward Compatibility Gate:**
   - Apakah versi baru membutuhkan perubahan skema database (*ALTER TABLE*)?
   - Skema database harus bersifat *backward-compatible* (kode versi lama harus tetap bisa berjalan dengan skema database baru untuk mendukung rolling update tanpa downtime).
3. **GitOps Diff Verification Gate:**
   - Reviewer memeriksa perbedaan baris kode (*diff*) pada repositori GitOps:
     ```diff
     - image:
     -   digest: sha256:3a1b... # v1.4.1
     + image:
     +   digest: sha256:7f83... # v1.4.2
     ```
   - Memastikan tidak ada variabel environment production yang terhapus atau tertukar dengan environment staging.
4. **Staging Telemetry Window Gate:**
   - Aplikasi versi kandidat wajib telah berjalan di environment staging minimal selama durasi tertentu (misal: 30–60 menit).
   - Selama jendela tersebut, automated integration test atau traffic simulation dijalankan. Metrik error rate harus berada di bawah ambang batas SLO.
5. **Change Freeze & Incident Check Gate:**
   - Memeriksa apakah saat ini sedang berlangsung insiden P1/P2 di cluster production. Jika ada insiden aktif, seluruh promosi dibekukan (*Freeze*).
   - Memeriksa kalender bisnis (misal: akhir bulan penggajian / Flash Sale) di mana *production change freeze* diberlakukan.
6. **4-Eyes Approval Gate:**
   - Merge Request ke branch production GitOps membutuhkan minimal 2 persetujuan: 1 dari tim pengembang (Application Owner) dan 1 dari tim SRE/Infrastruktur.
7. **Rollback Reference & Communication Gate:**
   - Sebelum tombol merge ditekan, tiket rilis harus mencantumkan: *"Jika deployment gagal, digest versi stabil sebelumnya adalah `sha256:3a1b...`"*.
   - Saluran komunikasi tim (Slack/Discord/Telegram on-call) diberi notifikasi bahwa rilis sedang berlangsung.

---

### 3.2 Implementasi GitLab CI Promosi Otomatis ke GitOps Repo

Berikut adalah contoh skrip pipeline GitLab CI (`.gitlab-ci.yml`) pada repositori aplikasi yang secara aman membuat Merge Request ke repositori GitOps setelah build dan test berhasil:

```yaml
stages:
  - build-and-test
  - package-and-scan
  - promote-staging
  - create-prod-mr

variables:
  GITOPS_REPO_URL: "https://oauth2:${GITOPS_DEPLOY_TOKEN}@gitlab.example.com/gitops/cluster-manifests.git"
  IMAGE_NAME: "registry.example.com/apps/payment-service"

# Tahap 1 & 2: Build, Scan, dan Dapatkan Digest
package-container:
  stage: package-and-scan
  image: docker:24.0-cli
  services:
    - docker:24.0-dind
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker buildx create --use
    - docker buildx build --platform linux/amd64,linux/arm64 -t "$IMAGE_NAME:$CI_COMMIT_SHORT_SHA" --push .
    # Ambil immutable image digest secara deterministik
    - IMAGE_DIGEST=$(docker buildx imagetools inspect "$IMAGE_NAME:$CI_COMMIT_SHORT_SHA" --raw | sha256sum | awk '{print $1}')
    - echo "IMMUTABLE_DIGEST=sha256:$IMAGE_DIGEST" >> build.env
    - echo "APP_VERSION=$CI_COMMIT_TAG" >> build.env
  artifacts:
    reports:
      dotenv: build.env

# Tahap 3: Otomatis Update Staging di Repositori GitOps
promote-staging:
  stage: promote-staging
  image: alpine/git:latest
  needs:
    - job: package-container
      artifacts: true
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - git config --global user.name "GitOps Bot"
    - git config --global user.email "gitops-bot@example.com"
    - git clone "$GITOPS_REPO_URL" gitops-repo
    - cd gitops-repo/apps/payment/environments/staging
    # Update baris digest di values.yaml menggunakan sed
    - sed -i "s|digest: .*|digest: \"$IMMUTABLE_DIGEST\"|g" values.yaml
    - git add values.yaml
    - git commit -m "chore(staging): update payment-service to $IMMUTABLE_DIGEST [skip ci]"
    - git push origin main
```

---

## 4. Taksonomi Kegagalan (Failure Taxonomy) & Protokol Eskalasi

Ketika sebuah rilis tidak berjalan semestinya, SRE harus mampu mengidentifikasi di lapisan mana kegagalan terjadi dalam hitungan detik. Menggunakan klasifikasi yang salah akan membuang waktu investigasi (misal: menyalahkan ArgoCD padahal database kehabisan koneksi).

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ MATRIKS IDENTIFIKASI & PENANGANAN KEGAGALAN RILIS                           │
├──────────────┬──────────────────────────────┬───────────────────────────────┤
│ Lapisan      │ Gejala & Indikator Error     │ Tindakan SRE & Owner          │
├──────────────┼──────────────────────────────┼───────────────────────────────┤
│ 1. CI        │ Job unit test exit 1,        │ Stop alur promosi!            │
│    Failure   │ buildx compiler error.       │ Owner: Software Developer.    │
│              │                              │ Tindakan: Perbaiki kode di MR.│
├──────────────┼──────────────────────────────┼───────────────────────────────┤
│ 2. Registry  │ Error `401 Unauthorized`     │ Jangan commit ke GitOps!      │
│    Failure   │ atau `quota exceeded`.       │ Owner: Platform / Infra SRE.  │
│              │                              │ Cek disk space / token harbor.│
├──────────────┼──────────────────────────────┼───────────────────────────────┤
│ 3. Manifest  │ Helm template render error,  │ Revert commit di GitOps repo. │
│    Failure   │ schema validation gagal.     │ Owner: GitOps Author / SRE.   │
│              │                              │ Jalankan `helm lint` lokal.   │
├──────────────┼──────────────────────────────┼───────────────────────────────┤
│ 4. ArgoCD    │ Sync status `Failed`         │ Inspeksi event ArgoCD:        │
│    Sync      │ Namespace tidak ditemukan,   │ Owner: SRE Cluster Admin.     │
│    Failure   │ CRD belum terpasang.         │ Cek RBAC & Admission Webhook. │
├──────────────┼──────────────────────────────┼───────────────────────────────┤
│ 5. Rollout   │ Pod `ImagePullBackOff`,      │ Rollback Git commit segera!   │
│    Failure   │ `CrashLoopBackOff`, OOMKilled│ Owner: SRE & Developer.       │
│              │ Exit code 137 / 1.           │ Cek `kubectl describe pod`.   │
├──────────────┼──────────────────────────────┼───────────────────────────────┤
│ 6. App Logic │ HTTP 500 Internal Server     │ Eksekusi Emergency Revert MR! │
│    Failure   │ Error, DB connection refused.│ Owner: Lead App Developer.    │
│              │ Liveness probe lolos tapi 5xx│ Cek application runtime logs. │
├──────────────┼──────────────────────────────┼───────────────────────────────┤
│ 7. SLO Burn  │ Latensi p95 naik drastis     │ Rollback bertahap / Abort!    │
│    Failure   │ (>500ms), error budget       │ Owner: Incident Commander.    │
│              │ terkuras > 10% dalam 1 jam.  │ Deklarasikan minor incident.  │
└──────────────┴──────────────────────────────┴───────────────────────────────┘
```

---

## 5. Protokol Pemulihan & Rebuild On-Premises (Disaster Recovery)

Jika terjadi insiden fatal di mana sebuah commit yang salah merusak konfigurasi sistem, atau cluster on-prem mengalami kegagalan hardware fisik total (*catastrophic failure*), SRE mengikuti hierarki pemulihan berikut:

### 5.1 Hierarki Revert Aplikasi (Fast-Track vs Clean Revert)
1. **Clean Revert (Standar Produksi):**
   - Jalankan `git revert <COMMIT-SHA>` pada branch `main` repositori GitOps.
   - Buat Merge Request darurat (*Emergency MR*).
   - Setelah di-merge, ArgoCD secara otomatis merekonsiliasi cluster kembali ke digest versi lama yang stabil.
   - *Keuntungan:* Sejarah Git tetap utuh, audit trail mencatat siapa yang melakukan pembatalan dan alasannya.
2. **Fast-Track UI/CLI Rollback (Hanya saat Insiden P1/Emergency):**
   - Di antarmuka ArgoCD UI atau CLI: nonaktifkan opsi `Automated Self-Heal` sementara waktu.
   - Jalankan rollback ke revisi sebelumnya via ArgoCD:
     ```bash
     argocd app rollback prod-payment-service <HISTORY-ID>
     ```
   - **PENTING:** Setelah situasi darurat teratasi, SRE **WAJIB** segera membuat commit revert di repositori GitOps dan mengaktifkan kembali `selfHeal: true`. Jika tidak, drift akan kembali muncul saat sinkronisasi berikutnya!

### 5.2 Alur Total Rebuild Cluster On-Premises (Metal to App Recovery)
Jika seluruh mesin fisik atau VM cluster mati total, rekonstruksi sistem dilakukan secara berurutan sesuai batas tanggung jawab modul di kurikulum ini:

```text
┌─────────────────────────┐
│ 1. OpenTofu (Fase 3)    │ ──> Provisioning metadata VM, virtual disk & bridge network
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 2. Ansible (Fase 4)     │ ──> OS hardening, install k3s server & worker, deploy MetalLB
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 3. Bootstrap (Fase 6)   │ ──> Install ArgoCD Helm Chart & hubungkan ke GitOps repository
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 4. GitOps Rehydration   │ ──> ArgoCD menarik ApplicationSet & seluruh workload otomatis berjalan!
└─────────────────────────┘
```

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. VERIFIKASI IMMUTABLE DIGEST & REGISTRY
# ==============================================================================

# Menampilkan digest sha256 asli dari container image di remote registry
docker buildx imagetools inspect registry.example.com/apps/payment-service:v1.4.2

# Mengambil digest sha256 image menggunakan crane (Google container tool)
crane digest registry.example.com/apps/payment-service:v1.4.2

# ==============================================================================
# 2. INSPEKSI GITOPS & ARGOCD RUNTIME
# ==============================================================================

# Memeriksa riwayat deployment dan revision ID pada aplikasi ArgoCD
argocd app history prod-payment-service

# Melihat detail perbedaan (diff) antara desired state di Git vs live cluster
argocd app diff prod-payment-service

# Memaksa sinkronisasi manual ke target revision tertentu
argocd app sync prod-payment-service --revision a1b2c3d4e5f6

# ==============================================================================
# 3. VERIFIKASI RUNTIME KUBERNETES
# ==============================================================================

# Memeriksa status rollout dari deployment di production
kubectl rollout status deployment/payment-service -n production

# Melihat riwayat revisi rollout di Kubernetes
kubectl rollout history deployment/payment-service -n production

# Menginspeksi image digest aktual yang sedang berjalan di dalam Pod
kubectl get pods -n production -l app=payment-service \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].imageID}{"\n"}{end}'
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa pipeline CI pada repositori kode aplikasi **dilarang** menyimpan file kredensial `kubeconfig` cluster production untuk menjalankan `kubectl apply` secara langsung?
- A. Karena `kubectl apply` tidak mendukung eksekusi di lingkungan container CI.
- B. Karena melanggar prinsip *least privilege* dan meningkatkan risiko keamanan; jika pipeline CI disusupi, penyerang mendapatkan akses kontrol penuh ke cluster production.
- C. Karena Kubernetes tidak mengizinkan koneksi API server dari luar subnet jaringan lokal underlay.
- D. Karena `kubeconfig` akan membuat proses kompilasi kode aplikasi menjadi lebih lambat hingga 50%.

#### Soal 2
Sebuah image di-push ke registry dengan tag `payment-service:v1.4.2`. Beberapa jam kemudian, developer menimpa tag tersebut dengan commit baru tanpa mengubah nomor versinya. Bagaimana SRE mencegah risiko ketidakpastian (*ambiguity*) ini pada alur promosi GitOps?
- A. Dengan selalu menggunakan tag `:latest` agar cluster selalu mendapatkan versi terbaru.
- B. Dengan melarang penggunaan container image di environment production.
- C. Dengan mereferensikan artefak menggunakan **Immutable Image Digest** (`sha256:...`) alih-alih tag teks biasa.
- D. Dengan me-restart seluruh node k3s setiap kali ada update aplikasi.

#### Soal 3
Setelah ArgoCD berhasil melakukan sync dengan status `Synced`, pod aplikasi mengalami `CrashLoopBackOff` akibat kesalahan variabel koneksi database. Manakah pernyataan yang paling tepat mengenai status evidence chain pada tahap ini?
- A. Rantai promosi sukses secara penuh karena status ArgoCD sudah `Synced`.
- B. Kegagalan berada pada lapisan *Rollout/Application Failure*; status `Synced` hanya membuktikan manifes berhasil dikirim ke API Server, bukan bukti pod berjalan sehat.
- C. SRE harus segera menghapus namespace production untuk membersihkan cluster.
- D. GitLab CI runner harus di-restart karena gagal menjalankan unit test.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Anda adalah SRE On-Call di sebuah perusahaan perbankan. Pada pukul 21:00, sebuah rilis promosi ke production untuk service `transfer-api` selesai di-merge ke repositori GitOps. Lima menit setelah ArgoCD melakukan sync, dashboard Grafana membunyikan alert P1: *Error Rate HTTP 500 melonjak dari 0.01% menjadi 14%*, dan *Latency p95 naik menjadi 4.5 detik*. 

Setelah diinvestigasi cepat, ditemukan bahwa tim developer menyertakan library enkripsi baru yang membutuhkan konfigurasi environment variable `ENCRYPTION_KEY_PATH`, namun variabel tersebut lupa dimasukkan ke dalam `values.yaml` production.

**Tugas Anda:**
1. Apa tindakan mitigasi darurat pertama yang harus Anda ambil untuk menghentikan dampak bisnis terhadap nasabah?
2. Jelaskan langkah pemulihan jangka pendek (*recovery*) menggunakan prinsip GitOps yang benar.
3. Sebutkan 2 tindakan pencegahan (*Action Items*) agar insiden serupa tidak terulang pada promosi rilis di masa depan!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Praktik Push-based CI di mana CI menyimpan master admin kubeconfig sangat berbahaya. Pola Pull-based GitOps (ArgoCD) mengisolasi akses di dalam cluster dan mencegah kebocoran kredensial produksi jika server CI dibobol.
2. **C.** Tag container dapat ditimpa (*mutable*), sedangkan nilai hash kriptografis `sha256` (*digest*) bersifat *immutable*. Menggunakan digest menjamin bahwa kode yang berjalan di production adalah byte demi byte yang sama persis dengan yang diuji di staging.
3. **B.** Status `Synced` pada ArgoCD hanya menyatakan bahwa desired state di Git cocok dengan live resource spec di Kubernetes. Kesiapan operasional workload ditentukan oleh status `Healthy`, event pod, dan metrik aplikasi di runtime.

#### Solusi Soal Kasus Nyata:
1. **Tindakan Mitigasi Darurat:**
   - Masuk ke antarmuka ArgoCD atau CLI, jalankan emergency rollback ke revisi sebelumnya yang stabil:
     `argocd app rollback prod-transfer-api <PREVIOUS-STABLE-REVISION>` atau buat *hotfix commit* darurat via Git.
   - Verifikasi bahwa traffic kembali normal dan alert P1 di Alertmanager telah terselesaikan (*resolved*).
2. **Langkah Pemulihan Jangka Pendek (GitOps Clean Recovery):**
   - Lakukan `git revert` terhadap commit yang bermasalah di repositori GitOps.
   - Buka Pull/Merge Request darurat bertanda `[EMERGENCY-REVERT]`, minta persetujuan cepat dari Lead SRE, dan merge ke branch `main`.
   - Pastikan ArgoCD kembali dalam status `Synced` dan `Healthy` dengan konfigurasi lama yang terbukti stabil.
3. **Tindakan Pencegahan (Action Items):**
   - **Automated Manifest Validation di CI:** Pasang tools seperti `conftest` atau `kubeval/datree` di pipeline CI repositori GitOps untuk memvalidasi bahwa seluruh *required environment variables* terdefinisi sebelum PR dapat di-merge.
   - **Staging Gate Verification:** Wajibkan smoke test otomatis di environment staging yang secara eksplisit memanggil endpoint transaksi nyata; jika `ENCRYPTION_KEY_PATH` hilang di staging, pipeline promosi akan otomatis menolak (*fail-closed*) pembuatan MR ke production.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah memahami alur promosi dan pentingnya bukti yang tidak dapat dimanipulasi (*immutable evidence*), lanjutkan ke materi berikutnya:
- [02 — Progressive Delivery dan Rollback](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-6-gitops/modul-6.3-end-to-end-flow/02-progressive-delivery-rollback.md) — Mempelajari teknik rilis bertahap tingkat lanjut (Canary Deployment & Blue-Green) menggunakan Argo Rollouts dan integrasi metrik evaluasi otomatis.
- [Fase 7 Observability](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/README.md) — Mengonfigurasi Prometheus, Loki, dan Tempo untuk mengukur telemetry gate secara akurat.
