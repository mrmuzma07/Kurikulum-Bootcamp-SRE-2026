# 01 — Alloy Components untuk Metrics, Logs, dan Traces

## Tujuan

Memahami Alloy sebagai graph collector/processor/exporter untuk tiga pilar tanpa mencampur tanggung jawab collector dengan storage, query, dashboard, atau SLO.

## 1. Component Graph

Model pipeline sebagai:

```text
source → discovery/receiver → processor/redaction → batch/queue/retry
→ exporter → backend → health/telemetry collector
```

Setiap edge memiliki protocol, auth reference, timeout, retry, queue capacity, dan failure metric. Component health tidak otomatis berarti backend menerima data.

## 2. Metrics

Alloy dapat scrape endpoint Prometheus-format lalu remote-write ke Mimir. Discovery harus allowlist namespace/label dan mencegah target production yang tidak disetujui. Drop atau relabel metric yang tidak dibutuhkan sebelum forwarding untuk mengendalikan cardinality.

```text
prometheus.scrape(<approved_targets>)
  → prometheus.relabel(<bounded_labels>)
  → prometheus.remote_write(<approved_mimir_reference>)
```

Endpoint, tenant, credential, dan TLS key hanya placeholder/reference secret mechanism.

## 3. Logs

Tail source yang disetujui, parse structured fields, redaksi `authorization`, `cookie`, password, token, dan PII, lalu kirim ke Loki. Jangan membuat setiap parsed field menjadi label. Label stream sebaiknya bounded seperti cluster, namespace, workload, dan severity; field lain tetap payload/query-time bila aman.

## 4. Traces

OTLP receiver menerima spans dari instrumented app. Processor dapat melakukan batch, sampling, attribute filtering, dan resource normalization sebelum export ke Tempo. Sampling memengaruhi representativeness; tail sampling membutuhkan memory/latency policy. Propagation harus mempertahankan trace context tanpa menyimpan header credential.

## 5. Failure dan Resource Boundary

Backend unavailable: queue terbatas, retry exponential dengan max duration, dropped telemetry counter, alert on pipeline degradation, dan runbook. Jangan mengizinkan retry tak terbatas yang menghabiskan memory. DaemonSet membutuhkan CPU/memory limits, toleration/priority review, host path scope, ServiceAccount/RBAC minimal, dan network policy.

## Static/Runtime

Static lane menguji graph, labels, redaction, queue, endpoint references, dan capacity. Runtime lane membuktikan collector loaded, source discovered, samples/logs/spans forwarded, backend queryable, dan drop/retry evidence. Helm render/config parse saja **belum memverifikasi** forwarding.

## Acceptance Criteria

- [ ] Metrics, logs, traces memiliki pipeline dan backend berbeda yang jelas.
- [ ] PII/credential redaction dilakukan sebelum storage/notification.
- [ ] Queue/retry/backpressure/drop policy terukur.
- [ ] RBAC, host access, resource, dan network boundary dibatasi.

## Troubleshooting dan Catatan SRE

Tidak ada sample: cek discovery, scrape status, relabel drop, queue, tenant, network, dan backend ingestion. Log hilang: cek file permission/rotation/parser/drop. Trace tidak berkorelasi: cek propagation, sampling, clock, dan service name.

## Kaitan

Lanjutkan ke [OTel/DaemonSet/Helm](02-otel-pipeline-daemonset-helm-debugging.md) dan [Modul 7.3 Mimir](../modul-7.3-mimir/README.md).

## Status Runtime

Alloy deployment dan data forwarding **belum diverifikasi** tanpa evidence runtime.
