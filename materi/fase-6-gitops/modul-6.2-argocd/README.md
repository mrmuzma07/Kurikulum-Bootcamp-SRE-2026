# Modul 6.2 — ArgoCD

> **Tujuan akhir:** memahami ArgoCD sebagai reconciler pull-based yang menerapkan desired state dari Git dengan kontrol source, destination, sync, drift, health, dan secret boundary.

## Capaian Modul

- [ ] Menjelaskan desired state, live state, reconciliation, `Synced`, `OutOfSync`, `Healthy`, dan `Degraded`.
- [ ] Membuat desain `Application`, `ApplicationSet`, `AppProject`, source revision, destination, dan sync policy.
- [ ] Menilai auto-sync, self-heal, prune, retry, sync waves, hooks, dan blast radius.
- [ ] Memisahkan app repository dan GitOps/manifest repository.
- [ ] Menjelaskan Sealed Secrets atau SOPS + age tanpa menyimpan key/decrypted content.
- [ ] Merancang drift, break-glass, rollback, bootstrap, dan recovery.

## Rencana 2 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [Source of truth dan repo](01-gitops-source-of-truth-repo-structure.md), [Application/sync](02-argocd-install-application-sync.md) | [LAB-01](lab/LAB-01-argocd-application-sync.md) |
| 2 | [ApplicationSet, secret, drift](03-applicationset-secrets-drift-selfheal.md) | [LAB-02](lab/LAB-02-drift-selfheal-applicationset.md) + [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 2.4 untuk context, rollout, health, dan troubleshooting.
- Fase 4 untuk k3s readiness.
- Fase 5 untuk chart, values, OCI, dan release safety.
- Modul 6.1 untuk CI artifact dan promotion gate.

## Acceptance Criteria

- [ ] Desired state, live state, source revision, destination, dan ownership dapat ditelusuri.
- [ ] Application/AppProject/ApplicationSet memiliki allowlist dan namespace boundary.
- [ ] Sync/prune/self-heal memiliki approval, deletion, retry, dan recovery policy.
- [ ] Secret mechanism tidak menaruh plaintext credential atau private key di Git.
- [ ] Drift drill dan rollback memiliki evidence design.

## Guardrail

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan install ArgoCD, mengaktifkan auto-prune, self-heal, atau drift edit pada production tanpa scope, access recovery, approval, dan rollback. ArgoCD `Healthy` bukan SLO.

## Troubleshooting

- `OutOfSync`: bedakan perubahan Git, manual drift, controller default, ignoreDifferences, dan generated output.
- `Degraded`: periksa resource health, rollout, events, image pull, dependency, dan telemetry.
- Sync gagal: klasifikasikan source/render, permission, admission, scheduling, atau application failure.
- Prune berbahaya: hentikan; review ownership, resource tracking, project allowlist, dan deletion policy.
- Self-heal loop: periksa controller lain, mutating webhook, non-deterministic template, dan ignoreDifferences.

## Kaitan

- [Modul 5.1 — Helm](../../fase-5-helm/modul-5.1-helm-fundamental/README.md)
- [Modul 5.2 — production chart](../../fase-5-helm/modul-5.2-chart-production/README.md)
- [Modul 6.1 — GitLab CI](../modul-6.1-gitlab-ci-cd/README.md)
- [Modul 6.3 — End-to-End](../modul-6.3-end-to-end-flow/README.md)
- Fase 7 untuk telemetry dan SLO.

## Catatan SRE

Reconciliation membuat perubahan manual terlihat, tetapi controller tidak dapat menebak apakah side effect database atau external API aman untuk diulang. Ownership dan recovery tetap tanggung jawab operator.

## Status Runtime

Materi tersedia. ArgoCD install, Application sync, ApplicationSet, drift, self-heal, secret controller, dan rollback belum diverifikasi tanpa evidence runtime.
