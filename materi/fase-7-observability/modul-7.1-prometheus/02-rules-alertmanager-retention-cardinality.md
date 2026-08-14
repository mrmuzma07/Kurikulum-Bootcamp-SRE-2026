# 02 — Rules, Alertmanager, Retention, dan Cardinality

## Tujuan

Membuat rule yang dapat dioperasikan, mengendalikan cardinality/retention, dan memastikan alert bergerak dari evaluation sampai notification tanpa menutupi missing data.

## 1. Recording Rule

Recording rule menyimpan hasil query mahal dengan nama stabil agar dashboard/alert konsisten. Rule dikelola sebagai code, direview, diberi owner, dan diuji dengan input fixture atau runtime evidence.

```yaml
groups:
  - name: <approved-service-recording-rules>
    interval: <approved-evaluation-interval>
    rules:
      - record: sre:service_request_error_ratio:rate5m
        expr: |
          sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (service) (rate(http_requests_total[5m]))
```

Denominator nol, absent series, counter reset, dan label mismatch harus memiliki keputusan eksplisit.

## 2. Alerting Rule

```yaml
- alert: <approved-service-error-rate>
  expr: sre:service_request_error_ratio:rate5m > <approved-threshold>
  for: <approved-duration>
  labels:
    severity: warning
    sre_owner: <approved-owner>
    sre_environment: <approved-environment>
  annotations:
    summary: <redacted-summary>
    runbook_url: <approved-runbook-url>
    query_reference: <approved-query-id>
    no_data_policy: <pause-or-investigate>
```

Rule harus menjawab symptom, threshold, window, severity, owner, runbook, query reference, dan no-data/error policy. `for` mencegah noise singkat tetapi menambah detection latency.

## 3. Alertmanager

Alertmanager menerima alert dari Prometheus/Grafana lalu melakukan grouping, routing, inhibition, deduplication, silence, dan receiver delivery. Buat route berdasarkan allowlist label seperti environment/severity/owner. Receiver secret direferensikan dari secret manager, bukan diletakkan dalam YAML.

```yaml
route:
  group_by: [alertname, sre_environment, sre_owner]
  group_wait: <approved-duration>
  group_interval: <approved-duration>
  repeat_interval: <approved-duration>
  receiver: <approved-default-receiver>
  routes:
    - matchers:
        - severity="critical"
      receiver: <approved-critical-receiver>
      continue: false
```

Silence harus memiliki creator, reason, expiry, scope, dan review. Inhibition harus sempit agar alert symptom tidak menutupi root cause penting. Alert `Firing` → Alertmanager receipt → notification accepted → operator menerima adalah evidence berbeda.

## 4. Cardinality, Retention, dan Capacity

Jangan jadikan user ID, request ID, raw URL, token, arbitrary query, atau exception text sebagai label. Batasi nilai environment/service/namespace/route. Ukur active series, samples/sec, scrape payload, WAL/disk, rule evaluation latency, query concurrency, dan retention. Retention mengontrol biaya/storage, bukan menggantikan backup atau HA.

## 5. Empat Scenario Kurikulum

- error rate >5%: definisikan denominator dan window; 5% bukan angka universal.
- p95 latency >500ms: pastikan histogram bucket dan aggregation benar.
- disk >85%: gunakan filesystem allowlist, exclude ephemeral mounts, dan runbook cleanup/expansion.
- node down: bedakan target scrape failure, exporter failure, network partition, dan node failure.

## Static/Runtime dan Acceptance

Static review memeriksa rule expression, label, route, retention, no-data, dan redaction. Runtime harus membuktikan evaluation timestamp/state, Alertmanager receipt ID/summarized payload, notification outcome, dan redacted evidence. Tanpa itu status **belum diverifikasi**.

- [ ] Rules versioned dan memiliki owner/severity/runbook.
- [ ] Alertmanager route/group/inhibition/silence memiliki bounded scope.
- [ ] Cardinality/retention/capacity memiliki measurement plan.
- [ ] Missing data tidak otomatis dianggap healthy.

## Troubleshooting dan Catatan SRE

Alert flood: cek grouping, cardinality, rule window, duplicate labels, dan inhibition. NoData: cek scrape/remote-write/query path; pause atau investigate, jangan auto-resolve menjadi sukses. Notification gagal: cek receiver/network/credential reference dan escalation path tanpa mencetak secret.

## Kaitan

Gunakan [LAB-02](lab/LAB-02-promql-rules-alertmanager.md) dan lanjutkan ke [Mimir](../modul-7.3-mimir/README.md).

## Status Runtime

Rule/route di atas adalah static examples. Rule evaluation dan notification runtime **belum diverifikasi**.
