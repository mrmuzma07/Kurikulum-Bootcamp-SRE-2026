# 02 — ArgoCD Install, Application, dan Sync

## Tujuan

Merancang instalasi ArgoCD dan lifecycle Application dengan destination, project, sync policy, health, serta recovery yang eksplisit.

## Instalasi via Helm

Instalasi ArgoCD ke k3s adalah mutation dan harus memiliki cluster/namespace disposable atau scope approved, context verification, access recovery, version pin, backup, dan rollback. Workflow konseptual:

```text
verify kubectl/helm/context
→ render/lint values ArgoCD
→ review CRD/RBAC/Ingress/resource footprint
→ approval
→ Helm install/upgrade --wait --timeout
→ API/UI access and health evidence
```

Jangan menulis password admin, repository credential, atau kubeconfig pada command, values, atau README. Status instalasi tanpa output aktual adalah **belum diverifikasi**.

## Application

Contoh non-secret minimal:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <application-name>
  namespace: <argocd-namespace>
  labels:
    app.kubernetes.io/part-of: <approved-platform>
spec:
  project: <approved-project>
  source:
    repoURL: <approved-gitops-repository>
    path: apps/internal-app/overlays/staging
    targetRevision: <approved-commit-or-tag>
  destination:
    name: <approved-cluster-name>
    namespace: <approved-namespace>
  syncPolicy:
    retry:
      limit: <approved-retry-limit>
```

Periksa:

- source path/revision;
- project repository allowlist;
- destination cluster/namespace allowlist;
- resource tracking dan ownership;
- namespace creation policy;
- health checks dan rollout timeout;
- retry/backoff;
- apakah sync manual atau automated;
- apakah prune/self-heal dibutuhkan dan aman.

## Sync Modes

| Mode | Kegunaan | Risiko |
|---|---|---|
| Manual | review eksplisit sebelum apply | perubahan lebih lambat |
| Automated | dev/staging dengan policy | perubahan cepat dan salah target |
| Self-heal | mengembalikan drift yang tidak sah | dapat melawan break-glass atau controller lain |
| Prune | menghapus resource yang tidak diinginkan | deletion blast radius |

`Synced` berarti desired/live comparison memenuhi kondisi yang diketahui ArgoCD. Ia bukan bukti endpoint, dependency, SLO, atau data safety.

## Health Gate

Gabungkan:

```text
Argo sync/health → Kubernetes rollout → Pod conditions/events
→ Service/Ingress/API smoke test → metrics/logs/traces → SLO decision
```

Redact log dan release metadata sebelum evidence disimpan.

## Rollback dan Recovery

Rollback dapat berupa revert GitOps commit atau sync ke revision sebelumnya. Sebelum rollback periksa migration, CRD, PVC, hook side effect, external API, dan compatibility. Simpan access recovery dan break-glass runbook; jangan bergantung pada UI saja.

## Acceptance Criteria

- [ ] Application spec memiliki source/destination/project yang bounded.
- [ ] Sync policy dan prune/self-heal decision dapat dijelaskan.
- [ ] Install/runtime status tidak diklaim tanpa evidence.
- [ ] Health gate melampaui status ArgoCD.

## Troubleshooting

- Application `Unknown`: cek API, repo source, cluster credential reference, dan controller logs redacted.
- `OutOfSync`: bandingkan desired/live diff dan manual owner.
- Sync timeout: bedakan render, admission, scheduling, rollout, dan external dependency.
- Prune proposal: stop dan review deletion ownership sebelum confirm.

## Kaitan

Gunakan [repo structure](01-gitops-source-of-truth-repo-structure.md), lanjutkan ke [ApplicationSet dan drift](03-applicationset-secrets-drift-selfheal.md), lalu praktikkan [LAB-01](lab/LAB-01-argocd-application-sync.md).

## Catatan SRE

Controller yang bekerja cepat tetap membutuhkan change budget, health signal, dan jalan keluar. Auto-sync tanpa guardrail hanyalah automation dari failure.
