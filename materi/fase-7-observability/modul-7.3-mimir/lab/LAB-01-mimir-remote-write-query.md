# LAB-01 — Mimir Remote-Write dan Query

## Tujuan

Membuktikan data path metrics secara bertahap, bukan hanya status Helm atau HTTP accepted.

## Prasyarat dan Guardrail

Baca Modul 7.3. Runtime hanya Mimir/collector disposable dengan storage yang scope-nya jelas.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Lane A — Static Simulation

Tulis data-path table:

| Tahap | Input/output | Failure | Evidence |
|---|---|---|---|
| collector | samples/queue | scrape/drop | queue summary |
| distributor | remote-write | auth/limit | response class |
| ingester | active samples | ring/replica | component health |
| object storage | blocks | bucket/network | block summary |
| querier | query | tenant/time range | query result |
| Grafana | panel | datasource | render/data evidence |

Review tenant, endpoint, retention, limits, object storage reference, and query window.

## Lane B — Optional Disposable Runtime

1. Verify context, namespace, storage, tenant reference, and target ownership.
2. Install/upgrade through approved Helm/GitOps path after render/diff review.
3. Generate bounded synthetic samples.
4. Verify remote-write response, queue, ingester/backend status, and ingestion delay.
5. Query exact metric/labels with bounded range and record redacted result summary.
6. Roll back/cleanup using scoped procedure.

## Evidence Template

```text
mimir_revision: <reference>
remote_write_response: <class-and-count>
ingestion_delay: <summary>
query_result: <series-summary-or-not-tested>
object_storage: <reference-and-status>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Data path stages and failure domains complete.
- [ ] Query result distinct from transport status.
- [ ] Tenant/object storage/credential boundary safe.
- [ ] Runtime evidence redacted and disposable.

## Troubleshooting

HTTP accepted/query empty: cek delay, tenant, label, ingester, object storage, compactor, and query frontend. Jangan retry tanpa bound.
