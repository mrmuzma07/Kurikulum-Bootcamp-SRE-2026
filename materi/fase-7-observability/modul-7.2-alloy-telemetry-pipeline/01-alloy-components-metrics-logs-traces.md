# 01 — Alloy Components untuk Metrics, Logs, dan Traces

> **Ringkasan:** Memahami **Grafana Alloy** sebagai agen pengumpul telemetri terpadu (*unified OpenTelemetry collector*) untuk ketiga pilar observabilitas (**Metrics, Logs, dan Traces**), merancang *component graph* menggunakan sintaks deklaratif Alloy, menerapkan penyaringan rahasia (*PII/credential redaction*) sebelum data meninggalkan node, serta mengelola *backpressure* agar agen tidak menghabiskan memori server.

---

## 1. Konsep & Arsitektur Grafana Alloy

### 1.1 Analogi Dunia Nyata: "Instalasi Pengolahan & Pemurnian Air Terpadu"
Di masa lalu, untuk memantau satu server Linux atau node Kubernetes, tim infrastruktur harus memasang **3 agen berbeda sekaligus**:
1. *Prometheus Node/Agent* untuk mengambil metrik angka.
2. *Promtail / Fluentbit* untuk menyedot baris log teks.
3. *Jaeger Agent / OpenTelemetry Collector* untuk menangkap jejak transaksi (*traces*).

Memasang 3 agen terpisah ibarat membangun 3 pipa saluran air dengan 3 pompa listrik berbeda di setiap rumah—boros CPU, boros RAM, dan sulit diperbarui.

**Grafana Alloy** hadir sebagai **Instalasi Pengolahan & Pemurnian Air Terpadu** di setiap node:
- **Pipa Masuk (*Receivers / Sources*):** Menerima semua jenis aliran data mentah (metrik Prometheus, file log container `/var/log/pods`, dan paket OpenTelemetry/OTLP).
- **Saringan Pemurnian (*Processors / Relabeling / Redaction*):** Menyaring kotoran dan racun berbahaya (menghapus password, token `Authorization`, nomor kartu kredit/PII, dan membuang label sampah ber-kardinalitas tinggi) **sebelum** air dialirkan ke luar gedung.
- **Tangki Penampung Darurat (*Batch, Queue & Write-Ahead Log*):** Jika waduk pusat sedang sibuk atau jaringan terputus sementara, air ditampung di tangki lokal berkapasitas terukur (bukan membanjiri rumah) lalu dikirim ulang secara bertahap.
- **Pipa Distribusi Akhir (*Exporters / Writers*):** Mengirimkan air yang sudah bersih ke tiga waduk penyimpanan khusus: **Mimir** (untuk Metrics), **Loki** (untuk Logs), dan **Tempo** (untuk Traces).

---

### 1.2 Arsitektur Grafik Komponen (Component Graph) di Alloy

