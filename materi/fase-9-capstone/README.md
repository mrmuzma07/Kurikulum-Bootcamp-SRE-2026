# Fase 9 — Capstone Project + Simulasi On-Call

> **Durasi:** Minggu 16–18  
> **Tema:** Production-Like Platform di Laptop

Fase ini mengintegrasikan boundary Fase 0–8 menjadi satu platform production-like yang dapat dibangun ulang, dipromosikan melalui GitOps, diamati, dipulihkan, dan diuji melalui Game Day. Target lokal menggunakan Mac Apple Silicon/ARM64 dan OrbStack; pola operasinya tetap mengikuti disiplin on-prem.

## Arsitektur Target

```text
MacBook Air M5 / Apple Silicon
└── OrbStack
    ├── VM infra
    │   ├── OpenTofu state/lock boundary melalui object storage lokal
    │   ├── GitLab runner lokal
    │   └── Ansible control node
    ├── VM servers
    │   └── k3s production multi-node
    └── k3d staging

k3s production:
  demo app → Helm → GitLab CI/GitOps → ArgoCD
  MetalLB L2 + dedicated IP pool → Ingress
  Prometheus → Mimir
  Alloy → Mimir/Loki/Tempo
  Grafana + Alertmanager
  Velero + etcd snapshot
```

Arsitektur ini adalah target desain. Ketersediaan VM, runner, registry, cluster, IP allocation, telemetry, backup, dan notifikasi runtime harus dibuktikan melalui evidence; materi ini tidak mengklaim eksekusi yang belum dilakukan.

## Prasyarat

- [Fase 1 — Container & OrbStack](../fase-1-container-orbstack/README.md).
- [Fase 2 — Kubernetes/k3s](../fase-2-kubernetes/README.md), termasuk MetalLB dan troubleshooting.
- [Fase 3 — OpenTofu](../fase-3-opentofu/README.md).
- [Fase 4 — Ansible](../fase-4-ansible/README.md).
- [Fase 5 — Helm](../fase-5-helm/README.md).
- [Fase 6 — GitOps](../fase-6-gitops/README.md).
- [Fase 7 — Observability](../fase-7-observability/README.md).
- [Fase 8 — Praktik SRE](../fase-8-sre-practices/README.md).

## Modul dan Output

| Modul | Fokus | Output utama |
|---|---|---|
| [9.1 Platform Rebuild](modul-9.1-platform-rebuild/README.md) | OpenTofu → Ansible → k3s | Contract rebuild, topology, handoff, readiness evidence |
| [9.2 Delivery, Observability, Recovery](modul-9.2-delivery-observability/README.md) | CI/GitOps → telemetry → backup | Digest, Argo revision, alert/SLO, restore evidence |
| [9.3 Game Day & Graduation](modul-9.3-game-day-graduation/README.md) | On-call, incident, readiness | Incident packet, postmortem, graduation decision |

## Boundary Ownership

```text
OpenTofu → VM/network/storage metadata dan disposable rebuild scope
Ansible → OS/bootstrap/readiness/patching/k3s host configuration
Kubernetes/k3s → workload, scheduling, service, storage, cluster health
Helm → chart packaging dan release rendering
GitLab CI/GitOps/ArgoCD → validation, provenance, desired state, promotion, reconciliation
MetalLB/Ingress → bare-metal service exposure dan traffic boundary
Fase 7 Observability → metrics, logs, traces, dashboard, alert, SLO evidence
Fase 8 SRE → runbook, on-call, backup/restore, incident command, readiness policy
Fase 9 Capstone → integrasi, rebuild proof, Game Day, graduation decision
```

## Evidence Chain

```text
capstone revision and scope
→ preflight/approval/plan summary
→ infrastructure/bootstrap/delivery action
→ artifact digest and GitOps/Argo revision
→ rollout/MetalLB/access/telemetry evidence
→ SLO/alert/paging decision
→ fault/backup/restore/rebuild outcome
→ mitigation/rollback/recovery
→ postmortem/action-item verification
→ graduation readiness decision
```

