# LAB-02 — PromQL, Rules, dan Alertmanager

## Tujuan

Membuat rule untuk error/latency/disk/node dan menelusuri signal sampai Alertmanager tanpa menyamakan firing dengan notification.

## Prasyarat dan Guardrail

Baca [Rules dan Alertmanager](../02-rules-alertmanager-retention-cardinality.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Receiver/webhook/Telegram/Slack harus berupa reference non-secret. Jangan mengirim alert ke channel nyata tanpa authorization.

## Lane A — Static Simulation

1. Buat fixture metric atau table input untuk counter, histogram, disk gauge, dan absent series.
2. Tulis recording rule error ratio dan p95.
3. Tulis alert rule untuk error rate >5%, p95 >500ms, disk >85%, node down.
4. Tentukan `for`, severity, owner, query reference, runbook, no-data/error policy.
5. Model Alertmanager `group_by`, route critical/warning, inhibition, silence expiry, dan fallback receiver.
6. Review duplicate alert, cardinality, threshold calibration, and notification redaction.

## Lane B — Optional Disposable Runtime

1. Verifikasi context/namespace/target dan approved receiver stub.
2. Load rules melalui approved path, bukan secret-bearing CLI flags.
3. Generate bounded synthetic error/latency/disk condition atau use controlled fixture.
4. Buktikan rule evaluation state/time/query summary.
5. Buktikan Alertmanager receipt/group/route secara redacted.
6. Jika ada receiver disposable, buktikan accepted/failed outcome tanpa mencetak payload.
7. Hapus silence/test condition dan cleanup target.

`Firing` tanpa Alertmanager/notification evidence tetap **belum diverifikasi** sebagai alert delivery.

## Evidence Template

```text
rule_revision: <reference>
evaluation: <pending-firing-resolved-or-no-data>
alert_summary: <redacted>
alertmanager_route: <route-class>
receiver_outcome: <accepted-failed-or-not-tested>
notification_status: <verified-or-not-tested>
status: <verified-or-belum-diverifikasi>
```

## Stop Conditions

NoData tanpa policy, route memilih receiver nyata, alert flood, secret muncul pada payload/log, threshold tidak bounded, atau target production.

## Acceptance Criteria

- [ ] Empat alert memiliki query/window/owner/severity/runbook/no-data policy.
- [ ] Grouping/routing/inhibition/silence mempunyai scope dan expiry.
- [ ] Evaluation, receipt, dan notification evidence terpisah.
- [ ] Static/runtime status jujur.

## Troubleshooting dan Kaitan

Alert tidak firing: cek query/labels/for/time range. Firing tetapi no receipt: cek route/silence/inhibition. Receipt tetapi no delivery: cek receiver/network/escalation. Hubungkan ke [Modul 7.5](../../modul-7.5-grafana-alerting-slo/README.md).