Grafana Alloy menggunakan bahasa konfigurasi deklaratif berbentuk ekspresi (sebelumnya dikenal sebagai sintaks *River*). Setiap blok fungsi disebut **Component**, dan output dari satu komponen dihubungkan ke input komponen berikutnya membentuk **Directed Acyclic Graph (DAG)**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                GRAFIK PIPELINE TELEMETRI GRAFANA ALLOY                      │
│                                                                             │
│  [JALUR 1: METRICS PIPELINE]                                                │
│  discovery.kubernetes ──> discovery.relabel ──> prometheus.scrape           │
│                                                        │                    │
│  [ Backend: MIMIR ] <── prometheus.remote_write <── prometheus.relabel      │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  [JALUR 2: LOGS PIPELINE]                                                   │
│  discovery.kubernetes ──> loki.source.kubernetes ──> loki.process           │
│                                                      (Redaksi PII & Label)  │
│  [ Backend: LOKI ]  <────────────────── loki.write <─────────┘              │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  [JALUR 3: TRACES PIPELINE (OpenTelemetry / OTLP)]                          │
│  otelcol.receiver.otlp (gRPC :4317 / HTTP :4318)                            │
│           │                                                                 │
│           ▼                                                                 │
│  otelcol.processor.memory_limiter (Rem Darurat Pelindung RAM)               │
│           │                                                                 │
│           ▼                                                                 │
│  otelcol.processor.attributes (Hapus Header Rahasia / Token)                │
│           │                                                                 │
│           ▼                                                                 │
│  otelcol.processor.batch ──> otelcol.exporter.otlp ──> [ Backend: TEMPO ]   │
└─────────────────────────────────────────────────────────────────────────────┘
```

> [!IMPORTANT]
> **Batas Tanggung Jawab (*Boundary of Ownership*):**
> Grafana Alloy adalah **Collector & Forwarder**, bukan tempat penyimpanan jangka panjang atau mesin penentu SLO! Status pod Alloy `Running` atau `Ready` hanya membuktikan bahwa proses agen menyala dan konfigurasi sintaksnya valid, **bukan bukti** bahwa Mimir, Loki, atau Tempo berhasil menerima dan menyimpan data telemetri Anda.

---

## 2. Konfigurasi Pipeline 3 Pilar di Alloy (`config.alloy`)

Mari kita bedah implementasi konfigurasi produksi untuk masing-masing pilar telemetri dengan mematuhi prinsip **Bounded Cardinality** dan **Zero Secret Leakage**.

### 2.1 Jalur 1: Pipeline Metrics (Scrape -> Relabel -> Remote-Write ke Mimir)

Berbeda dengan server Prometheus monolitik yang menyimpan data di disk lokal selama berminggu-minggu, Alloy bertindak sebagai agen ringan yang mengambil metrik lalu langsung meneruskannya (*remote_write*) ke penyimpanan terpusat **Grafana Mimir**.

```hcl
// 1. Temukan seluruh Pod di namespace yang diizinkan (Allowlist)
discovery.kubernetes "prod_pods" {
  role = "pod"
  namespaces {
    names = ["production", "staging"]
  }
}

// 2. Saring target hanya yang memiliki anotasi prometheus.io/scrape = "true"
discovery.relabel "filtered_pods" {
  targets = discovery.kubernetes.prod_pods.targets

  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_scrape"]
    action        = "keep"
    regex         = "true"
  }

  // Petakan hanya label yang aman dan bounded (Allowlist)
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
    target_label  = "service"
  }
}

// 3. Lakukan HTTP Scrape ke target yang lolos seleksi
prometheus.scrape "app_metrics" {
  targets         = discovery.relabel.filtered_pods.output
  scrape_interval = "15s"
  scrape_timeout  = "10s"
  forward_to      = [prometheus.relabel.sanitize_metrics.receiver]
}

// 4. Buang metrik sampah atau label ber-kardinalitas tinggi sebelum dikirim!
prometheus.relabel "sanitize_metrics" {
  forward_to = [prometheus.remote_write.mimir_prod.receiver]

  // Buang label berbahaya jika tidak sengaja diekspos oleh aplikasi
  rule {
    regex  = "(user_id|request_id|session_token|client_ip)"
    action = "labeldrop"
  }
}

// 5. Kirim ke Mimir dengan antrean WAL dan referensi Secret yang aman
prometheus.remote_write "mimir_prod" {
  endpoint {
    url = "http://mimir-distributor.monitoring.svc.cluster.local:8080/api/v1/push"

    headers = {
      "X-Scope-OrgID" = "onprem-prod-tenant",
    }

    // Konfigurasi antrean dan batas percobaan ulang (Backpressure Control)
    queue_config {
      capacity             = 10000
      max_shards           = 10
      min_backoff          = "100ms"
      max_backoff          = "30s"
      retry_on_http_429    = true
    }
  }

  external_labels = {
    cluster = "k3s-onprem-prod-01",
  }
}
```

---

### 2.2 Jalur 2: Pipeline Logs (Tail -> Parse -> Redact PII/Secret -> Kirim ke Loki)

Kesalahan paling fatal pada pengelolaan log adalah:
1. Menjadikan semua field JSON (seperti `trace_id`, `user_email`, `http_url`) sebagai **Stream Label Loki** (yang akan menghancurkan indeks Loki).
2. Membiarkan password, token `Bearer`, atau nomor identitas nasabah tersimpan mentah di dalam teks log.

Berikut adalah konfigurasi `loki.process` di Alloy yang melakukan penyensoran otomatis (*redaction*) sebelum log meninggalkan node:

```hcl
// 1. Ambil aliran log dari Pod Kubernetes yang ditemukan
loki.source.kubernetes "pod_logs" {
  targets    = discovery.relabel.filtered_pods.output
  forward_to = [loki.process.scrub_and_label.receiver]
}

