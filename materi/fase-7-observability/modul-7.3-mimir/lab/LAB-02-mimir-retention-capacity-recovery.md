# LAB-02 — Mimir Retention, Capacity, dan Recovery Review

## Tujuan

Mereview retention/capacity dan menyusun recovery drill tanpa menghapus data penting atau menyentuh storage production.

## Prasyarat dan Guardrail

Baca [Retention/HA](../02-retention-ha-query-recording-rules.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menjalankan retention deletion, restore, bucket mutation, atau failover pada production.

## Lane A — Static Simulation

1. Hitung sample rate × bytes × retention + replication/index/compaction/query headroom menggunakan asumsi berlabel.
2. Buat matrix single-node, replicated, object-storage outage, ingester loss, compactor delay, and tenant overload.
3. Tetapkan RPO/RTO, queue behavior, backup/restore owner, and stop condition.
4. Review recording/alert rule version, evaluation lag, and query performance.
5. Tulis recovery runbook yang membedakan delayed, durable, and lost telemetry.

## Lane B — Optional Disposable Runtime

Hanya bila disposable object storage/namespace dan approval tersedia: ukur ingest/cardinality/disk/queue, lakukan fault simulation yang reversible, amati recovery, lalu restore/cleanup. Jangan menghapus blocks atau mengubah retention aktif tanpa procedure resmi.

## Evidence Template

```text
capacity_assumptions: <summary>
retention_policy: <reference>
fault_scope: <approved-disposable-scope>
recovery_observed: <summary-or-not-tested>
rpo_rto: <summary>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Capacity dan retention assumptions terlihat.
- [ ] RPO/RTO/recovery owner jelas.
- [ ] Fault/recovery scope aman.
- [ ] Tidak ada raw state/credential/object key.

## Troubleshooting

Disk growth: audit cardinality/retention/compaction. Recovery lambat: cek ring/object storage/query load. Jangan menambah retention sebagai satu-satunya solusi.
