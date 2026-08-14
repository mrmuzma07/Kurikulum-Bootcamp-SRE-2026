# 02 — Retention, HA, Query, Recording Rules, dan Recovery

## Tujuan

Membuat capacity/reliability design untuk Mimir dan menghubungkan recording/alerting rules dengan evidence query dan SLO.

## 1. Capacity dan Retention

Hitung secara eksplisit:

```text
ingest_samples_per_second
× bytes_per_sample_estimate
× retention_window
+ index/compaction/replication overhead
+ query/cache/headroom
```

Gunakan measurement aktual bila runtime tersedia; angka latihan harus diberi label asumsi. Cardinality, label churn, scrape interval, replica, tenant limits, query range/concurrency, object storage I/O, dan compactor work memengaruhi capacity.

## 2. HA dan Recovery

HA bukan hanya replica count. Tetapkan failure domain node/zone, ring state, replication factor, quorum, rollout strategy, object storage availability, WAL/queue behavior, backup, restore test, and RPO/RTO. Recovery harus memisahkan data yang masih di queue, data durable, data delayed, dan data benar-benar hilang.

Retention deletion dan compaction dapat menghapus data; memerlukan approval, audit, and recovery expectation. Jangan menguji restore pada object storage/tenant aktif tanpa prosedur resmi.

## 3. Query dan Rules

Grafana query harus memiliki tenant, time range, step, limit, and timeout. Recording rules mengurangi query cost tetapi menambah write/cardinality. Alerting rule di ruler/Prometheus harus memiliki version, owner, evaluation window, no-data policy, and notification boundary.

```yaml
groups:
  - name: <approved-mimir-rule-group>
    rules:
      - record: sre:service_latency_p95:5m
        expr: <approved-histogram-quantile-expression>
      - alert: <approved-burn-rate-alert>
        expr: <approved-burn-rate-expression>
        for: <approved-duration>
        labels:
          severity: <approved-severity>
          sre_owner: <approved-owner>
        annotations:
          runbook_url: <approved-runbook-url>
```

## 4. Evidence

Pisahkan `remote_write accepted`, `ingestion timestamp`, `query returns series`, `Grafana panel renders`, `rule evaluates`, `Alertmanager receives`, dan `SLO decision`. Satu status controller tidak dapat menggantikan chain ini.

## Acceptance Criteria

- [ ] Capacity/retention memakai asumsi dan measurement plan.
- [ ] HA/recovery memiliki RPO/RTO, owner, and stop condition.
- [ ] Rules versioned, bounded, dan queryable.
- [ ] Data path evidence terpisah dari SLO.

## Troubleshooting dan Catatan SRE

High query latency: cek range/step/cardinality/cache/querier/frontend. Ingester overload: cek distributor limits, queue, replication, and noisy tenant. Rule delay: cek scheduler/evaluation, missing series, and backend delay.

## Kaitan

Praktikkan [LAB-02](lab/LAB-02-mimir-retention-capacity-recovery.md) dan hubungkan ke [SLO](../modul-7.5-grafana-alerting-slo/03-slo-error-budget-burn-rate-runbook.md).

## Status Runtime

Retention, HA, recovery, rule evaluation, dan Mimir query runtime **belum diverifikasi**.
