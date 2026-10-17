# 02 — Tempo, OTLP, dan Correlation Metrics-Logs-Traces

> **Ringkasan:** Memahami konsep **Distributed Tracing** menggunakan **Grafana Tempo**, membedah anatomi **Trace, Span, dan W3C Context Propagation**, merancang strategi **Head vs Tail Sampling**, menguasai kueri **TraceQL**, serta menghubungkan ketiga pilar observabilitas (**Metrics ↔ Traces ↔ Logs**) melalui *Exemplars* dan *Derived Fields* untuk diagnosa insiden dalam hitungan detik.

---

## 1. Konsep & Arsitektur Distributed Tracing dengan Grafana Tempo

### 1.1 Analogi Dunia Nyata: "Nomor Resi Pelacakan Paket Ekspedisi Lintas Kota"
Bayangkan sebuah aplikasi e-commerce modern yang terdiri dari 6 microservice terpisah: *API Gateway ➔ Auth Service ➔ Cart Service ➔ Payment Service ➔ Bank Adapter ➔ Notification Service*.
Ketika seorang pelanggan mengeluh: *"Saat saya klik tombol Bayar, loading berputar selama 8 detik lalu gagal!"*, bagaimana cara Anda menemukan titik macetnya?
- **Jika Hanya Punya Metrik (Mimir):** Anda tahu rata-rata waktu bayar naik jadi 8 detik, tetapi tidak tahu layanan hilir mana yang menyebabkan antrean pada transaksi pelanggan tersebut.
- **Jika Hanya Punya Log (Loki):** Anda harus mencari di antara jutaan baris teks dari 6 aplikasi berbeda secara manual.
- **Dengan Distributed Tracing (Grafana Tempo):** Setiap transaksi yang masuk diberi satu **Nomor Resi Unik (`trace_id`)** yang dibawa terus dari satu layanan ke layanan berikutnya. Saat Anda memasukkan nomor resi tersebut ke Tempo, muncul **Diagram Air Terjun Waktu (*Waterfall Timeline*)** yang presisi:
  ```text
  [Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736] Total Durasi: 8.120 ms (STATUS: ERROR)
  ├─ api-gateway      [██░░░░░░░░░░░░░░░░░░░░░░░░░░░░]   120 ms (OK)
  │  ├─ auth-service  [█░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]    35 ms (OK)
  │  ├─ cart-service  [█░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]    45 ms (OK)
  │  └─ payment-api   [░░████████████████████████████] 7.920 ms (ERROR 504!)
  │     └─ PostgreSQL [░░░███████████████████████████] 7.890 ms (LOCK TIMEOUT pada tabel `ledger`)
  ```
  Dalam **3 detik**, Anda langsung tahu bahwa akar masalahnya bukan di jaringan atau kode API Gateway, melainkan *Database Lock Timeout* pada tabel `ledger` di PostgreSQL!

---

### 1.2 Anatomi Trace, Span, dan Propagasi Konteks (`traceparent`)

1. **Trace:** Perjalanan utuh satu permintaan (*request*) dari awal masuk gerbang hingga respons selesai dikirim. Memiliki identitas **`trace_id`** (32 karakter heksadesimal / 128-bit).
2. **Span:** Satu langkah pekerjaan spesifik di dalam sebuah Trace (misal: satu panggilan HTTP, satu eksekusi fungsi, atau satu kueri SQL). Setiap Span memiliki:
   - **`span_id`** (16 karakter heksadesimal / 64-bit).
   - **`parent_span_id`** (ID milik Span pemanggil di atasnya; jika kosong berarti ia adalah **Root Span**).
   - **Start Time & Duration** (presisi nanodetik/mikrodetik).
   - **Span Status** (`Unset`, `Ok`, atau `Error`).
   - **Resource & Span Attributes** (pasangan key-value seperti `service.name="payment-api"`, `http.status_code=500`, `db.system="postgresql"`).
3. **W3C Trace Context Propagation (`traceparent`):**
   Bagaimana layanan `payment-api` tahu bahwa ia dipanggil oleh `api-gateway` dalam transaksi yang sama? Melalui pengiriman HTTP Header standar **`traceparent`**:
   ```http
   GET /v1/charge HTTP/1.1
   Host: payment-api.production.svc.cluster.local
   traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
                └┤ └──────────────┬───────────────┘ └───────┬──────┘ └┤
               Ver            trace_id               parent_span_id  Flags (01=Sampled)
   ```

