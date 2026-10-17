# 01 — Grafana USE/RED dan Dashboard as Code

> **Ringkasan:** Merancang dashboard observabilitas yang mampu menjawab pertanyaan insiden dalam hitungan detik menggunakan metodologi **USE (Utilization, Saturation, Errors)** untuk infrastruktur perangkat keras dan **RED (Rate, Errors, Duration)** untuk layanan aplikasi, serta mengelola siklus hidup dashboard sebagai kode (**Dashboard as Code**) melalui GitOps tanpa ketergantungan pada klik manual.

---

## 1. Metodologi Desain Dashboard SRE: Metode USE vs Metode RED

### 1.1 Analogi Dunia Nyata: "Panel Mesin Mobil (Metode USE) vs Layanan Kasir Supermarket (Metode RED)"
Banyak pemula memenuhi layar Grafana dengan puluhan grafik acak yang terlihat warna-warni tetapi sama sekali tidak membantu saat terjadi insiden pukul 02:00 pagi. Dalam disiplin SRE, kita memisahkan dashboard menjadi dua sudut pandang yang tegas:

1. **Metode USE (*Utilization, Saturation, Errors*) — Untuk Infrastruktur & Perangkat Keras (Node/Server):**
   Bayangkan mesin mobil Anda:
   - **Utilization (Tingkat Pemakaian):** Berapa persen tenaga mesin atau kapasitas jalan yang sedang terpakai saat ini? *(Contoh: CPU terpakai 80%, RAM terpakai 75%)*.
   - **Saturation (Tingkat Kejenuhan / Antrean):** Ketika kapasitas sudah penuh, **berapa panjang antrean pekerjaan yang terpaksa menunggu di belakang?** *(Contoh: 12 proses mengantre di CPU Run Queue / Load Average, atau proses menunggu giliran baca/tulis hard disk `iowait`)*.
   - **Errors (Jumlah Kegagalan Komponen):** Apakah ada komponen fisik yang rusak atau menolak bekerja? *(Contoh: Network interface `tx/rx dropped packets`, atau `OOMKiller` membunuh proses)*.

2. **Metode RED (*Rate, Errors, Duration*) — Untuk Layanan Aplikasi / Microservices:**
   Bayangkan loket kasir di sebuah supermarket:
   - **Rate (Laju Kedatangan):** Berapa banyak pembeli per detik yang datang ke meja kasir? *(Contoh: `250 requests/detik`)*.
   - **Errors (Laju Kegagalan):** Berapa banyak transaksi pembayaran kartu yang ditolak atau gagal diproses? *(Contoh: `0,2% HTTP 5xx Error Ratio`)*.
   - **Duration (Durasi / Waktu Tunggu):** Berapa lama pembeli harus berdiri menunggu hingga struk belanja keluar? *(Contoh: `p95 Latency = 180 milidetik`)*.

---

### 1.2 Tabel Matriks Kueri PromQL Lengkap: Metode USE & Metode RED

Berikut adalah rumus PromQL standar produksi yang wajib dipasang pada kedua jenis dashboard tersebut:

#### A. Matriks Dashboard Infrastruktur — Metode USE (Sumber: `node_exporter`)

| Komponen Fisik | 1. Utilization (Pemakaian %) | 2. Saturation (Antrean / Kejenuhan) | 3. Errors (Kegagalan Hardware/OS) |
|---|---|---|---|
| **CPU** | `100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))` | `node_load1 / count without (cpu, mode) (node_cpu_seconds_total{mode="idle"})`<br>*(Rasio Load Average terhadap jumlah Core)* | `rate(node_cpu_guest_seconds_total[5m])` atau Throttling container: `rate(container_cpu_cfs_throttled_periods_total[5m])` |
| **Memory (RAM)** | `100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))` | `rate(node_vmstat_pswpin[5m]) + rate(node_vmstat_pswpout[5m])`<br>*(Aktivitas keluar-masuk Memory Swap)* | `increase(node_vmstat_oom_kill[5m])`<br>*(Jumlah proses dibunuh oleh Kernel OOM Killer)* |
| **Storage Disk** | `100 * (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes))` | `rate(node_disk_io_time_weighted_seconds_total[5m])`<br>*(Kedalaman antrean I/O disk)* | `rate(node_filesystem_device_error[5m])` |
| **Network Card** | `rate(node_network_receive_bytes_total[5m]) * 8` | `rate(node_network_receive_drop_total[5m]) + rate(node_network_transmit_drop_total[5m])` | `rate(node_network_receive_errs_total[5m]) + rate(node_network_transmit_errs_total[5m])` |

