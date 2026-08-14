# 9.3.2 — Readiness Review dan Postmortem

## Graduation Decision

Panel menilai evidence packet pada domain berikut:

| Domain | Evidence minimum |
|---|---|
| Architecture | desired/as-built topology, ownership, dependency, access recovery |
| Rebuild | revision, target allowlist, plan summary, bootstrap/handoff, cleanup |
| Delivery | CI jobs, digest/provenance, GitOps/Argo revision, rollout, smoke |
| Exposure | MetalLB dedicated pool, Ingress traffic, ARP/network outcome |
| Observability | metrics/logs/traces correlation, dashboard, five alert routes |
| Reliability | SLO/error budget dan promotion/incident decision |
| Recovery | Velero/etcd backup, isolated restore, RPO/RTO, validation |
| Operations | three runbooks, Game Day timeline, postmortem, action verification |

Hasil hanya salah satu:

- **ready:** evidence wajib lengkap, outcome memenuhi policy, dan known gaps tidak menghalangi operasi aman;
- **conditional:** platform dapat dilanjutkan dengan risk acceptance, owner, expiry, dan gap remediation;
- **not ready:** evidence wajib hilang, recovery belum terbukti, atau risk tidak terkendali.

Pipeline green, Argo `Synced`/`Healthy`, node `Ready`, alert `Firing`, Velero `Completed`, etcd snapshot, atau dashboard terbuka tidak cukup sendiri.

## Evidence Packet Contract

```yaml
capstone_revision: <revision>
reviewed_at_utc: <timestamp>
reviewers: [<role>]
required_domains:
  architecture: <pass-fail-unknown>
  rebuild: <pass-fail-unknown>
  delivery: <pass-fail-unknown>
  exposure: <pass-fail-unknown>
  observability: <pass-fail-unknown>
  reliability: <pass-fail-unknown>
  recovery: <pass-fail-unknown>
  operations: <pass-fail-unknown>
known_gaps:
  - owner: <role>
    risk: <redacted-summary>
    due_utc: <date>
    verification: <test-reference>
decision: <ready-conditional-not-ready>
```

## Postmortem Blameless

Gunakan struktur:

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
- <observable strength>

## What Went Poorly
- <observable gap>

## Missing Signal
- <telemetry/process gap>

## Action Items
| ID | Action | Owner | Due date UTC | Verification |
|---|---|---|---|---|
| <id> | <specific action> | <team/role> | <date> | <test/evidence> |

## SLO/Error-Budget Decision
<approved follow-up>
```

Postmortem membahas system/process, bukan menyalahkan individu. Namun owner, due date, verification, dan follow-up tetap wajib.

## Review Gates

1. Evidence packet lengkap dan redacted.
2. Setiap claim punya source/revision/time/owner.
3. Known gap memiliki risk acceptance atau remediation.
4. RPO/RTO dan SLO outcome dapat ditelusuri.
5. Action item diverifikasi, bukan hanya dibuat.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Postmortem dan graduation packet tidak boleh berisi raw logs, raw alert payload, PII, kubeconfig, private key, password, backup archive, encryption key, atau decrypted secret.

## Kaitan

- Skenario ada di [9.3.1](01-game-day-oncall-scenarios.md).
- Drill ada di [LAB-01](lab/LAB-01-game-day-drill.md).
- Rebuild dan review ada di [LAB-02](lab/LAB-02-destroy-rebuild-graduation.md).
