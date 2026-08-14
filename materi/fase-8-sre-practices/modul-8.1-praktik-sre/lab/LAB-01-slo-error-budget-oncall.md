# LAB-01 — SLO, Error Budget, dan On-Call

## Tujuan

Menyusun SLO contract, menghitung error budget/burn rate, memetakan alert ke on-call, dan menghasilkan keputusan reliability tanpa memakai credential atau traffic production.

## Prasyarat dan Guardrail

Gunakan teori [SLI/SLO](../01-sli-slo-error-budget-toil.md) dan telemetry reference [Fase 7](../../../fase-7-observability/README.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Target runtime, bila dipilih, harus disposable; namespace, cluster, owner, traffic generator, window, dan approval diverifikasi. Jangan mengirim notification nyata tanpa boundary dan approval.

## Lane A — Static Simulation

### 1. SLO contract

Isi placeholder berikut untuk aplikasi sendiri:

```yaml
service: <approved-service>
owner: <approved-team>
window: <approved-calendar-window>
objective: <approved-percentage>
numerator: <successful-eligible-requests>
denominator: <eligible-requests>
eligibility: <approved-traffic-policy>
exclusions: <approved-expiring-exclusions>
missing_data_policy: <pause-or-review-policy>
query_reference: <fase-7-query-or-dashboard-reference>
runbook: <approved-runbook-reference>
review_cadence: <approved-cadence>
```

Jelaskan status code, retry, timeout, synthetic traffic, maintenance, and database dependency. Exclusion tanpa owner/expiry dianggap tidak valid.

### 2. Error-budget math

Gunakan angka sintetis yang dapat diaudit:

```text
eligible requests: <synthetic-denominator>
objective: <synthetic-objective>
error budget fraction: 1 - objective
allowed failures: denominator × budget fraction
observed failures: <synthetic-observed-failures>
remaining budget: allowed failures - observed failures
```

Buat contoh burn rate untuk window pendek dan panjang. Jelaskan kapan alert page, ticket, atau review manual. `NoData` harus memiliki policy terpisah dari zero errors.

### 3. On-call matrix

| Alert/severity | Primary | Secondary | Ack | Escalation | Runbook | Paging boundary |
|---|---|---|---|---|---|---|
| `<approved-alert>` | `<role>` | `<role>` | `<duration>` | `<role>` | `<reference>` | `<page-or-ticket>` |

Tambahkan handoff UTC, timezone coverage, fatigue control, access recovery, and change-freeze interaction.

### 4. Evidence dan decision

Tulis evidence chain static:

```text
SLO revision → query/dashboard review → alert metadata review
→ on-call coverage review → error-budget decision → promotion/change action
```

Static output hanya membuktikan desain telah direview. Tidak boleh ditulis sebagai metric query runtime atau paging success.

## Lane B — Optional Disposable Runtime

1. Verifikasi tool, kubeconfig context, cluster, namespace, service, owner, and target bukan production.
2. Review query/dashboard/alert diff, traffic scope, window, approval, and notification boundary.
3. Generate bounded synthetic traffic; jangan memakai data/PII customer.
4. Capture metric summary, alert state, timeline UTC, acknowledgement, decision, and post-check.
5. Bila fault injection disetujui, gunakan satu fault terbatas dengan timeout, stop condition, rollback, and cleanup.
6. Redact evidence: simpan query reference dan summary, bukan raw alert payload, token, atau contact detail.

SLO dashboard, error-budget alert, paging, dan reliability decision runtime **belum diverifikasi** tanpa evidence lengkap.

## Stop Conditions

- SLO numerator/denominator atau eligible traffic ambigu.
- `NoData` tidak memiliki policy.
- Alert tidak punya owner/severity/runbook/expiry.
- Target atau namespace bukan disposable.
- Notification akan dikirim ke contact point nyata tanpa approval.
- Traffic scope, rollback, atau evidence retention tidak jelas.

## Acceptance Criteria

- [ ] Contract dan math dapat direview orang lain.
- [ ] Dashboard/alert reference terhubung ke Fase 7.
- [ ] Promotion/change decision dan error-budget policy jelas.
- [ ] On-call coverage, escalation, handoff, fatigue, dan access recovery lengkap.
- [ ] Runtime claim tetap **belum diverifikasi** tanpa execution evidence.