Setiap evidence harus memiliki waktu UTC, target, owner, revision/identifier, status summary, dan redaction. Raw kubeconfig, state, plan, log, alert payload, backup archive, dan credential tidak menjadi deliverable.

## Kriteria Kelulusan

- [ ] VM/server cluster dibangun ulang dari nol melalui OpenTofu + Ansible pada disposable scope yang disetujui.
- [ ] Aplikasi production dipromosikan melalui pipeline lint → test → multi-arch build → push → update GitOps; tidak ada `kubectl apply` manual sebagai jalur delivery production.
- [ ] Image memiliki digest immutable dan provenance evidence.
- [ ] MetalLB memberikan IP dari dedicated pool dan Ingress dapat diverifikasi dari traffic evidence.
- [ ] Metrics, logs, traces, dashboard, dan correlation terhubung pada runtime.
- [ ] Minimal lima alert meaningful memiliki owner, severity, query, window, missing-data policy, runbook, route, dan notification evidence.
- [ ] SLO serta dashboard error budget tersedia dan digunakan dalam keputusan.
- [ ] Velero backup dan etcd snapshot berjalan; restore pada target disposable pernah divalidasi.
- [ ] Arsitektur, dependency map, minimal tiga runbook, dan minimal satu postmortem per insiden tersedia.
- [ ] Tujuh Game Day scenario dijalankan secara tabletop atau disposable runtime sesuai approval.
- [ ] Destroy/rebuild dan RPO/RTO memiliki outcome terukur.
- [ ] Graduation decision berstatus `ready`, `conditional`, atau `not ready` dengan known gaps.

## Dua Lane Praktik

**Static lane**: diagram, desired/as-built topology, contract, GitOps flow, immutable artifact policy, alert matrix, SLO math, backup plan, runbook, Game Day matrix, postmortem, rubric, dan readiness checklist. Static lint atau dokumen lengkap hanya membuktikan design review.

**Disposable runtime lane**:

```text
verify tools/context/namespace/cluster/VM target/storage/backup/access
→ target allowlist + plan/diff + approval + maintenance window
→ approved bootstrap/deploy/restore action
→ before/after health and telemetry
→ one bounded fault or scoped restore
→ alert/paging/timeline → runbook → mitigation/rollback/recovery
→ post-check SLO/error budget/RPO/RTO
→ cleanup → redacted evidence → postmortem/action verification
```

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menyimpan access key, password, private key, k3s token, registry credential, webhook, raw state/plan, raw backup, raw log, PII, rendered Secret, atau decrypted value di repository, README, evidence, shell history, CI artifact, dashboard, alert, atau postmortem. `sensitive = true` dan `.gitignore` bukan pengganti secret manager, encryption, key custody, rotation, backup, dan recovery.

Destroy, restore, patch, upgrade, drain, Helm lifecycle, failure injection, dan network change wajib memiliki scope disposable, target allowlist, preflight, plan/diff, approval, stop condition, rollback/recovery, serta cleanup. Jangan memakai `kubectl delete -A`, cluster reset aktif, `--auto-approve` sebagai default, secret melalui `--set`, atau chaos pada production.

## Status Runtime

Materi Fase9: **tersedia secara dokumentasi**. Destroy/rebuild, CI runner/registry, GitOps promotion, MetalLB allocation, observability correlation, lima alert dan notification, Velero restore, etcd restore, Game Day, rollback kurang dari lima menit, RPO/RTO, dan final readiness: **belum diverifikasi** tanpa execution evidence lengkap, redacted, dan dapat diaudit.

## Lanjut

Mulai dari [Modul 9.1 — Platform Rebuild](modul-9.1-platform-rebuild/README.md), lalu lanjutkan ke [Modul 9.2](modul-9.2-delivery-observability/README.md) dan [Modul 9.3](modul-9.3-game-day-graduation/README.md).
