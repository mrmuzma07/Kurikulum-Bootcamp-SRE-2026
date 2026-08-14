# LAB-01 — Alloy ke Loki: Log Pipeline

## Tujuan

Menguji log pipeline synthetic dari source sampai LogQL query dengan label bounded dan redaction.

## Prasyarat dan Guardrail

Baca [Loki labels/LogQL](../01-loki-labels-logql-redaction-retention.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Lane A — Static Simulation

1. Tulis synthetic JSON log tanpa PII/secret.
2. Tentukan parser, redaction fields, labels bounded, tenant, retention, and query time range.
3. Tulis lima query LogQL untuk error, JSON level, rate, workload, dan time-window comparison.
4. Review stream count, payload size, rotation, queue/drop, object storage, and access.

## Lane B — Optional Disposable Runtime

1. Verify context/namespace/backend/tenant and synthetic workload.
2. Deploy/update Alloy/Loki after render/diff/approval.
3. Emit bounded synthetic logs and observe redaction/drop metrics.
4. Query LogQL with bounded selectors; capture counts/time/status only.
5. Stop if any sensitive field appears; cleanup target and data through approved disposable procedure.

## Evidence Template

```text
source_revision: <reference>
stream_labels: <bounded-summary>
redaction: <verified-or-not-tested>
logql_results: <query-count-summary>
retention: <policy-reference>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Labels low-cardinality.
- [ ] Synthetic logs and redaction checked.
- [ ] Five LogQL query intents jelas.
- [ ] Runtime query evidence redacted.

## Troubleshooting

No logs: cek source path/permission/rotation/parser/tenant. Too many streams: remove unbounded labels. Sensitive log: stop, isolate, rotate via approved process, do not copy payload.
