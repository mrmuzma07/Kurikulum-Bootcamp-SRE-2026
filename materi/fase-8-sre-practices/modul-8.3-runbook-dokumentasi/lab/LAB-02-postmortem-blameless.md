# LAB-02 — Postmortem Blameless

## Tujuan

Menulis postmortem redacted dari insiden sintetis, memisahkan fact dari hypothesis, dan memverifikasi action item sampai keputusan SLO/error budget.

## Prasyarat dan Guardrail

Gunakan [postmortem contract](../02-runbook-metallb-topology-evidence-postmortem.md) dan [incident response](../../modul-8.1-praktik-sre/02-oncall-incident-response-change-management.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Gunakan incident ID, team, service, query reference, timestamp, dan status placeholder. Jangan menyalin raw logs, raw alert payload, PII, token, private key, password, kubeconfig, atau backup artifact.

## Lane A — Static Simulation

Scenario: `<approved-service>` mengalami latency/error spike setelah `<approved-change>`; alert terpicu, rollback dilakukan, kemudian SLO kembali pada window valid.

### Template

```markdown
# Postmortem <incident-id>

status: <draft-approved-closed>
revision: <git-revision>
impact: <redacted-impact>
detection: <alert-or-report-reference>

## Timeline UTC
- <utc>: <fact and evidence reference>
- <utc>: <decision/action and owner role>

## Contributing Factors
- <system/process factor>

## What Went Well
- <observable-strength>

## What Went Poorly
- <observable-gap>

## Missing Signal
- <telemetry/process gap>

## Action Items
| ID | Action | Owner | Due date UTC | Verification |
|---|---|---|---|---|
| <id> | <specific action> | <team/role> | <date> | <test/evidence> |

## SLO/Error-Budget Decision
<approved follow-up to objective, alert, freeze, or promotion policy>
```

Pastikan timeline membedakan observed fact, hypothesis, decision, action, dan outcome. Blameless berarti tidak mencari individu sebagai root cause; accountability tetap ada melalui owner, due date, dan verification.

## Lane B — Optional Evidence Review

Bila incident drill disposable telah dilakukan:

1. Link policy/runbook revision dan approval tanpa membuka credential.
2. Sertakan alert/query reference, timeline UTC, decision log summary, mitigation/rollback reference, dan post-check.
3. Redact raw logs/payload; simpan checksum atau evidence ID bila perlu.
4. Review action item bersama owner; catat verification result.
5. Update SLO/error-budget decision dan change/runbook revision.

Postmortem runtime, action-item verification, dan SLO follow-up **belum diverifikasi** tanpa evidence aktual.

## Stop Conditions

- Data impact belum direview atau mengandung PII.
- Timeline memakai local time tanpa timezone.
- Root cause menyalahkan individu tanpa system/process factor.
- Action item tidak memiliki owner, due date, atau verification.
- Evidence berisi raw secret/log/payload.

## Acceptance Criteria

- [ ] Impact, detection, timeline UTC, contributing factors, strengths, gaps, missing signal, dan action item ada.
- [ ] Owner, due date, verification, dan SLO/error-budget decision eksplisit.
- [ ] Postmortem redacted dan tidak menyalahkan individu.
- [ ] Action verification memiliki evidence reference atau status `belum diverifikasi`.
