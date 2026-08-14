# Fase 6 — GitOps: GitLab CI + ArgoCD

> **Tujuan fase:** membangun alur delivery pull-based dari commit, CI, artifact immutable, GitOps repository, ArgoCD reconciliation, sampai evidence health dan promotion yang dapat diaudit.

## Durasi dan Modul

Minggu 10–11 — tiga modul dengan static review dan disposable runtime lane.

| Modul | Fokus | Status |
|---|---|---|
| 6.1 | GitLab CI/CD | ✅ Tersedia |
| 6.2 | ArgoCD | ✅ Tersedia |
| 6.3 | End-to-End Flow | ✅ Tersedia |

## Capaian Fase

- [ ] Menjelaskan Git sebagai source of truth desired state dan perbedaan CI dengan CD pull-based.
- [ ] Menulis pipeline GitLab dengan stages, jobs, `rules`, `needs`, artifacts, cache, environment, dan approval.
- [ ] Menjalankan desain lint, test, build multi-arch, scan, sign, dan push image tanpa mutable promotion reference.
- [ ] Menyusun pipeline OpenTofu plan/apply dan Ansible lint/check/run dengan protected approval.
- [ ] Menjelaskan runner shared versus self-hosted OrbStack runner, isolation, privilege, dan ARM64/AMD64.
- [ ] Menjelaskan ArgoCD Application, ApplicationSet, project, sync policy, self-heal, prune, drift, health, dan rollback.
- [ ] Memisahkan application source repository dan GitOps/manifest repository.
- [ ] Merancang secret boundary menggunakan protected variables, workload identity, Sealed Secrets, atau SOPS + age.
- [ ] Menyusun evidence chain dari commit hingga ArgoCD revision dan health/SLO evidence.
- [ ] Mengenal konsep canary/blue-green melalui Argo Rollouts tanpa klaim runtime tanpa telemetry.

> Pipeline hijau, artifact tersedia, atau ArgoCD `Synced`/`Healthy` tidak otomatis membuktikan application correctness atau SLO.

## Alur Utama

```text
Developer push → GitLab CI lint/test/build multi-arch/push registry
→ update GitOps repo via reviewed change
→ ArgoCD pull/reconcile → cluster k3s
→ rollout/readiness/telemetry evidence → promotion/rollback
```

## Rencana Belajar

| Hari | Materi | Praktik |
|---|---|---|
| 1 | [Modul 6.1](modul-6.1-gitlab-ci-cd/README.md), pipeline stages/jobs | Review pipeline dan job graph |
| 2 | Runner, artifact/cache, IaC/Ansible gates | [LAB-01](modul-6.1-gitlab-ci-cd/lab/LAB-01-ci-lint-test-build-push.md) |
| 3 | [Modul 6.2](modul-6.2-argocd/README.md), GitOps repository | [LAB-01](modul-6.2-argocd/lab/LAB-01-argocd-application-sync.md) |
| 4 | Application, ApplicationSet, drift, secret boundary | [LAB-02](modul-6.2-argocd/lab/LAB-02-drift-selfheal-applicationset.md) |
| 5 | [Modul 6.3](modul-6.3-end-to-end-flow/README.md), promotion dan evidence | [LAB-01](modul-6.3-end-to-end-flow/lab/LAB-01-end-to-end-gitops-flow.md) |
| 6 | Progressive delivery dan rollback | [LAB-02](modul-6.3-end-to-end-flow/lab/LAB-02-canary-blue-green-introduction.md) + evaluasi |

## Dua Lane Praktik

### Static lane

```text
pipeline/application design → YAML and policy review → render/validation review
→ evidence schema → promotion and rollback runbook
```

Gunakan lane ini bila GitLab, runner, registry, ArgoCD, atau cluster tidak tersedia. Static review tidak membuktikan job, push, sync, self-heal, readiness, atau SLO.

### Disposable runtime lane

```text
verify tools/context/namespace → review diff and approval
→ CI job or controlled local equivalent → artifact evidence
→ ArgoCD sync → rollout/health evidence → drift/rollback drill
```

