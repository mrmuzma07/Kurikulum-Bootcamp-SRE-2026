# 01 — Loki, Label Strategy, LogQL, Redaction, dan Retention

## Tujuan

Membangun log pipeline yang searchable tanpa mengubah arbitrary log fields menjadi index cardinality tak terkendali atau menyimpan secret.

## 1. Label Strategy

Loki mengelompokkan log ke streams berdasarkan labels. Gunakan label bounded seperti `cluster`, `namespace`, `workload`, `container`, dan `environment`. Jangan gunakan `user_id`, `request_id`, raw URL, exception text, token, atau query value sebagai stream label.

Parsing JSON tidak sama dengan indexing semua field. Simpan field yang aman untuk query-time filter dan batasi ukuran/retention. Tenant isolation dan retention harus eksplisit.

## 2. LogQL Dasar

```logql
{namespace="<approved-namespace>", workload="<approved-workload>"} |= "ERROR"

{namespace="<approved-namespace>"} | json | level="error"

rate({namespace="<approved-namespace>", workload="<approved-workload>"}[5m])
```

Selalu batasi time range dan stream selector. `rate()` terhadap log stream bukan sama dengan application error ratio tanpa definisi denominator. Query output/evidence harus diredáksi.

## 3. Redaction dan Retention

Filter `Authorization`, `Cookie`, password, API key, private key block, session token, dan PII sebelum storage/export. Redaction yang terlambat tetap dapat membocorkan data pada collector buffer atau debug log. Tetapkan retention per tenant/classification, storage capacity, compaction, deletion audit, and legal/operational requirements.

## 4. Failure Modes

Backpressure, dropped entries, file rotation, permission, clock skew, tenant/auth failure, object storage outage, dan query overload harus memiliki metric/runbook. Log pipeline sehat tidak membuktikan service sehat; log absence dapat berarti no traffic, drop, filter, atau outage.

## Acceptance Criteria

- [ ] Labels low-cardinality dan bounded.
- [ ] LogQL memiliki selector/time range/expected interpretation.
- [ ] Redaction dan retention terjadi pada boundary yang benar.
- [ ] Drop/backpressure/storage failure memiliki policy.

## Troubleshooting dan Catatan SRE

Stream terlalu banyak: audit labels dan drop high-cardinality. Query kosong: cek time zone/time range/tenant/labels/ingestion delay. Log mengandung secret: stop distribution, preserve minimal incident metadata, rotate credential melalui prosedur resmi, dan jangan menyalin payload ke evidence.

## Kaitan

Praktikkan [LAB-01](lab/LAB-01-alloy-loki-log-pipeline.md) dan lanjutkan ke [Tempo correlation](02-tempo-otlp-traces-correlation.md).

## Status Runtime

Loki ingestion, query, retention, dan redaction runtime **belum diverifikasi**.
