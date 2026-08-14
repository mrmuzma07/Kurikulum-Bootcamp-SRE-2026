# LAB-02 — Tempo dan Correlation Trace-Log-Metrics

## Tujuan

Membuktikan correlation pada synthetic workload dan memahami batas sampling, propagation, clock, dan query evidence.

## Prasyarat dan Guardrail

Baca [Tempo/OTLP](../02-tempo-otlp-traces-correlation.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Lane A — Static Simulation

1. Model request → metric counter/histogram → trace spans → structured log with safe trace ID.
2. Review OTLP endpoint/protocol, resource attributes, propagation, sampling, retention, max trace size.
3. Buat correlation matrix: RED spike, trace reference, span timeline, Loki query, expected caveat.
4. Pastikan trace ID tidak dipakai sebagai Loki stream label dan payload tidak memuat credential.

## Lane B — Optional Disposable Runtime

1. Verify synthetic app, collector, Loki/Tempo/Mimir, Grafana, namespace, and time synchronization.
2. Generate bounded requests with controlled error/latency.
3. Verify metrics query, trace ingestion/query, safe log trace reference, and Grafana correlation.
4. Record trace reference class, timestamps, sampling, query summary; never raw spans/logs.
5. Cleanup synthetic workload and data within disposable policy.

## Evidence Template

```text
workload_revision: <reference>
metric_summary: <redacted>
trace_summary: <span-count-and-status>
log_reference: <safe-reference-class>
correlation: <verified-or-not-tested>
sampling: <policy>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Propagation, sampling, and clock assumptions jelas.
- [ ] Correlation memakai bounded references.
- [ ] Query results and SLO separated.
- [ ] Runtime only synthetic/disposable.

## Troubleshooting

Trace absent: cek instrumentation/exporter/protocol/sampling. Log reference mismatch: cek service name/time/propagation. Clock skew: fix time sync before interpreting latency.