#### B. Matriks Dashboard Layanan — Metode RED (Sumber: Aplikasi / Recording Rules)

| Dimensi RED | Pertanyaan Operasional SRE | Kueri PromQL / Recording Rule Standar | Unit Grafik Grafana |
|---|---|---|---|
| **Rate (R)** | Berapa total beban trafik yang masuk ke layanan ini? | `sum by (service) (rate(http_requests_total{sre_environment="$env"}[5m]))` | `reqps` (Requests/sec) |
| **Errors (E)** | Berapa persentase permintaan pengguna yang gagal (HTTP 5xx)? | `sum by (service) (rate(http_requests_total{sre_environment="$env", status=~"5.."}[5m])) / sum by (service) (rate(http_requests_total{sre_environment="$env"}[5m]))` | `percentunit` (0.0 – 1.0 ➔ 0% – 100%) |
| **Duration (D)**| Berapa waktu respons untuk 95% dan 99% pengguna (`p95` & `p99`)? | `histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket{sre_environment="$env"}[5m])))` | `s` (Seconds — otomatis berubah ke `ms`) |

---

## 2. Dashboard as Code melalui GitOps (Zero ClickOps)

### 2.1 Mengapa Mengedit Dashboard Langsung di Web UI Grafana Berbahaya?
Jika seorang engineer membuat atau mengubah panel grafik langsung di browser Grafana (*ClickOps*):
1. **Hilang Saat Pod Restart:** Jika pod Grafana menggunakan penyimpanan ephemeral atau direbuild di node baru, seluruh perubahan manual akan musnah.
2. **Tidak Ada Jejak Audit & Code Review:** Jika seseorang tidak sengaja menghapus filter `status=~"5.."` pada panel Error Rate, tidak ada yang tahu siapa yang mengubahnya dan tidak bisa di-*git revert*.
3. **Inkonsistensi Antar Environment:** Dashboard di Staging dan Production menjadi berbeda-beda.

### 2.2 Arsitektur Provisioning Dashboard Otomatis di Kubernetes (k8s-sidecar)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│             ALUR GITOPS DASHBOARD AS CODE DI KUBERNETES                     │
│                                                                             │
│  [1. Repositori GitOps]                                                     │
│   File: `dashboards/red-service-overview.json`                              │
│   Dibungkus dalam `ConfigMap` dengan label: `grafana_dashboard: "1"`        │
│                              │                                              │
│                              ▼ (ArgoCD Sync)                                │
│  [2. Kubernetes API Server]                                                 │
│   ConfigMap `red-service-dashboard-cm` dibuat/diperbarui di namespace       │
│                              │                                              │
│                              ▼ (Watch Event)                                │
│  [3. Pod Grafana]                                                           │
│   ┌─────────────────────────────────────┐    ┌───────────────────────────┐  │
│   │ Container Sidecar: `k8s-sidecar`    │───>│ Container Utama: `grafana`│  │
│   │ Menyalin file JSON ke folder lokal: │    │ Otomatis memuat ulang     │  │
│   │ `/tmp/dashboards/red-overview.json` │    │ dashboard dalam 10 detik! │  │
│   └─────────────────────────────────────┘    └───────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 2.3 Contoh Manifes GitOps ConfigMap Dashboard RED (`dashboard-red-cm.yaml`)

Berikut adalah contoh deklarasi `ConfigMap` tingkat produksi yang memuat definisi JSON Dashboard RED yang bersih dari kredensial, menggunakan variabel ber-allowlist (`environment`), memiliki unit yang tepat, dan menyertakan tautan Runbook:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sre-service-red-dashboard
  namespace: monitoring
  labels:
    # Label penanda wajib agar otomatis ditemukan oleh sidecar Grafana!
    grafana_dashboard: "1"
    app.kubernetes.io/part-of: observability-stack
  annotations:
    # Mengelompokkan dashboard ke dalam folder "SRE Production" di UI Grafana
    k8s-sidecar-target-directory: "/tmp/dashboards/SRE-Production"
