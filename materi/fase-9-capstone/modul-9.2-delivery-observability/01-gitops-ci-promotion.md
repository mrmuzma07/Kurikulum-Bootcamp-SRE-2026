# 9.2.1 — CI, GitOps, dan Promotion Evidence

## Rantai Delivery

```text
source commit
→ lint/unit/integration test
→ multi-arch build
→ vulnerability/provenance checks
→ push immutable digest
→ reviewed GitOps change
→ ArgoCD sync/reconcile
→ Helm render/release
→ rollout/readiness/smoke
→ telemetry window
→ SLO/error-budget decision
```

Setiap tahap memiliki identifier, revision, target, owner, UTC timestamp, dan status. Pipeline green hanya membuktikan job yang dijalankan berhasil; ia tidak membuktikan target, traffic, dependency, application health, atau SLO.

## Artifact Policy

Image production harus dipromosikan berdasarkan digest, bukan tag mutable. Simpan digest dan provenance summary, bukan credential atau raw CI log. Multi-arch manifest harus mencantumkan platform yang memang diuji dan alasan jika ada platform yang dikecualikan.

Trivy atau scanner lain adalah signal supply-chain, bukan bukti keamanan total. Exception wajib memiliki severity, owner, due date, compensating control, dan expiry.

## GitOps Boundary

GitLab CI memvalidasi dan mempublikasikan artifact/desired-state change. Merge request atau mekanisme promotion yang disetujui mengubah GitOps repository. ArgoCD membaca revision yang direview dan melakukan reconciliation ke cluster production.

```text
CI tidak memiliki kubeconfig production
CI tidak melakukan kubectl apply production
ArgoCD memiliki boundary reconciliation yang diaudit
```

Manual edit production adalah break-glass, bukan jalur delivery; drift harus dikembalikan melalui repository dan follow-up review.

## Protected Promotion

Promotion review memeriksa:

- digest dan provenance;
- chart/values diff dan compatibility;
- staging evidence dari k3d;
- migration/data caveat;
- capacity, PDB, and dependency;
- SLO/error-budget window;
- rollback code/config dan recovery data;
- owner, approval, maintenance window, dan stop condition.

Canary atau blue-green membutuhkan traffic boundary, stable/candidate isolation, metric gate, pause/abort policy, dan capacity review. `--atomic` tidak membalikkan migration atau external side effect.

## Rollback

Rollback kurang dari lima menit adalah target yang harus diukur dari declaration sampai traffic sehat, bukan klaim dari command selesai. Bedakan:

1. rollback desired state;
2. Argo reconciliation;
3. rollout/readiness;
4. traffic smoke;
5. telemetry/SLO recovery;
6. data/schema recovery.

Git revert tidak otomatis mengembalikan data migration, queue, external API, atau side effect.

## Evidence Minimum

```yaml
source_revision: <commit>
ci_run: <run-id>
artifact_digest: <sha256-summary>
provenance: <attestation-reference>
gitops_revision: <commit>
argocd_revision: <revision>
target: <staging-or-disposable-production>
rollout: <summary>
smoke: <redacted-status>
telemetry_window: <utc-window>
slo_decision: <promote-pause-rollback>
approval: <approval-reference>
```

## Static dan Runtime

Static lane meninjau pipeline YAML, policy, promotion contract, dan rollback decision tree. Runtime lane harus menunjukkan execution evidence dari CI/registry/GitOps/Argo/rollout/traffic/telemetry. `Synced`, `Healthy`, atau Helm `deployed` saja tidak cukup.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan mencetak `CI_JOB_TOKEN`, deploy token, kubeconfig, registry credential, decrypted secret, atau raw artifact. Jangan memakai secret melalui `--set`; gunakan approved secret management reference.

## Kaitan

- Praktik chain ada di [LAB-01](lab/LAB-01-end-to-end-promotion.md).
- Signal dan recovery evidence dibahas di [9.2.2](02-observability-slo-backup-evidence.md).
- Incident rollback dikonsumsi [Modul 9.3](../modul-9.3-game-day-graduation/README.md).
