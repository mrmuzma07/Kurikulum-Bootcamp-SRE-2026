# LAB-01 — Runbook Validation Drill

## Tujuan

Memvalidasi bahwa runbook dapat memandu operator dari symptom ke decision, mitigation, rollback, post-check, dan evidence tanpa improvisasi berbahaya.

## Prasyarat dan Guardrail

Gunakan [runbook workload](../01-runbook-node-disk-certificate-crashloop.md), [runbook MetalLB/evidence](../02-runbook-metallb-topology-evidence-postmortem.md), dan [Modul 8.1](../../modul-8.1-praktik-sre/README.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Static tabletop adalah default. Runtime drill hanya disposable dengan approval, timeout, scoped fault, rollback, cleanup, dan evidence redaction. Jangan melakukan chaos pada production.

## Lane A — Static Tabletop

Pilih dua scenario: node down, disk full, certificate expired, CrashLoopBackOff, atau MetalLB IP failure.

Untuk setiap scenario, reviewer memeriksa:

```text
runbook revision/last-reviewed/expiry
symptom/scope/severity
owner/escalation
preconditions/context safety
read-only checks/query references
decision tree and hypothesis boundary
stop condition
safe mitigation
rollback/data recovery caveat
communication/post-check
evidence fields and retention
```

Isi tabel drill:

| Time UTC | Operator role | Fact observed | Runbook step | Decision | Evidence summary |
|---|---|---|---|---|---|
| `<utc>` | `<role>` | `<summary>` | `<step>` | `<decision>` | `<reference>` |

Reviewer harus menandai langkah ambigu, command terlalu luas, missing stop condition, missing owner, expired revision, dan data recovery caveat.

## Lane B — Optional Disposable Drill

1. Verify context, namespace, target, owner, access recovery, telemetry, and incident channel boundary.
2. Review fault injection and rollback plan; obtain approval.
3. Inject one bounded symptom or use a pre-recorded/static fault scenario.
4. Follow read-only checks; record decision and escalation.
5. Apply safe mitigation, verify telemetry, then rollback/cleanup.
6. Capture before/after health, alert/timeline, action summary, post-check, and runbook gaps.

Runbook drill, alert/paging, mitigation, rollback, and recovery status **belum diverifikasi** without execution evidence.

## Stop Conditions

- Runbook expired atau owner unavailable.
- Context/target ambiguous.
- First action destructive atau tidak reversible.
- Data/PV/database risk tidak memiliki recovery caveat.
- No telemetry, approval, rollback, or cleanup.

## Acceptance Criteria

- [ ] Dua runbook diuji tabletop dengan reviewer berbeda.
- [ ] Read-only first checks dan decision tree dapat diikuti.
- [ ] Stop condition mencegah blast radius tidak terkontrol.
- [ ] Mitigation, rollback, post-check, communication, dan evidence lengkap.
- [ ] Gap menjadi action item dengan owner dan due date.