data:
  service-red-overview.json: |
    {
      "uid": "sre-red-overview-v1",
      "title": "SRE — Service RED Golden Signals",
      "tags": ["sre", "red", "production"],
      "timezone": "utc",
      "schemaVersion": 39,
      "version": 1,
      "editable": false,
      "links": [
        {
          "title": "SOP & Runbook Insiden RED",
          "type": "link",
          "url": "https://git.internal.example.com/sre/runbooks/-/blob/main/ServiceRED.md",
          "targetBlank": true
        }
      ],
      "templating": {
        "list": [
          {
            "name": "env",
            "label": "Environment",
            "type": "custom",
            "query": "production,staging",
            "current": { "text": "production", "value": "production" }
          }
        ]
      },
      "panels": [
        {
          "id": 1,
          "title": "1. Request Rate (req/s)",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 8, "x": 0, "y": 0 },
          "datasource": { "type": "prometheus", "uid": "mimir-prod-uid" },
          "targets": [
            {
              "refId": "A",
              "expr": "service:http_requests:rate5m{sre_environment=\"$env\"}",
              "legendFormat": "{{service}}"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "reqps",
              "custom": { "drawStyle": "line", "fillOpacity": 15 }
            }
          }
        },
        {
          "id": 2,
          "title": "2. HTTP 5xx Error Ratio (%)",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 8, "x": 8, "y": 0 },
          "datasource": { "type": "prometheus", "uid": "mimir-prod-uid" },
          "targets": [
            {
              "refId": "A",
              "expr": "service:http_requests_error_ratio:rate5m{sre_environment=\"$env\"}",
              "legendFormat": "{{service}}"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "percentunit",
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  { "color": "green", "value": null },
                  { "color": "yellow", "value": 0.01 },
                  { "color": "red", "value": 0.05 }
                ]
              }
            }
          }
        },
        {
          "id": 3,
          "title": "3. Request Duration p95 Latency",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 8, "x": 16, "y": 0 },
          "datasource": { "type": "prometheus", "uid": "mimir-prod-uid" },
          "targets": [
            {
              "refId": "A",
              "expr": "service:http_request_duration_seconds:p95_rate5m{sre_environment=\"$env\"}",
              "legendFormat": "{{service}} (p95)"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "s",
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  { "color": "green", "value": null },
                  { "color": "orange", "value": 0.25 },
                  { "color": "red", "value": 0.5 }
                ]
              }
            }
          }
        }
      ]
    }