// 2. Proses, Ekstrak Label Terbatas, dan Sensor Data Rahasia (Redaction)
loki.process "scrub_and_label" {
  forward_to = [loki.write.loki_prod.receiver]

  // Tahap A: Parse format log container CRI (Docker/containerd)
  stage.cri {}

  // Tahap B: Parse log aplikasi berformat JSON (tanpa menjadikannya label!)
  stage.json {
    expressions = {
      log_level = "level",
      component = "component",
    }
  }

  // Tahap C: Jadikan HANYA field ber-kardinalitas sangat rendah sebagai label
  stage.labels {
    values = {
      level     = "log_level",
      component = "component",
    }
  }

  // Tahap D: REDAKSI WAJIB! Sensor Bearer Token, Password, dan API Key
  stage.replace {
    expression = "(?i)(authorization|bearer|password|api_key|secret|token)([\"'\\s:=]+)([^\"'\\s&]+)"
    replace    = "***REDACTED_SECRET***"
  }

  // Tahap E: Sensor alamat email (PII Redaction)
  stage.replace {
    expression = "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
    replace    = "[REDACTED_EMAIL]"
  }
}

// 3. Kirim log yang sudah bersih ke server Loki
loki.write "loki_prod" {
  endpoint {
    url       = "http://loki-gateway.monitoring.svc.cluster.local:80/loki/api/v1/push"
    tenant_id = "onprem-prod-tenant"
  }
}
```

---

### 2.3 Jalur 3: Pipeline Traces (OTLP Receiver -> Memory Limiter -> Sanitize -> Tempo)

Untuk menangkap jejak perjalanan transaksi lintas microservice (*Distributed Tracing*), aplikasi mengirimkan span menggunakan protokol standar **OpenTelemetry Protocol (OTLP)** ke Alloy.

```hcl
// 1. Terima data trace (dan metrik/log OTLP) dari aplikasi di port standar
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }
  http {
    endpoint = "0.0.0.0:4318"
  }

  output {
    traces = [otelcol.processor.memory_limiter.guard.input]
  }
}

// 2. REM DARURAT RAM: Cegah Alloy mengalami OOMKilled saat terjadi banjir trace!
otelcol.processor.memory_limiter "guard" {
  check_interval         = "1s"
  limit_percentage       = 75
  spike_limit_percentage = 20

  output {
    traces = [otelcol.processor.attributes.scrub_sensitive.input]
  }
}

// 3. Hapus atribut HTTP Header atau kredensial sensitif dari Span Trace
otelcol.processor.attributes "scrub_sensitive" {
  action {
    key    = "http.request.header.authorization"
    action = "delete"
  }
  action {
    key    = "http.request.header.cookie"
    action = "delete"
  }
  action {
    key    = "db.connection_string"
    action = "delete"
  }

  output {
    traces = [otelcol.processor.batch.optimize.input]
  }
}

// 4. Kumpulkan span dalam paket (Batch) agar hemat bandwidth jaringan
otelcol.processor.batch "optimize" {
  send_batch_size     = 1024
  send_batch_max_size = 2048
  timeout             = "2s"

  output {
    traces = [otelcol.exporter.otlp.tempo_prod.input]
  }
}