> [!WARNING]
> **Jebakan "Broken Trace" (Rantai Terputus):**
> Jika salah satu aplikasi di tengah alur membuat panggilan HTTP baru ke layanan lain tetapi lupa meneruskan (*propagate*) header `traceparent` dari request yang masuk, maka rantai pelacakan akan terputus menjadi dua trace terpisah! Pastikan seluruh HTTP client dan message broker (Kafka/RabbitMQ) diinstrumentasi dengan OpenTelemetry SDK.

---

## 2. Arsitektur Grafana Tempo, Sampling, dan TraceQL

### 2.1 Mengapa Grafana Tempo Sangat Efisien di Lingkungan On-Premises?
Sistem tracing generasi lama (seperti Jaeger + Elasticsearch) mengindeks setiap atribut span ke dalam database pencarian, sehingga membutuhkan puluhan node Elasticsearch berkapasitas RAM besar.
**Grafana Tempo** menggunakan pendekatan yang sejalan dengan Mimir dan Loki:
- **100% Berbasis Object Storage (MinIO S3):** Tempo menyimpan data span dalam format blok kolom **Apache Parquet** yang terkompresi tinggi di dalam bucket MinIO.
- **Pencarian Cepat dengan TraceQL:** Menggunakan struktur kolom Parquet di MinIO, Tempo dapat memindai atribut span (`duration`, `status`, `service.name`) menggunakan bahasa kueri **TraceQL** tanpa ketergantungan pada database indeks eksternal!

---

### 2.2 Strategi Sampling: Head Sampling vs Tail Sampling

Menyimpan 100% trace pada sistem dengan 50.000 transaksi/detik akan menghasilkan puluhan Terabyte data per hari. SRE mengendalikan volume dan biaya penyimpanan menggunakan **Sampling**:

| Jenis Sampling | Cara Kerja & Lokasi Keputusan | Kelebihan | Kekurangan |
|---|---|---|---|
| **1. Head Sampling (Probabilistic)** | Keputusan diambil **di awal (saat request baru masuk ke Root Span)** secara acak (misal: rekam hanya 1% atau 5% dari seluruh trafik). | Sangat ringan, tidak butuh memori besar di collector, mudah dikonfigurasi di SDK aplikasi. | **Bisa melewatkan insiden langka!** Jika ada 10 transaksi gagal di antara 100.000 transaksi sukses, peluang 1% bisa saja membuang ke-10 trace error tersebut! |
| **2. Tail Sampling (Policy-Based)** | Keputusan diambil **di akhir (di Collector/Alloy)** setelah seluruh span selesai terkumpul: **Simpan 100% trace yang mengalami `status=ERROR` atau `duration > 1s`, dan simpan hanya 1% trace yang sukses normal!** | **Tidak pernah kehilangan bukti error atau latensi lambat!** Sangat ideal untuk investigasi SRE. | Membutuhkan RAM lebih besar di tingkat Collector untuk menahan span sementara waktu hingga transaksi selesai. |

---

### 2.3 Contoh Kueri TraceQL Dasar di Grafana Tempo

```traceql
# 1. Cari seluruh trace di service "payment-api" yang berstatus ERROR
{ resource.service.name = "payment-api" && status = error }

# 2. Cari transaksi yang memakan waktu lebih dari 1 detik (1s) dan mengembalikan HTTP 5xx
{ resource.service.name = "order-api" && span.http.status_code >= 500 && duration > 1s }

# 3. Cari transaksi di mana "api-gateway" memanggil kueri database PostgreSQL yang lambat (> 500ms)
{ resource.service.name = "api-gateway" } >> { span.db.system = "postgresql" && duration > 500ms }
```

---

## 3. Segitiga Emas Korelasi SRE: Metrics ↔ Traces ↔ Logs

Kekuatan sesungguhnya dari tumpukan observabilitas Grafana (**Mimir + Loki + Tempo**) terletak pada kemampuannya menghubungkan ketiga sinyal tersebut secara otomatis.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│             SEGITIGA KORELASI OBSERVABILITAS (THE GOLDEN TRIANGLE)          │
│                                                                             │
│                     ┌───────────────────────────────┐                       │
│                     │   1. METRICS (Grafana Mimir)  │                       │
│                     │  "Ada lonjakan latensi p95!"  │                       │
│                     └───────┬───────────────▲───────┘                       │
│                             │               │                               │
│        (Klik titik Exemplar │               │ (Ekstrak Metrik RED dari      │
│         `trace_id` di grafik)               │  Span / LogQL Metric Query)   │
│                             ▼               │                               │
│     ┌───────────────────────────────┐       │       ┌───────────────────────┐
│     │   2. TRACES (Grafana Tempo)   │ ──────┼─────> │3. LOGS (Grafana Loki) │
│     │ "Waterfall: Span DB 4.8 detik"│ <─────┴────── │"Detail Error Baris 42"│
│     └───────────────────────────────┘  (Klik tombol └───────────────────────┘
│                                         "Logs for this span" / Derived Field)
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Cara Kerja Ketiga Jembatan Korelasi:
1. **Dari Metrics ke Traces (Melalui *Prometheus Exemplars*):**
   - Saat aplikasi mencatat durasi request ke dalam bucket Histogram (`http_request_duration_seconds_bucket`), SDK OpenTelemetry ikut melampirkan contoh `trace_id` dari request tersebut sebagai **Exemplar**.
   - Di grafik latensi Grafana, Exemplar tampil sebagai titik-titik berlian (*diamond dots*). Saat SRE mengklik titik yang melonjak tinggi di grafik, Grafana langsung membuka **Waterfall Trace** transaksi tersebut di Tempo!
