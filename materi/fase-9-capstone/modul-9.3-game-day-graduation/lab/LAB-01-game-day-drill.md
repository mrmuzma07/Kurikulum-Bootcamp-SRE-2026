# LAB-01 — Game Day Drill dan Postmortem

> **Target:** menjalankan tabletop atau disposable drill untuk satu skenario, menghubungkan alert sampai recovery, dan menghasilkan postmortem blameless redacted.

## Prasyarat

- [9.3.1 — Skenario](../01-game-day-oncall-scenarios.md) dan [9.3.2 — Readiness](../02-readiness-review-postmortem.md) dibaca.
- Minimal tiga runbook tersedia dan memiliki stop condition.
- On-call roles, escalation, communication boundary, target, approval, maintenance window, and rollback authority ditetapkan.
- Observability/notification evidence dari Modul 9.2 tersedia atau diberi status belum diverifikasi.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Satu iterasi hanya satu fault dan satu disposable target. Jangan melakukan chaos/failure injection pada production. Jangan mengirim raw logs, alert payload, PII, password, token, private key, atau kubeconfig ke channel incident/postmortem.

## Evidence Contract

```yaml
incident_id: <incident-id>
scenario: <gd-01-to-gd-07>
mode: <tabletop-or-disposable-runtime>
revision: <git-revision>
target: <approved-target>
roles: <role-summary>
severity: <sev>
ack_utc: <timestamp-or-unknown>
timeline_utc: <redacted-events>
alert_paging: <summary>
runbook: <runbook-id>
mitigation: <summary>
rollback_recovery: <summary>
postcheck: <health-and-slo-summary>
cleanup: <pass-fail-unknown>
postmortem: <reference>
```

## Langkah

### 1. Brief dan Freeze

IC mengonfirmasi scenario, scope, objective, stop condition, communications cadence, and rollback. Semua participant menyatakan access/recovery path. Untuk tabletop, tandai setiap expected action sebagai simulation.

### 2. Detection dan Declaration

Mulai dari alert/report yang disetujui. Catat UTC, signal reference, severity, acknowledgement, IC, and escalation. Jangan menambah fault atau traffic di luar scope.

### 3. Read-Only Triage

Periksa target, recent revision, nodes/workload, traffic, metrics/logs/traces, dependencies, backup, and network. Gunakan query/runbook reference; jangan copy raw data. IC memilih mitigation yang reversible dan bounded.

### 4. Mitigation dan Recovery

Operations lead menjalankan action yang disetujui. Ukur rollback atau recovery dari decision/start sampai post-check. Bila stop condition tercapai, hentikan action dan eskalasi; jangan memaksa success.

### 5. Post-Check dan Cleanup

Validasi health, traffic, telemetry, alert clear/ack, SLO/error budget, data consistency, and target cleanup. Catat unknown sebagai unknown. Runtime status hanya `terverifikasi` bila evidence chain utuh.

### 6. Postmortem

Dalam 24 jam latihan, isi:

- impact redacted;
- detection dan missing signal;
- timeline UTC;
- contributing system/process factors;
- what went well/poorly;
- action items dengan owner, due date, verification;
- SLO/error-budget decision.

Review bersama participant tanpa menyalahkan individu.

## Acceptance Criteria

- [ ] Roles, severity, acknowledgement, escalation, scope, timeout, dan stop condition ada.
- [ ] Read-only checks mendahului perubahan.
- [ ] Alert/paging, runbook, mitigation, rollback/recovery, post-check, dan cleanup tercatat.
- [ ] Timeline UTC dan postmortem redacted tersedia.
- [ ] Actual RTO/SLO hanya diisi bila benar-benar diukur; selain itu `unknown`.
- [ ] Tidak ada security/guardrail violation.

## Troubleshooting

- **Tidak ada alert:** gunakan tabletop atau catat notification belum diverifikasi; jangan mengarang receipt.
- **Operator berbeda pendapat:** IC menetapkan decision log dan escalation.
- **Mitigasi memperburuk impact:** trigger stop condition, rollback, dan communication; jangan memperluas scope.
- **Postmortem menyalahkan orang:** ubah menjadi contributing system/process factor dan tetap pertahankan accountability action item.

## Lanjut

Lanjutkan ke [LAB-02 — Destroy/Rebuild Graduation](LAB-02-destroy-rebuild-graduation.md).
