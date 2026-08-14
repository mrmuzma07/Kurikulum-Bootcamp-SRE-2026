# LAB-02 — Incident Response dan Change Freeze

## Tujuan

Melakukan tabletop incident dari detection sampai postmortem, kemudian menentukan freeze, rollback, dan emergency change review.

## Prasyarat dan Guardrail

Gunakan [incident/change theory](../02-oncall-incident-response-change-management.md), runbook [Modul 8.3](../../modul-8.3-runbook-dokumentasi/README.md), dan telemetry [Fase 7](../../../fase-7-observability/README.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Static lane tidak memicu incident. Runtime hanya disposable, bounded, approved, timeout-limited, dan memiliki rollback/cleanup.

## Lane A — Static Tabletop

### Scenario

`<incident-id>`: error rate aplikasi meningkat setelah `<approved-change-reference>`. Alert burn rate aktif, sebagian request timeout, dan pipeline sebelumnya green.

Isi:

| Tahap | Role | Timestamp UTC | Observed fact | Decision/action | Evidence |
|---|---|---|---|---|---|
| detection | `<role>` | `<utc>` | `<summary>` | declare/not declare | `<query-ref>` |
| declaration | IC | `<utc>` | `<summary>` | severity/scope | `<incident-ref>` |
| triage | ops lead | `<utc>` | `<summary>` | hypothesis | `<redacted-summary>` |
| mitigation | ops lead | `<utc>` | `<summary>` | bounded action | `<change-ref>` |
| rollback/fix | approver | `<utc>` | `<summary>` | revision decision | `<revision>` |
| monitoring | IC | `<utc>` | `<summary>` | acceptance | `<slo-ref>` |
| closure | IC | `<utc>` | `<summary>` | close/follow-up | `<postmortem-ref>` |

### Change freeze dan CAB ringan

Tulis change record:

```text
change_id: <approved-change-id>
classification: <normal-high-risk-emergency>
scope: <bounded-scope>
risk_impact: <summary>
freeze_status: <active-or-not-active>
maintenance_window: <utc-window>
peer_review: <roles>
rollback: <revision-or-recovery-reference>
data_migration_caveat: <approved-caveat>
exception_approver: <approved-role>
post_change_review: <utc-deadline>
```

Tentukan apakah promotion harus dipause, apakah emergency change dibenarkan, siapa IC/approver, dan bagaimana komunikasi tanpa raw payload.

### Blameless review

Catat contributing factors, signal yang hilang, what went well/poorly, action item, owner, due date, verification, dan keputusan SLO/error budget. Jangan menyebut individu sebagai root cause.

## Lane B — Optional Disposable Drill

1. Verifikasi target/namespace/owner, context, alert boundary, and access recovery.
2. Review fault injection, change, rollback, and cleanup plan; minta approval.
3. Inject satu fault bounded atau deploy synthetic bad revision.
4. Amati telemetry/alert/paging; catat declaration dan role assignment.
5. Mitigate/rollback sesuai runbook; jangan menjalankan destructive command luas.
6. Monitor representative window, cleanup, dan tulis post-drill review.

Incident drill, paging, mitigation, rollback, dan SLO outcome **belum diverifikasi** tanpa evidence aktual.

## Stop Conditions

- Production target atau context tidak jelas.
- Tidak ada IC/approver/rollback.
- Fault dapat memengaruhi data persistent atau external service.
- Alert/paging boundary tidak tersedia.
- Migration atau external side effect tidak dapat dipulihkan.

## Acceptance Criteria

- [ ] Timeline detection hingga closure lengkap dan UTC.
- [ ] Freeze/CAB/emergency change decision memiliki owner dan expiry.
- [ ] Mitigation dan rollback dibedakan dari resolution.
- [ ] Postmortem/action verification direncanakan.
- [ ] Semua runtime evidence akan diredáksi dan status jujur.
