# 01 — Promotion Flow dan Evidence Immutable

## Tujuan

Menghubungkan source commit, pipeline, artifact, GitOps revision, ArgoCD reconciliation, rollout, telemetry, dan keputusan promotion menjadi evidence chain yang dapat diaudit.

## Alur End-to-End

```text
source commit
→ CI lint/test/chart validation
→ multi-arch build
→ scan/sign/provenance
→ immutable image digest
→ reviewed GitOps change
→ ArgoCD source revision
→ cluster rollout/readiness
→ smoke test + telemetry
→ staging decision
→ production approval/promotion
```

CI aplikasi sebaiknya tidak memegang kubeconfig production. CI mempublikasikan artifact dan membuat reviewed change pada GitOps repository. ArgoCD menarik desired state dari repository tersebut.

## Evidence Chain

| Tahap | Evidence minimum | Batasan |
|---|---|---|
| Source | commit SHA, author/reviewer, change scope | bukan bukti build |
| CI | pipeline/job ID, rules, runner architecture, test summary | hanya scope job |
| Artifact | image digest, platform, SBOM/sign/provenance reference | bukan bukti deployment |
| GitOps | commit/MR, values/manifest diff, reviewer | desired state saja |
| ArgoCD | application, revision, sync outcome, target | bukan SLO |
| Rollout | deployment revision, readiness, events summary | bukan traffic health penuh |
| App | smoke test, error/latency summary | perlu window representatif |
| SLO | metric query/reference, window, decision | bergantung telemetry Fase 7 |

Semua evidence harus redacted, memiliki retention/access policy, dan tidak memuat token, kubeconfig, rendered Secret, raw CI log, raw plan/state, atau private key.

## Promotion Staging → Production

Promotion bukan sekadar menyalin tag. Reviewer harus memeriksa:

- artifact digest dan compatibility;
- chart/version dan migration plan;
- GitOps diff dan target namespace/cluster;
- staging health dan telemetry window;
- open incident/change freeze;
- approval production dan rollback reference;
- owner serta communication path.

Production environment harus protected dan serialized. Gunakan reviewed commit/tag sebagai desired state; jangan mengubah `latest` secara diam-diam.

## Failure Taxonomy

1. **CI failure:** lint/test/build/scan gagal; tidak ada promotion.
2. **Registry failure:** push atau pull digest gagal; cek auth/quota/network tanpa mencetak credential.
3. **Manifest failure:** render/schema/policy invalid; revert/fix GitOps.
4. **Argo sync failure:** source, permission, admission, atau resource apply gagal.
5. **Rollout failure:** scheduling, image pull, readiness, dependency, atau capacity.
6. **Application failure:** endpoint/error/latency tidak memenuhi acceptance.
7. **SLO failure:** telemetry menunjukkan budget/error objective gagal meski controller Healthy.

Klasifikasi menentukan siapa owner, tindakan stop, dan evidence berikutnya.

## Recovery

Revert GitOps commit atau promote revision sebelumnya hanya setelah menilai migration, CRD, PVC, hook, external API, dan schema compatibility. Git/Argo rollback tidak otomatis membalikkan data migration atau side effect eksternal.

Jika cluster perlu direbuild, alur recovery adalah OpenTofu metadata → Ansible host/k3s readiness → ArgoCD bootstrap → GitOps rehydration. Credential dan state tetap berada pada backend/manager yang disetujui.

## Acceptance Criteria

- [ ] Chain commit sampai SLO memiliki identifier dan owner.
- [ ] Staging/prod gate, protected environment, approval, dan rollback jelas.
- [ ] Pipeline green, digest, Synced, Healthy, rollout, dan SLO dibedakan.
- [ ] Failure taxonomy memiliki stop/action/evidence.
- [ ] Tidak ada secret/raw artifact pada evidence.

## Kaitan

Gunakan [Modul 6.1](../modul-6.1-gitlab-ci-cd/README.md), [Modul 6.2](../modul-6.2-argocd/README.md), dan [Fase 7 Observability](../../../../fase-7-observability/README.md).
