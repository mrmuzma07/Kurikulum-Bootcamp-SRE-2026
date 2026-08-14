# Modul 7.3 — Metrics Skala Production: Mimir

> **Tujuan:** memahami central metrics platform dari remote-write sampai long-term query dengan tenancy, object storage, retention, HA, dan recovery.

## Capaian

- [ ] Menjelaskan distributor, ingester, querier, query-frontend, store-gateway, compactor, ruler, dan ring.
- [ ] Mendesain remote-write dari collector per cluster menuju Mimir terpusat.
- [ ] Menetapkan tenant, auth/TLS boundary, object-storage lifecycle, retention, replication, dan capacity.
- [ ] Membedakan remote-write accepted, data queryable, Grafana panel, dan SLO evidence.
- [ ] Menulis recording/alerting rules sebagai code dan merencanakan recovery.

## Materi dan Praktik

| Hari | Materi | Lab |
|---|---|---|
| 1 | [Arsitektur dan remote-write](01-arsitektur-mimir-remote-write-object-storage.md) | [LAB-01](lab/LAB-01-mimir-remote-write-query.md) |
| 2 | [Retention, HA, query, recovery](02-retention-ha-query-recording-rules.md) | [LAB-02](lab/LAB-02-mimir-retention-capacity-recovery.md) |

## Prasyarat

Modul 7.1 dan 7.2, Helm, storage/retention fundamentals, serta GitOps Fase 6. Object storage nyata dan multi-cluster bukan asumsi default local lab.

## Acceptance Criteria

- [ ] Topologi menjelaskan data path, control path, replica, tenant, dan failure domain.
- [ ] Capacity plan mencakup ingest rate, cardinality, retention, object storage, query load, dan headroom.
- [ ] Remote-write retry/backpressure dan data-loss/delay decision terdokumentasi.
- [ ] Recovery dan deletion/retention safety memiliki owner dan approval.

## Guardrail dan Status Runtime

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Bucket key, tenant credential, TLS private key, dan raw state tidak boleh masuk README/values/evidence. Mimir deployment, object storage, remote-write, dan query **belum diverifikasi** tanpa evidence.

## Troubleshooting dan Catatan SRE

Remote-write HTTP success bukan jaminan sample sudah durable/queryable. Periksa queue, distributor, ingester, ring, compactor, object storage, tenant, dan clock. Retention/compaction deletion bersifat data-impacting dan membutuhkan review.

## Kaitan

Lanjutkan ke [Grafana](../modul-7.5-grafana-alerting-slo/README.md) dan correlation [Loki + Tempo](../modul-7.4-loki-tempo/README.md).
