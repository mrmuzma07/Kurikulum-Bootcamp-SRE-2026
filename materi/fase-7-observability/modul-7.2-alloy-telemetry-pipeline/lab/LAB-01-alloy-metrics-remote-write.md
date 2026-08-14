# LAB-01 — Alloy Metrics dan Remote-Write

## Tujuan

Mereview pipeline scrape → relabel → queue → remote-write dan membuktikan perbedaan collector health dengan Mimir query.

## Prasyarat dan Guardrail

Baca [Alloy components](../01-alloy-components-metrics-logs-traces.md) dan [Mimir architecture](../../modul-7.3-mimir/01-arsitektur-mimir-remote-write-object-storage.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Lane A — Static Simulation

1. Gambar graph input/exporter → scrape → bounded relabel → remote-write.
2. Tetapkan endpoint/tenant/auth/TLS sebagai references, queue size, retry max, timeout, and drop policy.
3. Hitung sample/cardinality budget dan resource request/limit.
4. Buat failure matrix: target down, backend 5xx, queue full, auth failure, out-of-order, tenant denied.
5. Review ARM64/AMD64 DaemonSet, RBAC, host access, and network policy.

## Lane B — Optional Disposable Runtime

1. Verifikasi context, namespace, Mimir target, storage, and no production selection.
2. Render/lint Helm values; obtain approval before install/upgrade.
3. Verify Alloy config/component health and discovered targets.
4. Generate bounded synthetic metrics.
5. Capture queue/retry/drop summary and remote-write response class.
6. Query Mimir after expected ingestion delay using approved tenant reference.
7. Cleanup/rollback within disposable scope.

Remote-write success tanpa query result bukan evidence metrics tersedia.

## Evidence Template

```text
alloy_revision: <chart-config-reference>
source_scope: <approved-target-class>
queue_summary: <redacted>
remote_write: <accepted-failed-not-tested>
mimir_query: <series-summary-or-not-tested>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Queue/retry/drop/cardinality policy jelas.
- [ ] Credential/tenant/TLS hanya reference.
- [ ] Collector, transport, ingestion, dan query evidence dibedakan.
- [ ] Runtime scope disposable.

## Troubleshooting

Backlog: cek backend capacity/network/tenant. Query kosong: cek ingestion delay, tenant, labels, and Mimir components. Drop meningkat: cek relabel/cardinality/queue and resource.