// 5. Ekspor ke backend Grafana Tempo
otelcol.exporter.otlp "tempo_prod" {
  client {
    endpoint = "tempo-distributor.monitoring.svc.cluster.local:4317"
    tls {
      insecure = true // Hanya untuk jaringan internal cluster tertutup tanpa TLS
    }
    headers = {
      "X-Scope-OrgID" = "onprem-prod-tenant",
    }
  }
}
```

---

## 3. Pengendalian Backpressure, Antrean, dan Batas Resource

Apa yang terjadi jika server backend penyimpanan (**Mimir, Loki, atau Tempo**) sedang mengalami gangguan atau direstart selama 10 menit?
- Jika agen pengumpul (Alloy) terus menampung seluruh metrik, log, dan trace di dalam memori RAM tanpa batas (*unbounded retry queue*), maka dalam beberapa menit pod Alloy di seluruh node cluster akan kehabisan RAM, mengalami `OOMKilled`, atau membuat node Kubernetes kekurangan memori!

### 3.1 Strategi Pertahanan Berlapis di Alloy
1. **Bounded Queue & Exponential Backoff:**
   - Setiap exporter memiliki batas antrean (`queue_capacity`). Saat antrean penuh karena backend mati terlalu lama, Alloy akan **membuang data paling lama/baru (*drop telemetry*)** dan menaikkan metrik internal `alloy_queue_dropped_total` daripada mematikan server!
2. **Memory Limiter Processor (`otelcol.processor.memory_limiter`):**
   - Wajib dipasang sebagai komponen **pertama** setelah `otelcol.receiver`. Jika pemakaian RAM pod Alloy menyentuh 75% dari batas `resources.limits.memory`, komponen ini langsung menolak trafik OTLP sementara (mengirim kode retry ke aplikasi) dan memicu *Garbage Collection* Go.
3. **Sampling Kebijakan Trace (Head vs Tail Sampling):**
   - Menyimpan 100% trace dari ribuan request per detik sangat mahal. Gunakan *Head Sampling* (misal ambil 5%–10% trace normal secara acak) atau *Tail Sampling* (simpan 100% trace yang mengalami error HTTP 5xx atau latensi tinggi, dan ambil 1% untuk request yang sukses cepat).

---

## 4. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala di Lapangan | Kemungkinan Penyebab | Langkah Investigasi & Solusi SRE |
|---|---|---|
| **Pod Alloy `Running` tapi Mimir/Loki/Tempo Kosong (0 Data)** | 1. Filter `discovery.relabel` terlalu ketat (semua target ter-drop).<br>2. Kesalahan URL endpoint atau header `X-Scope-OrgID`.<br>3. NetworkPolicy memblokir trafik keluar ke namespace `monitoring`. | 1. Buka UI internal Alloy (`kubectl port-forward pod/<alloy> 12345:12345`) dan cek komponen mana yang memiliki `0 exports`.<br>2. Periksa metrik internal Alloy: `prometheus_remote_storage_samples_failed_total` dan `loki_write_dropped_bytes_total`. |
| **Pod Alloy Restart Berulang (`OOMKilled` / Exit 137)** | 1. Tidak memasang `otelcol.processor.memory_limiter`.<br>2. Ukuran `queue_config.capacity` terlalu besar dibandingkan limit RAM pod.<br>3. Aplikasi memuntahkan baris log raksasa (misal stack trace 10 MB/detik). | 1. Pasang `memory_limiter` di urutan pertama setelah receiver.<br>2. Kecilkan `max_shards` dan `capacity` pada `remote_write`.<br>3. Batasi ukuran baris log maksimal (`max_line_size = "256KiB"`). |
| **Data Sensitif (Token/Password) Ditemukan di Loki** | 1. Aturan `stage.replace` regex salah atau diletakkan **setelah** komponen write.<br>2. Developer mencetak header HTTP mentah dengan nama key yang tidak umum. | 1. Segera rotasi/cabut (*revoke*) token/password yang bocor tersebut!<br>2. Perbaiki regex `stage.replace` di `loki.process` dan uji dengan contoh teks log sebelum di-deploy. |

---

## 5. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. VALIDASI & FORMAT KONFIGURASI ALLOY SECARA LOKAL (STATIC CHECK)
# ==============================================================================

# Merapikan format penulisan file config.alloy (seperti tofu fmt)
alloy fmt config.alloy

# Memvalidasi sintaks dan grafik hubungan antar komponen tanpa menjalankan server
alloy fmt --test config.alloy

# ==============================================================================
# 2. INSPEKSI & DEBUGGING RUNTIME ALLOY DI KUBERNETES
# ==============================================================================

# Membuka antarmuka Web UI Graph Inspector Alloy di browser lokal (http://localhost:12345)
kubectl port-forward -n monitoring daemonset/alloy 12345:12345

# Memeriksa kesehatan komponen internal Alloy via HTTP API
curl -s http://localhost:12345/api/v0/web/components | jq '.[] | {id: .local_id, health: .health.state}'

# Memeriksa apakah ada sampel metrik atau log yang gagal dikirim (Dropped/Failed)
curl -s http://localhost:12345/metrics | grep -E "(prometheus_remote_storage_samples_failed_total|loki_write_dropped|otelcol_exporter_send_failed)"

# Memicu reload konfigurasi Alloy secara langsung setelah ConfigMap diperbarui
curl -X POST http://localhost:12345/-/reload
```

