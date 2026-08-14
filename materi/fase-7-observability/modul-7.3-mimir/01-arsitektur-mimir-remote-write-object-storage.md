# 01 — Arsitektur Mimir, Remote-Write, dan Object Storage

## Tujuan

Memahami path metrics terpusat dari Alloy/Prometheus ke Mimir dan failure domain setiap komponen.

## 1. Komponen dan Data Path

```text
Alloy/Prometheus
  → remote_write
  → distributor
  → ingester/ring/replication
  → blocks/object storage
  → compactor/store-gateway
  → querier/query-frontend
  → Grafana
```

Distributor memvalidasi/mendistribusikan samples; ingester menyimpan data aktif; object storage menyimpan blocks; compactor menggabungkan/retensi; store-gateway membaca blocks; querier/query-frontend melayani query; ruler mengevaluasi rule sesuai deployment model. Ring dan replication menentukan availability/cost.

## 2. Remote-Write Contract

Dokumentasikan endpoint, tenant header/reference, TLS/auth reference, queue capacity, retry/backoff, timeout, batch, sample limit, and expected status. HTTP accepted bukan bukti sample durable atau queryable. Pantau queue pending, failed samples, retries, latency, out-of-order, and backend ingestion.

## 3. Object Storage

Mimir memerlukan object storage untuk long-term blocks pada topologi production. Review bucket lifecycle, encryption, versioning, access policy, network path, capacity, delete/retention behavior, and recovery. Access key/secret key tidak boleh masuk values, README, log, atau evidence. Gunakan secret manager reference dan ownership jelas.

## 4. Tenancy dan Security

Tenant boundary harus menentukan siapa boleh write/query/rule. Enforce auth/TLS/network policy, limit per tenant, and RBAC. Jangan memperluas query ke tenant lain untuk troubleshooting tanpa approval dan audit.

## Static/Runtime

Static topology review dapat memakai placeholder endpoint, tenant, bucket, and credentials. Runtime membuktikan remote-write accepted, ingester/storage state, query result after ingestion delay, and Grafana datasource. Tanpa data path evidence status **belum diverifikasi**.

## Acceptance Criteria

- [ ] Data/control/failure path tiap komponen dijelaskan.
- [ ] Remote-write queue/retry dan ingestion delay memiliki policy.
- [ ] Object storage lifecycle, security, retention, dan recovery direview.
- [ ] Tenant/auth/RBAC/network boundary bounded.

## Troubleshooting dan Catatan SRE

Query kosong: cek tenant, time range, ingestion delay, distributor/ingester, object storage, compactor, dan query frontend. Jangan menyimpulkan data hilang dari satu query. Remote-write backlog dapat menandakan backend overload atau network failure.

## Kaitan

Lanjutkan ke [Retention/HA/Rules](02-retention-ha-query-recording-rules.md) dan [Grafana](../modul-7.5-grafana-alerting-slo/README.md).

## Status Runtime

Mimir deployment, object storage, remote-write, dan query **belum diverifikasi** tanpa execution evidence.