2. **Dari Traces ke Logs (Melalui *Trace-to-Logs Mapping*):**
   - Setiap baris log JSON yang ditulis aplikasi wajib menyertakan field `"trace_id": "4bf92f35..."`.
   - Saat SRE mengklik satu Span yang error di Tempo, tombol **"Logs for this span"** otomatis mengeksekusi kueri LogQL ke Loki:
     `{namespace="production", service="payment-api"} |= "4bf92f3577b34da6a3ce929d0e0e4736"` pada jendela waktu span tersebut!
3. **Dari Logs ke Traces (Melalui *Loki Derived Fields*):**
   - Pada konfigurasi Datasource Loki di Grafana, SRE mendefinisikan regex penangkap `trace_id` (`"trace_id":"([a-f0-9]{32})"`). Setiap kali ada baris log error di Loki, string `trace_id` otomatis berubah menjadi **tombol link biru** yang jika diklik langsung membuka diagram Tempo!

---

## 4. Konfigurasi Produksi `tempo.yaml` & Datasource Korelasi

### 4.1 Konfigurasi `tempo.yaml` dengan MinIO S3 (Secret-Safe)

```yaml
multitenancy_enabled: true

server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: "0.0.0.0:4317"
        http:
          endpoint: "0.0.0.0:4318"

ingester:
  max_block_duration: 30m
  lifecycler:
    ring:
      replication_factor: 1 # Gunakan 1 di lab OrbStack, dan 3 di klaster produksi

storage:
  trace:
    backend: s3
    wal:
      path: /var/tempo/wal
    s3:
      bucket: "tempo-traces-prod"
      endpoint: "minio.storage.svc.cluster.local:9000"
      insecure: true
      # Disuntikkan dari K8s Secret dengan argumen -config.expand-env=true
      access_key: "${MINIO_ACCESS_KEY_ID}"
      secret_key: "${MINIO_SECRET_ACCESS_KEY}"

compactor:
  compaction:
    # Retensi penyimpanan trace selama 14 hari (336 jam) di MinIO
    block_retention: 336h

overrides:
  defaults:
    ingestion:
      max_traces_per_user: 10000
      max_bytes_per_trace: 5000000
```

---

### 4.2 Konfigurasi Grafana Datasource Provisioning untuk Korelasi Otomatis

Berikut adalah deklarasi `datasources.yaml` di Grafana yang mengikat **Mimir, Loki, dan Tempo** menjadi satu kesatuan:

