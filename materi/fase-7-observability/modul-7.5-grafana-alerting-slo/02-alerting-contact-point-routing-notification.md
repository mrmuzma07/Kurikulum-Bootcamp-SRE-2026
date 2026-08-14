# 02 — Alert Rule, Contact Point, Routing, dan Notification

## Tujuan

Memisahkan rule evaluation dari routing dan delivery sehingga operator dapat membuktikan di mana signal berhenti.

## 1. Alert Boundary

```text
query/data → evaluator → pending/firing/no-data/error
→ Alertmanager/Grafana notification policy
→ grouping/inhibition/silence
→ contact point/webhook/channel
→ receiver accepted → operator action
```

Prometheus/Alertmanager dan Grafana unified alerting dapat memiliki boundary berbeda. Pilih ownership untuk tiap rule, jangan membuat duplicate alert tanpa deduplication plan.

## 2. Alert Contract

Setiap alert memiliki query reference, threshold, evaluation interval, `for`, severity, owner, environment, runbook URL placeholder, dashboard reference, no-data policy, error policy, and notification route. Label harus bounded dan tidak menyimpan request/user data.

```yaml
alert: <approved-latency-alert>
expr: <approved-p95-query> > <approved-threshold>
for: <approved-duration>
labels:
  severity: critical
  sre_owner: <approved-owner>
annotations:
  summary: <redacted-summary>
  runbook_url: <approved-runbook-url>
  no_data_policy: <pause-and-investigate>
```

## 3. Routing dan Notification

Route berdasarkan severity/environment/owner allowlist. Grouping mengurangi flood; inhibition hanya boleh menekan symptom yang benar-benar redundant. Silence harus expire. Contact point secret berada di secret manager/protected integration, bukan JSON, Helm values, dashboard, atau shell argument.

Notification evidence perlu status sent/accepted/retried/failed, timestamp, route/reference, and redacted destination class. Alertmanager receipt tidak membuktikan operator menerima atau bertindak.

## 4. Failure dan Maintenance

NoData/error bukan otomatis OK. Maintenance window/silence harus memiliki ticket, scope, expiry, owner, and post-check. Notification failure memerlukan fallback/escalation dan alert meta-monitoring. Jangan menguji webhook nyata tanpa authorization dan disposable receiver.

## Acceptance Criteria

- [ ] Rule/evaluator/router/contact point/receiver/operator dibedakan.
- [ ] Alert memiliki owner, severity, query, window, threshold, runbook, no-data policy.
- [ ] Silence/inhibition/maintenance bounded dan expire.
- [ ] Notification secret-safe dan delivery evidence terpisah.

## Troubleshooting dan Catatan SRE

Alert tidak firing: cek query/time range/label/for/no-data/evaluator. Firing tapi tidak diterima: cek route/group/silence/inhibition. Diterima tapi tidak sampai operator: cek channel/receiver/escalation. Jangan mencetak payload yang mungkin memuat PII/secret.

## Kaitan

Gunakan [SLO dan burn rate](03-slo-error-budget-burn-rate-runbook.md) serta [LAB-02](lab/LAB-02-alert-firing-notification-failure-injection.md).

## Status Runtime

Alert evaluation, Alertmanager routing, contact point, dan notification **belum diverifikasi**.
