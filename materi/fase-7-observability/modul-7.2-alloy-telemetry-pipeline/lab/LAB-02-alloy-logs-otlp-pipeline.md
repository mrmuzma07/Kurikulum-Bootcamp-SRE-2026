# LAB-02 — Alloy Logs dan OTLP Pipeline

## Tujuan

Mendesain dan menguji secara aman tail/parse/redact logs serta OTLP traces ke backend disposable.

## Prasyarat dan Guardrail

Baca [OTel pipeline](../02-otel-pipeline-daemonset-helm-debugging.md) dan Modul 7.4.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menggunakan log nyata yang berisi PII, token, password, authorization header, atau private key.

## Lane A — Static Simulation

1. Buat sample structured log sintetis tanpa credential dan trace fixture dengan trace ID dummy.
2. Tetapkan source path, parser, redaction fields, bounded labels, queue, retry, sampling, and storage references.
3. Review OTLP protocol/resource attributes/propagation dan Tempo/Loki endpoints.
4. Tentukan expected LogQL query dan correlation steps.
5. Model backend unavailable, rotation, permission, clock skew, and drop handling.

## Lane B — Optional Disposable Runtime

1. Verifikasi disposable namespace, synthetic workload, backend tenant, and receiver scope.
2. Deploy/update Alloy via approved Helm/GitOps path after diff review.
3. Emit only synthetic structured logs/traces.
4. Verify redaction before export, queue/drop summary, Loki query, Tempo trace query, and traceID correlation.
5. Capture only metadata/query summary; do not export raw payload.
6. Cleanup synthetic workload and collector within approved scope.

## Evidence Template

```text
pipeline_revision: <reference>
synthetic_source: <class-and-count>
redaction_check: <summary>
loki_query: <result-summary-or-not-tested>
tempo_query: <result-summary-or-not-tested>
correlation: <verified-not-verified>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Synthetic data only dan redaction boundary jelas.
- [ ] Labels/attributes bounded.
- [ ] Loki/Tempo query dan correlation tidak mencetak payload.
- [ ] Drop/backpressure/error policy memiliki evidence.

## Stop Conditions dan Troubleshooting

Stop bila log/trace nyata masuk, redaction gagal, receiver bukan disposable, atau time/tenant ambiguity. Query kosong: cek parser, labels, tenant, time range, protocol, sampling, and ingestion delay.
