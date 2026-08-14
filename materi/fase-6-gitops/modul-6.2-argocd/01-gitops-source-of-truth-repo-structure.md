# 01 — GitOps Source of Truth dan Struktur Repository

## Tujuan

Memisahkan source code aplikasi dari desired state deployment dan membuat promotion history yang dapat diaudit.

## GitOps Model

```text
app repository
  → CI lint/test/build/scan/sign/push image digest
  → reviewed change ke GitOps repository
  → ArgoCD pull desired state
  → reconcile live cluster
```

Git adalah source of truth untuk desired state, bukan log semua live state. Cluster dapat drift karena perubahan manual, controller, admission, atau dependency; ArgoCD membandingkan desired dan live state.

## App Repo versus GitOps Repo

| Repository | Isi | Owner utama |
|---|---|---|
| App repo | source, test, Dockerfile, chart source bila ownership disepakati | developer/app team |
| GitOps repo | environment values/manifests, Application, promotion commit, policy | platform/SRE/release owner |
| IaC repo | OpenTofu dan Ansible source | platform/infrastructure owner |

Pemisahan mencegah CI aplikasi langsung memiliki kubeconfig production. CI sebaiknya membuat reviewed merge request atau update artifact reference yang terkontrol.

## Struktur Contoh

```text
gitops-repo/
├── apps/
│   └── internal-app/
│       ├── base/
│       └── overlays/
│           ├── staging/
│           └── production/
├── argocd/
│   ├── projects/
│   ├── applications/
│   └── applicationsets/
└── policies/
```

Dengan Helm, GitOps repo dapat menyimpan chart reference dan values non-secret:

```yaml
source:
  repoURL: <approved-gitops-repository>
  path: apps/internal-app
  targetRevision: <approved-commit-or-branch>
  helm:
    valueFiles:
      - values-staging.yaml
```

Pinning commit lebih reproducible daripada branch bergerak. Jika memakai branch untuk development, production promotion harus menggunakan reviewed commit/tag.

## Promotion Contract

```text
app commit → pipeline ID → image digest → chart version
→ GitOps commit → reviewer/approval → target environment → Argo revision
```

Jangan mengubah `latest` atau branch production secara diam-diam. Setiap update harus menjelaskan artifact, environment, compatibility, owner, dan rollback reference.

## Secret Boundary

GitOps repository hanya boleh menyimpan:

- reference secret;
- encrypted value dengan key di luar repository, bila policy mengizinkan;
- External Secrets/Sealed Secrets/SOPS metadata non-secret.

Ciphertext SOPS tanpa key management, backup, rotation, dan recovery bukan solusi lengkap.

## Acceptance Criteria

- [ ] Ownership app/GitOps/IaC jelas.
- [ ] Production desired state hanya berubah melalui review.
- [ ] Artifact reference immutable atau dipin.
- [ ] Secret boundary dan recovery dijelaskan.

## Troubleshooting

- Argo source tidak ditemukan: periksa repo URL, revision, path, credential reference, dan project allowlist.
- Promotion tidak terdeteksi: cek commit target, webhook/polling, branch, dan Argo refresh.
- Desired state ambigu: hindari dua controller mengelola field/resource yang sama.

## Kaitan

Lanjutkan ke [Application dan sync](02-argocd-install-application-sync.md), serta gunakan chart dari [Fase 5](../../fase-5-helm/README.md).

## Catatan SRE

Source of truth yang tidak memiliki ownership, review, dan rollback hanya memindahkan konfigurasi manual ke tempat lain.