```

---

## 3. Standar Kualitas & Rantai Bukti (Evidence) sebuah Dashboard

Membuat file JSON yang valid secara sintaks (`jq . file.json`) **bukan bukti** bahwa dashboard tersebut berfungsi di lingkungan operasional! Setiap panel wajib memenuhi 5 kriteria audit SRE:

1. **Datasource UID Eksplisit (`uid: "mimir-prod-uid"`):** Jangan biarkan field datasource kosong atau mengandalkan nama default yang bisa berubah antar cluster.
2. **Kesesuaian Satuan Unit (`unit`):**
   - Jika kueri menghasilkan rasio `0.0` hingga `1.0`, wajib gunakan unit **`percentunit`**. Jika kueri sudah dikalikan `100` (`0` hingga `100`), gunakan unit **`percent`**. Salah memilih di antara keduanya akan membuat error 5% tampil sebagai `0.05%` atau `500%`!
   - Jika metrik dasar dalam satuan detik (`_seconds`), gunakan unit **`s`** (Grafana otomatis mengubah `0.045 s` menjadi `45 ms` di layar!).
3. **Time Window & Zona Waktu Konsisten (`timezone: "utc"`):** Gunakan UTC atau zona waktu standar yang disepakati agar saat mencocokkan grafik dengan baris log server tidak terjadi kebingungan selisih jam.
4. **Bounded Variables:** Hindari membuat variabel template yang mengeksekusi kueri `label_values(http_requests_total, request_id)` saat halaman dibuka karena akan langsung membebani server Mimir.
5. **Editable = False:** Setel `"editable": false` pada JSON yang di-provision via GitOps untuk mengingatkan engineer bahwa perubahan wajib diajukan melalui Merge Request di Git.

---

## 4. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **ConfigMap Sudah Di-apply tapi Dashboard Tidak Muncul di Grafana** | 1. Lupa menambahkan label `grafana_dashboard: "1"` pada metadata ConfigMap.<br>2. Format JSON di dalam ConfigMap cacat (kurang koma atau kurung kurawal). | 1. Cek label ConfigMap: `kubectl get cm -n monitoring -l grafana_dashboard=1`.<br>2. Periksa log container `grafana-sc-dashboard` (sidecar) dan log container utama `grafana`. |
| **Panel Menampilkan `Datasource mimir-prod-uid was not found`** | Nilai `uid` pada definisi Datasource Grafana tidak cocok dengan `datasource.uid` yang tertulis di dalam file JSON dashboard. | Periksa `kubectl get cm/secret` provisioning datasource Grafana dan samakan string `uid`-nya secara presisi. |
| **Angka Persentase Error Terlihat 100x Lebih Besar (Misal Error 2% Tampil 200%)** | Rumus PromQL sudah dikalikan `100 * (...)`, namun unit panel di Grafana disetel ke **`percentunit`** (yang mengalikan 100 lagi!). | Hapus `100 *` dari kueri PromQL jika menggunakan `percentunit`, atau ubah unit panel menjadi `percent` (`0-100`). |

---

## 5. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. VALIDASI FILE JSON DASHBOARD SEBELUM COMMIT KE GITOPS (STATIC CHECK)
# ==============================================================================

# Memastikan struktur JSON valid dan tidak ada syntax error
jq empty service-red-overview.json && echo "[OK] Valid JSON"

# Memeriksa bahwa tidak ada kredensial/password/token yang tidak sengaja tertulis di JSON
grep -iE "(password|secret|bearer|token|apikey)" service-red-overview.json || echo "[OK] Clean"

# Mengekstrak dan memvalidasi JSON dari dalam file YAML ConfigMap Kubernetes
yq '.data["service-red-overview.json"]' dashboard-red-cm.yaml | jq empty

# ==============================================================================
# 2. VERIFIKASI PROVISIONING DASHBOARD VIA HTTP API GRAFANA (RUNTIME CHECK)
# ==============================================================================

# Mencari apakah dashboard dengan UID tertentu sudah berhasil ter-load di Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80
curl -s http://localhost:3000/api/search?query=RED | jq '.[] | {uid: .uid, title: .title, folderTitle: .folderTitle}'
```

---

## 6. Evaluasi Pemahaman Mandiri

### 6.1 Pilihan Ganda (3 Soal)

#### Soal 1
Manakah pasangan metodologi desain dashboard dan target pemantauannya yang **paling tepat** menurut standar industri SRE?
- A. **Metode USE** (Utilization, Saturation, Errors) untuk memantau komponen resource infrastruktur/perangkat keras (CPU, RAM, Disk, Network), sedangkan **Metode RED** (Rate, Errors, Duration) untuk memantau kinerja layanan aplikasi/microservices dari sudut pandang permintaan pengguna.
- B. Metode USE khusus untuk menghitung gaji karyawan, sedangkan Metode RED khusus untuk mengatur firewall.
- C. Metode RED hanya bisa digunakan untuk memantau suhu prosesor dan kecepatan kipas server.
- D. Metode USE dan RED tidak bisa digunakan di lingkungan Kubernetes on-premises.

#### Soal 2
Dalam **Metode USE**, metrik manakah yang paling tepat merepresentasikan dimensi **Saturation (Tingkat Kejenuhan / Antrean)** pada resource **CPU** dan **Memory** di server Linux?
- A. Persentase CPU idle dan jumlah kapasitas total hard disk.
- B. Panjang antrean proses yang menunggu giliran CPU (*CPU Run Queue / Load Average*) dan laju halaman memori yang dipindahkan ke disk (*Memory Swap In/Out rate*).
- C. Jumlah pod yang berstatus `Running`.
- D. Versi kernel Ubuntu yang sedang terpasang.

#### Soal 3
Apa yang terjadi jika sebuah kueri PromQL menghasilkan nilai rasio `0.03` (yang berarti 3% error), tetapi engineer mengatur konfigurasi unit panel di Grafana menjadi **`percent` (0-100)** alih-alih **`percentunit` (0.0-1.0)**?
- A. Grafana akan menampilkan angka **`0.03%`** (terlihat 100 kali lebih kecil dari kenyataan!), sehingga engineer mengira error sangat kecil padahal error sebenarnya sudah mencapai 3%!
- B. Grafana akan otomatis memperbaiki kueri PromQL tersebut.
- C. Server Mimir akan menolak menjalankan kueri tersebut.
- D. Layar monitor komputer akan berkedip merah.

