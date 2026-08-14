# 01 — Grafana USE/RED dan Dashboard as Code

## Tujuan

Membuat dashboard yang menjawab pertanyaan operasi, dapat direview sebagai code, dan dibuktikan menampilkan data runtime.

## 1. USE dan RED

**USE** untuk resource: Utilization, Saturation, Errors. Contoh panel CPU utilization, filesystem utilization, queue saturation, scrape errors, node condition.

**RED** untuk service: Rate, Errors, Duration. Contoh request rate, error ratio, p95 latency, dependency error, and traffic by bounded service/route template.

Panel harus memiliki datasource UID/reference, query version, unit, legend, threshold, time window, variable allowlist, owner, and runbook link. Jangan menjadikan dashboard sebagai tempat menyimpan credential.

## 2. Dashboard as Code

```json
{
  "title": "<approved-service-red-dashboard>",
  "tags": ["sre", "red"],
  "templating": {"list": [{"name": "environment", "type": "custom", "query": "<approved-values>"}]},
  "panels": [
    {
      "title": "Error ratio",
      "type": "timeseries",
      "datasource": {"type": "prometheus", "uid": "<approved-datasource-uid>"},
      "targets": [{"expr": "<approved-recording-rule>", "refId": "A"}],
      "fieldConfig": {"defaults": {"unit": "percent"}}
    }
  ]
}
```

JSON di atas hanyalah contract non-secret. Valid JSON/provisioning success tidak berarti panel menemukan data. Dashboard deployment via GitOps perlu ownership, revision, RBAC, and rollback.

## 3. Evidence

Bukti panel harus mencatat datasource/reference, query ID atau revision, time range, series/sample summary yang diredáksi, render status, dan timestamp. `Grafana dapat dibuka` hanya bukti access/UI, bukan data correctness atau SLO.

## Acceptance Criteria

- [ ] Panel USE/RED menjawab pertanyaan operasi.
- [ ] Query, unit, threshold, variable, owner, dan runbook jelas.
- [ ] Dashboard JSON tidak memiliki secret/PII.
- [ ] Runtime evidence membuktikan data panel, bukan hanya provisioning.

## Troubleshooting dan Catatan SRE

Panel kosong: cek datasource, UID, tenant, time range, query, label, ingestion, and clock. Panel menyesatkan: cek unit/aggregation/denominator. Variable terlalu bebas dapat menyebabkan expensive query atau tenant leak; gunakan allowlist.

## Kaitan

Praktikkan [LAB-01](lab/LAB-01-dashboard-data-source-evidence.md) dan lanjutkan ke [Alerting](02-alerting-contact-point-routing-notification.md).

## Status Runtime

Grafana deployment, datasource, dashboard render/data, dan cross-signal panel **belum diverifikasi**.