```yaml
apiVersion: 1
datasources:
  # 1. Datasource Mimir (Metrics) dengan Exemplars menuju Tempo
  - name: Mimir-Production
    type: prometheus
    uid: mimir-prod-uid
    url: http://mimir-query-frontend.monitoring.svc.cluster.local:8080/prometheus
    jsonData:
      httpHeaderName1: "X-Scope-OrgID"
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo-prod-uid
    secureJsonData:
      httpHeaderValue1: "onprem-prod-tenant"

  # 2. Datasource Loki (Logs) dengan Derived Fields menuju Tempo
  - name: Loki-Production
    type: loki
    uid: loki-prod-uid
    url: http://loki-gateway.monitoring.svc.cluster.local:80
    jsonData:
      httpHeaderName1: "X-Scope-OrgID"
      derivedFields:
        - name: TraceID
          matcherRegex: '"trace_id":\s*"([a-fA-F0-9]{32})"'
          url: "$${__value.raw}"
          datasourceUid: tempo-prod-uid
    secureJsonData:
      httpHeaderValue1: "onprem-prod-tenant"

  # 3. Datasource Tempo (Traces) dengan Trace-to-Logs menuju Loki
  - name: Tempo-Production
    type: tempo
    uid: tempo-prod-uid
    url: http://tempo-query-frontend.monitoring.svc.cluster.local:3200
    jsonData:
      httpHeaderName1: "X-Scope-OrgID"
      tracesToLogsV2:
        datasourceUid: loki-prod-uid
        spanStartTimeShift: "-1m"
        spanEndTimeShift: "1m"
        filterByTraceID: true
        filterBySpanID: false
        tags:
          - key: "service.name"
            value: "service"
    secureJsonData:
      httpHeaderValue1: "onprem-prod-tenant"
```

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Tombol "Logs for this span" di Tempo Diklik tapi Hasil Loki Kosong** | 1. Nama label service di Tempo (`service.name="payment-api"`) berbeda dengan nama label stream di Loki (`app="payment"`).<br>2. Aplikasi belum menyertakan `trace_id` di dalam baris log JSON.<br>3. Jam sistem (*clock*) antar node tidak sinkron. | 1. Samakan pemetaan `tags` pada `tracesToLogsV2` agar sesuai dengan label stream Loki.<br>2. Pastikan logger aplikasi menginjeksi `trace_id` dari konteks OTLP aktif. |
| **Diagram Trace di Tempo Hanya Menampilkan 1 Span Tunggal (Tidak Menyambung ke Service Hilir)** | Layanan pemanggil tidak meneruskan (*propagate*) HTTP Header `traceparent` saat memanggil API layanan berikutnya. | Pasang *OpenTelemetry HTTP Client Instrumentation Interceptor* pada kode aplikasi agar header `traceparent` otomatis disisipkan di setiap outgoing request. |
| **Durasi Span Anak Terlihat Mendahului Span Induk (Negative Timestamp / Clock Skew)** | Server Node A dan Server Node B memiliki selisih waktu sistem (*system clock drift*) beberapa ratus milidetik. | Pastikan layanan sinkronisasi waktu **`chronyd` / NTP** berjalan aktif dan terhubung ke NTP server internal yang sama di seluruh node bare-metal/VM (`chronyc tracking`). |

---

## 6. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. VERIFIKASI KESEHATAN & PENERIMAAN SPAN DI GRAFANA TEMPO
# ==============================================================================

# Memeriksa status kesiapan layanan Tempo via HTTP API
kubectl port-forward -n monitoring svc/tempo 3200:3200
curl -s http://localhost:3200/ready

# Memeriksa jumlah span yang berhasil diterima oleh Tempo Distributor
curl -s http://localhost:3200/metrics | grep "tempo_distributor_spans_received_total"

# ==============================================================================
# 2. MENGAMBIL DATA TRACE & MENJALANKAN TRACEQL VIA HTTP API
# ==============================================================================

# Mengambil detail lengkap sebuah Trace berdasarkan Trace ID (32 hex)
curl -s -H "X-Scope-OrgID: onprem-prod-tenant" \
  http://localhost:3200/api/traces/4bf92f3577b34da6a3ce929d0e0e4736 | jq .

# Mengeksekusi pencarian TraceQL langsung dari terminal CLI
curl -sG -H "X-Scope-OrgID: onprem-prod-tenant" \
  http://localhost:3200/api/search \
  --data-urlencode 'q={ resource.service.name = "payment-api" && status = error }' | jq .
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Apa fungsi dari HTTP Header standar **`traceparent`** (W3C Trace Context) dalam arsitektur microservices yang terdistribusi?
- A. Menyimpan password database agar bisa dibaca oleh semua layanan.
- B. Meneruskan identitas `trace_id` dan `parent_span_id` dari satu layanan ke layanan berikutnya melalui jaringan, sehingga seluruh span dari berbagai server dapat dirangkai menjadi satu pohon perjalanan transaksi (*trace waterfall*) yang utuh.
- C. Menghentikan proses container jika pemakaian CPU melebihi 80%.
- D. Menggantikan fungsi protokol DNS di dalam cluster Kubernetes.

#### Soal 2
Mengapa strategi **Tail Sampling** (memutuskan penyimpanan di akhir transaksi) lebih disukai untuk kebutuhan investigasi insiden SRE dibandingkan **Head Sampling** (mengambil 1% secara acak di awal)?
- A. Karena Tail Sampling membuang seluruh transaksi yang mengalami error.
- B. Karena dengan Tail Sampling, sistem dapat dikonfigurasi untuk **menyimpan 100% transaksi yang mengalami error atau latensi tinggi**, sambil hanya menyimpan sebagian kecil (misal 1%) transaksi normal yang sehat, sehingga bukti insiden langka tidak pernah terbuang.
- C. Karena Tail Sampling tidak membutuhkan koneksi jaringan ke aplikasi.
- D. Karena Head Sampling wajib membayar lisensi ke cloud publik.

