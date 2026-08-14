# Modul 6.3 — End-to-End Flow

> **Tujuan akhir:** menghubungkan source commit, CI artifact, GitOps promotion, ArgoCD reconciliation, rollout health, telemetry, dan rollback dalam satu evidence chain.

## Capaian Modul

- [ ] Menelusuri commit → pipeline → image digest → GitOps revision → ArgoCD sync → health.
- [ ] Memisahkan app repo dan manifest repo serta menetapkan ownership promotion.
- [ ] Merancang staging → production approval dan protected environment.
- [ ] Mengklasifikasikan CI, registry, manifest, sync, rollout, application, dan SLO failure.
- [ ] Menyusun rollback Git/ArgoCD dengan caveat migration, CRD, PVC, dan external side effect.
- [ ] Mengenal canary/blue-green dan boundary Argo Rollouts.

## Rencana 2 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [Promotion dan evidence](01-promotion-flow-evidence.md) | [LAB-01](lab/LAB-01-end-to-end-gitops-flow.md) |
| 2 | [Progressive delivery dan rollback](02-progressive-delivery-rollback.md) | [LAB-02](lab/LAB-02-canary-blue-green-introduction.md) + [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 6.1 dan 6.2 selesai.
- Fase 5 chart, values, OCI, test, dan rollback.
- Fase 7 dibutuhkan untuk telemetry/SLO runtime gate.

## Acceptance Criteria

- [ ] Evidence chain memiliki commit, pipeline, artifact digest, manifest revision, target, approval, sync, rollout, dan health.
- [ ] Promotion tidak mengandalkan perubahan langsung ke cluster production.
- [ ] Failure taxonomy dan stop condition dapat digunakan operator.
- [ ] Rollback dibedakan dari data recovery.
- [ ] Canary/blue-green memiliki metric, pause, abort, capacity, dan recovery design.

## Guardrail

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Git revert atau ArgoCD rollback tidak otomatis membalikkan migration, CRD, PVC, external API, atau side effect hook. Jangan mengklaim progressive delivery tanpa telemetry dan evidence.

## Troubleshooting

- CI hijau tetapi app gagal: cek manifest revision, sync, rollout, dependency, telemetry, dan SLO.
- Argo synced tetapi endpoint gagal: `Synced` hanya desired/live comparison; lanjutkan health gate.
- Rollback tidak memulihkan app: klasifikasikan image, config, data schema, external dependency, dan observability.
- Canary tidak progres: cek metric analysis, traffic routing, capacity, pause, dan abort policy.

## Kaitan

- [Modul 6.1](../modul-6.1-gitlab-ci-cd/README.md)
- [Modul 6.2](../modul-6.2-argocd/README.md)
- [Fase 5 — Helm](../../fase-5-helm/README.md)
- Fase 7 untuk metrics, logs, traces, dan SLO.
- Fase 8 untuk incident response, error budget, dan runbook.

## Catatan SRE

Delivery pipeline adalah control plane perubahan, bukan pengganti observability. Ukur lead time, failure rate, rollback rate, deployment health, dan dampak service.

## Status Runtime

Materi tersedia. End-to-end CI → GitOps → ArgoCD, rollback, canary, blue-green, dan SLO gate belum diverifikasi tanpa evidence runtime.
