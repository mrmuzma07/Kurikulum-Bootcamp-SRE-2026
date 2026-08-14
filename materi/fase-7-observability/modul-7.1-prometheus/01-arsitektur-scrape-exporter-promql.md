# 01 — Arsitektur Scrape, Exporter, dan PromQL

## Tujuan

Memahami bagaimana Prometheus mengambil sample, memberi identitas series, dan mengubah counter/gauge/histogram menjadi query yang berguna untuk operasi on-prem.

## 1. Pull, Target, Job, dan Instance

Prometheus secara periodik melakukan HTTP scrape ke endpoint metrics. `job` mengelompokkan konfigurasi scrape; `instance` biasanya alamat target; labels lain menjelaskan cluster, environment, namespace, atau service. Target `UP` hanya berarti scrape request berhasil, bukan bahwa endpoint bisnis sehat.

```yaml
scrape_configs:
  - job_name: <approved-node-exporter-job>
    scrape_interval: <approved-interval>
    scrape_timeout: <approved-timeout>
    static_configs:
      - targets: [<approved-node-address>:<metrics-port>]
        labels:
          sre_environment: <approved-environment>
          sre_owner: <approved-owner>
```

Pada Kubernetes, service discovery menemukan Pod/Service/EndpointSlice. Relabeling harus allowlist dan bounded; jangan menyalin semua annotation atau raw URL menjadi label. Pastikan target discovery tidak memilih cluster production secara tidak sengaja.

## 2. TSDB dan Tipe Metric

- **Counter:** nilai hanya naik dan reset saat proses restart; gunakan `rate()`/`increase()`.
- **Gauge:** nilai dapat naik/turun, misalnya memory atau queue depth.
- **Histogram:** bucket counter untuk distribusi latency; `histogram_quantile()` menghitung estimasi percentile.
- **Summary:** quantile dihitung di client; sulit digabung lintas instance dan window.

TSDB lokal memiliki retention, block compaction, WAL, disk usage, dan failure domain sendiri. Retention panjang dan cardinality tinggi memerlukan kapasitas yang dihitung, bukan sekadar menaikkan flag.

## 3. Exporter On-Prem

`node_exporter` wajib dipahami untuk CPU, memory, filesystem, network, dan load host fisik/VM. `blackbox_exporter` melakukan probe HTTP/TCP/DNS dari lokasi observability, sehingga mengukur reachability dari sudut pandang tertentu. Exporter harus dibatasi dengan network policy, allowlist target, resource limits, dan auth/TLS reference bila diperlukan.

Jangan menulis credential exporter, basic-auth password, bearer token, atau TLS private key ke config yang di-commit. Gunakan secret mechanism yang disetujui dan hanya referensikan nama/ID non-secret.

## 4. PromQL dari Nol

Contoh query non-secret; sesuaikan label dan metric name melalui schema review:

```promql
# CPU utilization per instance, window bounded
100 * (1 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle", environment="<env>"}[5m])
))

# Request rate per service
sum by (service) (rate(http_requests_total{environment="<env>"}[5m]))

# Error ratio; denominator nol harus memiliki policy
sum(rate(http_requests_total{status=~"5..",environment="<env>"}[5m]))
/
sum(rate(http_requests_total{environment="<env>"}[5m]))

# p95 histogram; bucket label wajib tetap ada sampai aggregation
histogram_quantile(0.95,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket{environment="<env>"}[5m])
  )
)
```

`irate()` cocok untuk melihat perubahan sangat singkat tetapi lebih noisy; alert biasanya memakai `rate()` dengan window yang representatif. Selalu definisikan unit, time range, denominator, dan perilaku saat no-data/zero traffic.

## 5. Static Lane dan Runtime Lane

Static lane memeriksa schema, target allowlist, query, unit, cardinality, dan failure matrix. Optional runtime hanya pada disposable cluster: verifikasi context/namespace, approval, scrape target, query result, timestamp, dan redacted output. Render config atau target `UP` tidak membuktikan metric query atau SLO.

## Acceptance Criteria

- [ ] Perbedaan job/instance/target/series jelas.
- [ ] Tipe metric dan fungsi PromQL tepat.
- [ ] Exporter on-prem memiliki access, resource, dan secret boundary.
- [ ] Query memiliki window, aggregation, unit, dan no-data policy.

## Troubleshooting

Target `DOWN`: bedakan DNS, route/firewall, TLS/auth, timeout, exporter process, dan clock. Query kosong: cek time range, label matcher, tenant, ingestion delay, dan metric name; jangan langsung memperlebar label.

## Catatan SRE

Metrics adalah model pengukuran, bukan data mentah tanpa batas. Setiap label adalah dimensi biaya query dan storage. Mulai dari pertanyaan operasional, lalu pilih metric dan aggregation yang menjawabnya.

## Kaitan

Lanjutkan ke [Rules dan Alertmanager](02-rules-alertmanager-retention-cardinality.md) dan [Modul 7.2 Alloy](../modul-7.2-alloy-telemetry-pipeline/README.md).

## Status Runtime

Contoh dan query adalah static design. Prometheus install dan scrape runtime **belum diverifikasi** tanpa evidence aktual.
