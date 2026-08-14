# LAB-02 — Alert Firing, Notification, dan Failure Injection Disposable

## Tujuan

Menguji signal sampai notification dan membuat failure injection bounded untuk memvalidasi runbook tanpa menyentuh production.

## Prasyarat dan Guardrail

Baca [Alerting](../02-alerting-contact-point-routing-notification.md) dan [SLO/runbook](../03-slo-error-budget-burn-rate-runbook.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Failure injection/chaos production dilarang. Receiver harus disposable/stub; webhook/channel nyata memerlukan authorization terpisah.

## Lane A — Static Simulation

1. Pilih satu synthetic fault: error ratio >5%, p95 >500ms, disk >85% fixture, atau node-down signal fixture.
2. Tulis rule/evaluator/no-data/error policy, route/group/inhibition/silence expiry, contact point reference, and runbook.
3. Buat evidence chain collection → query → dashboard → firing → Alertmanager receipt → notification → action.
4. Tentukan stop condition, rollback, cleanup, escalation, dan SLO/error-budget decision.

## Lane B — Optional Disposable Runtime

1. Verify disposable context/namespace/app/receiver and obtain explicit approval.
2. Snapshot baseline metric/log/trace/dashboard status redacted.
3. Inject only the approved bounded fault with timeout and traffic cap.
4. Observe rule pending/firing, Alertmanager route/receipt, disposable notification accepted/failed, and runbook decision.
5. Abort/rollback immediately on scope expansion, data leakage, resource pressure, or unexpected target.
6. Restore workload, clear silence, verify post-check window, and cleanup.

Alert firing tanpa notification receipt/outcome bukan bukti notifikasi berfungsi. Runtime failure injection saat ini **belum diverifikasi**.

## Evidence Template

```text
fault: <approved-disposable-fault>
baseline: <summary>
rule_state: <summary>
alertmanager_receipt: <accepted-failed-not-tested>
notification: <accepted-failed-not-tested>
runbook_action: <summary>
rollback_postcheck: <summary>
slo_decision: <summary>
status: <verified-or-belum-diverifikasi>
```

## Stop Conditions

Production target, receiver nyata, no rollback, no-data without policy, resource saturation, secret/PII in notification, or fault exceeds approved scope.

## Acceptance Criteria

- [ ] Signal stages and evidence fields complete.
- [ ] Receiver/notification safe and bounded.
- [ ] Fault disposable, reversible, approved, and time-limited.
- [ ] Runbook/SLO decision documented without overclaim.

## Troubleshooting

No firing: query/fixture/time range/evaluator. Firing no receipt: route/silence/inhibition. Receipt no notification: receiver/network/fallback. Recovery slow: stop fault and follow rollback/runbook.
