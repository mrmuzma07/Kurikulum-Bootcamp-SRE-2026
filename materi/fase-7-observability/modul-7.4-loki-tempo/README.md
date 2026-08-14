# Modul 7.4 — Logs & Traces: Loki + Tempo

> **Tujuan:** mengumpulkan logs dan traces dengan label/attribute yang aman, queryable, ter-retain, dan dapat dikorelasikan dengan metrics.

## Capaian

- [ ] Mendesain Loki stream label low-cardinality dan query LogQL dasar.
- [ ] Memisahkan parsing log dari indexing dan memahami volume/query cost.
- [ ] Menjelaskan Tempo OTLP ingest, span/resource attributes, sampling, dan trace storage.
- [ ] Menghubungkan log ↔ trace ↔ metrics melalui traceID tanpa membocorkan secret.
- [ ] Menetapkan redaction, tenant, retention, backpressure, dan missing-data policy.

## Materi dan Praktik

| Hari | Materi | Lab |
|---|---|---|
| 1 | [Loki, label, LogQL, redaction](01-loki-labels-logql-redaction-retention.md) | [LAB-01](lab/LAB-01-alloy-loki-log-pipeline.md) |
| 2 | [Tempo, OTLP, correlation](02-tempo-otlp-traces-correlation.md) | [LAB-02](lab/LAB-02-tempo-trace-log-metrics-correlation.md) |

## Prasyarat

Modul 7.2 Alloy, dasar structured logging, HTTP request/trace propagation, dan Grafana Fase 7.5.

## Acceptance Criteria

- [ ] Labels/attributes bounded, ownership jelas, dan PII/credential terfilter.
- [ ] Query memiliki time range, tenant, expected result, serta no-data interpretation.
- [ ] Sampling/retention/storage dan backpressure didokumentasikan.
- [ ] Correlation evidence memakai redacted trace ID/reference, bukan raw payload.

## Guardrail dan Status Runtime

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Authorization header, bearer token, password, private key, dan PII dilarang pada logs/traces/labels/dashboard. Loki/Tempo ingestion dan correlation **belum diverifikasi** tanpa evidence runtime.

## Troubleshooting dan Catatan SRE

Log ada tetapi tidak queryable dapat disebabkan label/time range/tenant/query parser. Trace ada tetapi tidak berkorelasi dapat disebabkan propagation atau sampling. Jangan menaikkan label cardinality sebagai solusi cepat.

## Kaitan

Hubungkan ke [Alloy](../modul-7.2-alloy-telemetry-pipeline/README.md), [Grafana](../modul-7.5-grafana-alerting-slo/README.md), dan incident practice Fase 8.