#### Soal 3
Fitur apakah pada Prometheus/Mimir yang memungkinkan titik-titik grafik Histogram menyimpan referensi `trace_id`, sehingga engineer dapat melompat langsung dari grafik lonjakan latensi ke tampilan span di Grafana Tempo?
- A. Alertmanager Inhibition
- B. Prometheus **Exemplars**
- C. Node Exporter Textfile Collector
- D. Linux Kernel OOM Killer

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Tim Anda telah mengonfigurasi Mimir, Loki, dan Tempo di cluster k3s produksi. Saat terjadi lonjakan latensi pada `checkout-service`, Anda membuka sebuah trace di Grafana Tempo dengan `trace_id = 7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d`.

Pada diagram *Waterfall* di Tempo, Anda melihat fenomena aneh:
1. Span milik `api-gateway` (di Node Worker-01) tercatat mulai pada pukul **`14:00:01.500`**.
2. Span anak milik `checkout-service` (di Node Worker-02) yang dipanggil oleh `api-gateway` justru tercatat mulai pada pukul **`14:00:00.200`** (1,3 detik **sebelum** induknya memanggil!).
3. Selain itu, saat Anda mengklik tombol **"Logs for this span"**, layar Loki terbuka tetapi menampilkan `No logs found`, padahal di terminal pod `checkout-service` lognya jelas-jelas ada.

**Pertanyaan & Tugas Anda:**
1. Apa penyebab teknis mengapa waktu mulai Span anak terlihat terjadi 1,3 detik lebih awal daripada Span induknya? Bagaimana cara memperbaikinya di tingkat sistem operasi node?
2. Sebutkan 2 penyebab paling umum mengapa fitur korelasi *"Logs for this span"* dari Tempo ke Loki gagal menemukan baris log meskipun lognya ada di Loki!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Tanpa propagasi header `traceparent` dari layanan hulu ke hilir, setiap layanan akan menganggap panggilan yang masuk sebagai transaksi baru dan membuat `trace_id` baru yang terpisah-pisah (*orphaned traces*).
2. **B.** Head sampling bersifat buta terhadap hasil akhir request karena keputusan diambil saat request baru tiba di gerbang depan. Tail sampling menunggu sebentar hingga status dan durasi diketahui, memastikan setiap anomali terekam 100%.
3. **B.** **Exemplars** adalah metadata tambahan (berisi pasangan label `trace_id="..."`) yang ditempelkan pada pengamatan bucket Histogram Prometheus/OpenTelemetry untuk menjembatani metrik agregat dengan contoh trace spesifik.

#### Solusi Soal Kasus Nyata:
1. **Penyebab Waktu Span Anak Mendahului Induk (Clock Skew / Time Drift):**
   - Setiap span diberi stempel waktu menggunakan jam lokal (*system clock*) dari masing-masing server tempat pod berjalan. Jika jam sistem pada Node Worker-02 terlambat 1,3 detik dibandingkan Node Worker-01 karena daemon **NTP / `chronyd`** mati atau tidak sinkron, maka grafik timeline gabungan di Tempo akan menjadi kacau.
   - **Solusi:** Pastikan layanan `chronyd` atau `systemd-timesyncd` aktif di semua node via Ansible playbook (`systemctl enable --now chronyd`) dan terhubung ke server NTP referensi yang sama.
2. **Dua Penyebab Gagalnya Korelasi "Logs for this span":**
   - **Ketidakcocokan Nama Label Service:** Konfigurasi `tracesToLogsV2` mencari label `{service="checkout-service"}` di Loki, padahal di Alloy label log tersebut disimpan dengan nama `{app="checkout-service"}` atau `{workload="checkout"}`.
   - **Absennya `trace_id` di Teks Log atau Selisih Jam:** Aplikasi belum mencetak string `trace_id` ke dalam baris log JSON, atau karena adanya *clock skew* antar node, rentang pencarian waktu sempit di Loki meleset dari waktu pencatatan log.

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai Grafana Loki, Grafana Tempo, dan korelasi segitiga observabilitas:
- Praktikkan secara langsung di [LAB-02 — Tempo Trace, Log, dan Metrics Correlation](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.4-loki-tempo/lab/LAB-02-tempo-trace-log-metrics-correlation.md).
- Lanjutkan ke modul pamungkas di Fase 7: [Modul 7.5 — Grafana USE/RED Dashboards, Unified Alerting, & SLO Burn Rate](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-7-observability/modul-7.5-grafana-alerting-slo/README.md).