Runtime hanya pada project, runner, registry, cluster, namespace, dan repository disposable yang scope-nya jelas. Jangan mengubah production atau context yang belum diverifikasi.

## Boundary Ownership

| Layer | Tanggung jawab |
|---|---|
| OpenTofu | VM, network, storage, metadata non-secret |
| Ansible | OS bootstrap, hardening, readiness, konfigurasi host/k3s |
| Kubernetes/k3s | scheduling, service discovery, storage, cluster health |
| Helm | chart packaging, templating, release rendering |
| GitLab CI | validation, build, artifact publication, controlled IaC/configuration jobs |
| GitOps repository | desired application state dan promotion history |
| ArgoCD | pull-based sync, reconciliation, drift detection, self-heal |
| Observability/Fase 7 | metrics, logs, traces, SLO evidence |

## Deliverables

1. Pipeline GitLab CI untuk lint, test, build multi-arch, scan, dan push artifact.
2. Pipeline OpenTofu plan/apply dan Ansible lint/check dengan approval serta protected environment.
3. Struktur app repository dan GitOps repository yang memiliki ownership jelas.
4. ArgoCD Application dan ApplicationSet untuk target staging/prod yang allowlist-nya eksplisit.
5. Drift/self-heal drill dan runbook rollback berbasis Git revision.
6. Evidence chain redacted dari commit, pipeline, image digest, chart/values revision, ArgoCD revision, target, dan health outcome.
7. Nilai kuis minimal **80%** pada setiap modul.

## Guardrail Fase

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menyimpan GitLab PAT, deploy token, registry credential, SSH private key, kubeconfig, ArgoCD password, age private key, raw Kubernetes Secret, decrypted SOPS content, raw OpenTofu state/plan, atau raw CI artifact.
- Gunakan protected/masked variables, secret manager, OIDC/workload identity, Sealed Secrets, atau SOPS + age sesuai boundary. Ciphertext tidak menggantikan key management.
- Jangan mencetak `CI_JOB_TOKEN`, deploy token, kubeconfig, repository credential, atau decrypted secret ke log, cache, artifact, dan evidence.
- `tofu plan`, Ansible `--check/--diff`, dan CI dry run bukan jaminan zero side effect. Apply, SSH changes, registry push, dan cluster sync memerlukan scope, approval, recovery, serta rollback.
- Jangan memakai `-auto-approve` sebagai default, `kubectl delete -A`, manual drift edit di production, atau auto-prune tanpa review ownership/deletion.
- ArgoCD sync/self-heal bukan SLO dan tidak membalikkan migration, CRD, PVC, atau external side effect secara otomatis.
- Runtime CI, runner, registry, ArgoCD, sync, self-heal, ApplicationSet, progressive delivery, dan promotion berstatus **belum diverifikasi** tanpa evidence aktual.

## Kaitan

- [Fase 1 — Container & OrbStack](../fase-1-container-orbstack/README.md) menyediakan image multi-arch dan registry.
- [Fase 2 — Kubernetes](../fase-2-kubernetes/README.md) menyediakan cluster, context safety, rollout, dan troubleshooting.
- [Fase 3 — OpenTofu](../fase-3-opentofu/README.md) menyediakan plan, state, provider, dan promotion boundary.
- [Fase 4 — Ansible](../fase-4-ansible/README.md) menyediakan host/k3s readiness.
- [Fase 5 — Helm](../fase-5-helm/README.md) menyediakan chart dan OCI artifact.
- Fase 7 menyediakan telemetry dan SLO gate.

## Catatan SRE

GitOps mengurangi drift konfigurasi dan membuat desired state dapat diaudit; GitOps tidak menghapus failure domain. Selalu pisahkan pipeline success, sync status, workload health, telemetry, dan SLO.

## Status Runtime

Materi Fase6: **tersedia**. GitLab pipeline, runner, registry push, ArgoCD installation/sync, drift self-heal, ApplicationSet, secret controller, progressive delivery, dan production promotion: **belum diverifikasi** tanpa execution evidence.