---

## 6. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Apa keuntungan arsitektural utama menggunakan **Grafana Alloy** dibandingkan menjalankan Promtail, Prometheus Agent, dan OpenTelemetry Collector secara terpisah di setiap node Kubernetes?
- A. Alloy dapat menggantikan fungsi database PostgreSQL dan Redis secara bersamaan.
- B. Alloy menyatukan pengumpulan ketiga sinyal telemetri (**Metrics, Logs, dan Traces**) ke dalam satu proses biner tunggal dengan *component graph* terpadu, sehingga menghemat overhead CPU/RAM node dan menyeragamkan mekanisme penyaringan keamanan (redaksi PII/secret).
- C. Alloy tidak membutuhkan ruang memori RAM sama sekali saat berjalan.
- D. Alloy secara otomatis menulis kode aplikasi untuk tim developer.

#### Soal 2
Pada komponen `loki.process` di Grafana Alloy, seorang engineer mengekstrak field dari log JSON: `user_id`, `trace_id`, `request_duration_ms`, dan `level`. Field manakah yang **BOLEH** dipromosikan ke dalam blok `stage.labels` sebagai label stream Loki?
- A. `user_id` dan `trace_id` agar mudah dicari per pengguna.
- B. `request_duration_ms` karena angkanya selalu berubah setiap milidetik.
- C. Hanya `level` (misal: `info`, `warn`, `error`) karena memiliki kardinalitas sangat rendah dan terbatas (*bounded*), sedangkan `user_id` dan `trace_id` dibiarkan tetap di dalam isi teks log untuk difilter pada saat kueri (*query-time filtering*).
- D. Seluruh field JSON wajib dimasukkan ke dalam `stage.labels`.

#### Soal 3
Mengapa komponen `otelcol.processor.memory_limiter` wajib diletakkan tepat setelah `otelcol.receiver.otlp` di dalam pipeline traces?
- A. Untuk mengenkripsi paket jaringan menggunakan protokol SSH.
- B. Sebagai mekanisme *backpressure* dan pelindung memori; jika pemakaian RAM Alloy mendekati batas container limit akibat lonjakan trafik atau kemacetan di backend Tempo, ia akan menahan paket baru dan memicu *garbage collection* agar pod Alloy tidak mati terkena `OOMKilled`.
- C. Untuk menggandakan jumlah span trace yang dikirim ke Tempo hingga 10 kali lipat.
- D. Untuk menghapus seluruh label `service.name` dari data trace.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Tim Keamanan Informasi (*Infosec*) melakukan audit terhadap sistem observabilitas cluster k3s on-premises Anda. Mereka menemukan bahwa ketika pengguna gagal melakukan login di `auth-service`, aplikasi mencatat baris log berikut ke `stdout`:
`2026-10-17T10:15:00Z ERROR [auth] Login failed for email=budi.santoso@gmail.com with token=eyJhbGciOiJIUzI1NiIsInR5c...`
dan pada span OpenTelemetry tercatat atribut HTTP Header:
`http.request.header.authorization = "Bearer eyJhbGciOiJIUzI1NiIsInR5c..."`

