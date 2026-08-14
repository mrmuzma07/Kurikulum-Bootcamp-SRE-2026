# 02 — Tempo, OTLP, dan Correlation Metrics-Logs-Traces

## Tujuan

Memahami trace ingest dan correlation yang membantu diagnosis tanpa menjadikan trace payload, attribute, atau correlation ID sebagai sumber kebocoran.

## 1. OTLP dan Trace Model

Trace terdiri dari spans dengan `trace_id`, `span_id`, parent/child, timestamps, status, service/resource attributes, events, dan links. OTLP transport memerlukan protocol, endpoint, timeout, retry, batch, compression, auth/TLS reference, and sampling policy.

`service.name`, bounded environment/namespace, route template, and version membantu query. Jangan masukkan authorization header, raw SQL credential, password, PII, atau arbitrary request content sebagai attribute.

## 2. Sampling dan Storage

Head sampling murah tetapi bisa membuang trace penting sebelum tahu error. Tail sampling lebih informatif namun memerlukan memory/latency/capacity. Tetapkan retention, object storage, compaction, max trace size, and drop policy. Trace volume bukan gratis; monitor collector/backend queue.

## 3. Correlation

Instrumented app meneruskan W3C Trace Context secara benar. Structured log dapat menyertakan trace ID yang telah ditentukan aman, lalu Grafana menghubungkan:

```text
RED metric spike → service dashboard → trace exemplar/trace ID reference
→ Tempo span timeline → Loki log stream around same trace/time window
```

Correlation hanya valid jika clock, service name, propagation, sampling, dan time window konsisten. Trace tersedia tidak berarti semua request ter-trace atau SLO baik.

## 4. Static/Runtime Evidence

Static review memeriksa schema, attribute allowlist, sampling, redaction, retention, and links. Runtime harus membuktikan OTLP accepted, trace query returns bounded result, log contains safe trace reference, and Grafana correlation works; payload tetap redacted.

## Acceptance Criteria

- [ ] OTLP contract dan trace attributes secret-safe.
- [ ] Sampling/retention/capacity mempunyai alasan dan policy.
- [ ] Propagation dan clock correlation dijelaskan.
- [ ] Trace/log/metric evidence dipisahkan dari SLO.

## Troubleshooting dan Catatan SRE

Trace tidak ada: cek instrumentation, exporter, protocol, queue, sampling, tenant, and backend. Parent hilang: cek propagation/context boundary. Timestamp aneh: cek clock sync. Jangan menaikkan sampling global tanpa capacity review.

## Kaitan

Praktikkan [LAB-02](lab/LAB-02-tempo-trace-log-metrics-correlation.md) dan gunakan hasilnya di [Grafana](../modul-7.5-grafana-alerting-slo/README.md).

## Status Runtime

Tempo ingest, trace query, dan cross-signal correlation **belum diverifikasi** tanpa evidence.