---

### 6.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Saat terjadi insiden kelambatan sistem pada pukul 20:00 malam, seorang engineer membuka **Dashboard Node Infrastruktur (Metode USE)** dan melihat bahwa **CPU Utilization** pada seluruh worker node hanya berada di angka **45%**, dan **Memory Utilization** hanya **50%**. Ia langsung menyimpulkan di grup insiden: *"Semua layanan aplikasi aman dan tidak ada masalah performa karena CPU dan RAM server masih sangat lega!"*

Sementara itu, ratusan pengguna di aplikasi mobile sedang mengalami *Request Timeout (10 detik)* karena koneksi pool database di dalam aplikasi `order-service` macet total.

**Pertanyaan & Tugas Anda:**
1. Jelaskan mengapa kesimpulan engineer tersebut keliru! Mengapa grafik **Metode USE (CPU & RAM)** yang rendah tidak menjamin bahwa layanan aplikasi bebas dari masalah?
2. Jika engineer tersebut membuka **Dashboard Metode RED** milik `order-service`, apa yang akan terlihat pada ketiga grafik **Rate, Errors, dan Duration** saat koneksi database aplikasi macet?
3. Mengapa kita wajib menyimpan definisi dashboard tersebut sebagai `ConfigMap` di repositori GitOps (`editable: false`) daripada membiarkan anggota tim mengedit grafik secara bebas di browser?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **A.** Brendan Gregg merumuskan **USE** untuk komponen fisik/OS yang memiliki kapasitas terbatas (sehingga mengalami utilisasi dan antrean saturasi), sedangkan Tom Wilkie merumuskan **RED** untuk layanan berbasis *request-driven* (microservices/API).
2. **B.** *Utilization* mengukur persentase pemakaian kapasitas (`0–100%`), sedangkan *Saturation* mengukur dampak antrean ketika permintaan melebihi kapasitas sesaat (seperti *Load Average* / *Run Queue* pada CPU dan *Swap paging* pada RAM).
3. **A.** Kesalahan unit antara `percent` (`0-100`) dan `percentunit` (`0.0-1.0`) adalah salah satu bug visualisasi paling berbahaya di Grafana karena dapat menyembunyikan pelanggaran SLO sebesar 100 kali lipat!

#### Solusi Soal Kasus Nyata:
1. **Mengapa CPU/RAM Rendah Tidak Menjamin Aplikasi Sehat:**
   - Saat aplikasi `order-service` menunggu respons database yang mengalami *lock/deadlock* atau *connection pool exhaustion*, thread aplikasi hanya **diam menunggu I/O jaringan (blocking wait)** tanpa melakukan komputasi matematika.
   - Menunggu jaringan tidak memakan siklus CPU maupun RAM! Oleh karena itu, CPU Utilization justru terlihat turun/rendah padahal seluruh transaksi pengguna sedang macet total.
2. **Tampilan pada Dashboard Metode RED `order-service`:**
   - **Duration (`p95`/`p99` Latency):** Melonjak tajam hingga menyentuh batas timeout (10 detik).
   - **Errors (HTTP 5xx Ratio):** Melonjak tinggi (`504 Gateway Timeout` / `500 Internal Server Error`) begitu batas waktu tunggu habis.
   - **Rate (Request/s):** Mungkin awalnya melonjak karena pengguna menekan tombol *Retry*, lalu anjlok jika circuit breaker terbuka.
3. **Pentingnya Dashboard as Code via GitOps:**
   - Menjamin dashboard bersifat **immutable, terdokumentasi di Git, melalui proses Peer Review (PR/MR)** agar tidak ada kesalahan rumus/unit, serta otomatis ter-deploy ulang jika cluster direkonstruksi dari nol (*Disaster Recovery*).

---

## 7. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai desain dashboard USE/RED dan otomatisasi *Dashboard as Code*:
- Praktikkan di [LAB-01 — Dashboard, Datasource, dan Evidence](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.5-grafana-alerting-slo/lab/LAB-01-dashboard-data-source-evidence.md).
- Lanjutkan ke [02 — Alert Rule, Contact Point, Routing, dan Notification](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.5-grafana-alerting-slo/02-alerting-contact-point-routing-notification.md) untuk merancang arsitektur perutean peringatan terpadu dan penanganan kondisi `NoData`.