Kedua data sensitif tersebut saat ini terkirim mentah ke server **Loki** dan **Tempo**.

**Pertanyaan & Tugas Anda:**
1. Mengapa melakukan pembersihan (*redaction*) di tingkat **Grafana Alloy (sebelum data dikirim ke jaringan)** jauh lebih baik daripada baru menyembunyikannya di tampilan dashboard Grafana?
2. Komponen dan tahapan (*stage*) apa yang harus Anda tambahkan pada jalur **Logs** di `config.alloy` untuk menyensor alamat email dan token tersebut?
3. Komponen dan aksi (*action*) apa yang harus Anda tambahkan pada jalur **Traces** di `config.alloy` untuk menghapus header `authorization` tersebut?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Konsolidasi agen mengurangi jejak pemakaian resource (*footprint*) pada setiap worker node, menyederhanakan siklus upgrade Helm chart/DaemonSet, serta memastikan aturan keamanan dan discovery Kubernetes hanya perlu dikonfigurasi satu kali.
2. **C.** Loki mengindeks kombinasi label menjadi *log streams*. Memasukkan `trace_id` atau `user_id` ke dalam `stage.labels` akan menciptakan jutaan *stream* kecil yang melumpuhkan performa penyimpanan dan kueri Loki.
3. **B.** Tanpa `memory_limiter`, lonjakan span trace saat terjadi serangan DDoS atau *retry storm* akan memenuhi buffer memori Alloy hingga kernel Linux membunuh prosesnya (`OOMKilled`).

#### Solusi Soal Kasus Nyata:
1. **Mengapa Redaksi Wajib di Tingkat Collector (Alloy):**
   - Jika data rahasia baru disembunyikan di layar Grafana, data asli tetap tersimpan secara permanen di dalam disk object storage (MinIO/S3) milik Loki dan Tempo. Siapa pun yang memiliki akses API query langsung ke Loki/Tempo atau akses ke backup disk dapat mencuri JWT token aktif dan melanggar kepatuhan privasi data (UU PDP / GDPR). Dengan menyensor di Alloy, data rahasia dimusnahkan di memori lokal node sebelum pernah menyentuh jaringan atau disk penyimpanan!
2. **Solusi di Jalur Logs (`loki.process`):**
   - Tambahkan blok `stage.replace` menggunakan ekspresi reguler (Regex) untuk menimpa pola `token=...` dan pola alamat email menjadi teks `***REDACTED***` sebelum diteruskan ke `loki.write`.
3. **Solusi di Jalur Traces (`otelcol.processor.attributes`):**
   - Tambahkan komponen `otelcol.processor.attributes` di antara receiver dan batch processor dengan aturan:
     ```hcl
     action {
       key    = "http.request.header.authorization"
       action = "delete"
     }
     ```

---

## 7. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai cara merancang komponen internal Grafana Alloy untuk ketiga sinyal telemetri:
- Lanjutkan ke [02 — OTel Pipeline, DaemonSet, Helm, dan Debugging](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.2-alloy-telemetry-pipeline/02-otel-pipeline-daemonset-helm-debugging.md) untuk mempelajari arsitektur deployment Alloy sebagai Kubernetes DaemonSet yang aman, kompatibilitas multi-arsitektur (ARM64/AMD64), serta teknik debugging tanpa membocorkan rahasia.
- Praktikkan di [LAB-01 — Alloy Metrics Remote-Write](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.2-alloy-telemetry-pipeline/lab/LAB-01-alloy-metrics-remote-write.md).
