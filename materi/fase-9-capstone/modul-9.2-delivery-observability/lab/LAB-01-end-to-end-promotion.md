# LAB-01 — End-to-End Promotion Evidence

> **Target:** mengikuti satu perubahan aplikasi dari commit sampai GitOps, ArgoCD, traffic, telemetry, dan keputusan promotion/rollback.

## Prasyarat

- Handoff Modul 9.1 berstatus approved atau static equivalent.
- Demo app dan chart Helm memiliki test, health endpoint, resource policy, serta values non-secret.
- k3d staging dan disposable k3s production boundary terdokumentasi.
- GitLab CI/GitOps/ArgoCD owner, protected approval, rollback, dan migration caveat tersedia.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

CI tidak boleh memiliki kubeconfig production. Jangan mencetak `CI_JOB_TOKEN`, deploy token, registry credential, rendered Secret, atau raw CI artifact. Tidak ada `kubectl apply` manual sebagai production delivery path. Jangan memakai `--set` untuk secret.

## Evidence Contract

```yaml
lab: LAB-01
source_revision: <commit>
ci_run: <run-id>
checks: <lint-test-build-scan-summary>
artifact_digest: <sha256-summary>
provenance_ref: <attestation-id>
gitops_revision: <commit>
argocd_revision: <revision>
target: <staging-or-disposable-production>
rollout: <summary>
traffic: <redacted-smoke-summary>
telemetry_window_utc: <window>
slo_decision: <promote-pause-rollback>
rollback_elapsed: <measured-or-unknown>
known_gaps: [<gap>]
```

## Langkah

### 1. Source dan CI

Buat perubahan kecil yang dapat dibalik. Review diff, tests, migration caveat, resource/capacity impact, and ownership. Jalankan pipeline lint → test → multi-arch build → scan/provenance → push. Simpan digest dan summary, bukan raw log.

Jika job gagal, klasifikasikan failure dan perbaiki melalui commit baru. Jangan menandai promotion sukses berdasarkan job yang belum selesai atau artifact tag mutable.

### 2. Staging k3d

Deploy melalui jalur staging yang disepakati, bukan manual command yang mengaburkan evidence. Verifikasi chart render, digest, rollout, readiness, smoke, and telemetry. Catat cluster/context identifier tanpa kubeconfig.

### 3. GitOps Promotion

Buat GitOps change/MR yang mengubah referensi image ke digest immutable. Review values, chart, migration, policy, and diff. Setelah approval, biarkan ArgoCD membaca revision dan reconcile sesuai boundary.

Capture commit, Argo revision, sync/health summary, rollout, traffic, and telemetry. `Synced`/`Healthy` harus dikaitkan dengan outcome aplikasi.

### 4. Promotion Decision

Buka telemetry window yang disetujui. Evaluasi error ratio, latency, saturation, dependency, alert, error budget, and traffic eligibility. Pilih `promote`, `pause`, atau `rollback` dengan owner dan alasan.

Progressive delivery wajib memiliki stable/candidate traffic boundary, metric gate, pause/abort, capacity, and migration caveat. `--atomic` tidak menyelesaikan external side effect.

### 5. Rollback Drill pada Disposable Scope

Jika drill rollback dilakukan, gunakan bad release yang telah disetujui pada disposable target. Ukur dari incident declaration sampai traffic dan telemetry pulih; jangan hanya mengukur command. Bedakan GitOps revert, Argo sync, rollout, smoke, dan SLO recovery.

### 6. Close Packet

Isi evidence contract dan tandai setiap tahap `pass`, `fail`, atau `unknown`. Bila runtime tools/runner/registry/Argo tidak tersedia, tulis **belum diverifikasi**, bukan contoh output sebagai fakta.

## Acceptance Criteria

- [ ] Source, CI, digest/provenance, GitOps revision, Argo revision, rollout, smoke, telemetry, dan SLO decision terhubung.
- [ ] Production path tidak memakai manual `kubectl apply`.
- [ ] Promotion memakai immutable digest dan protected approval.
- [ ] Rollback target <5 menit hanya dicatat bila benar-benar diukur end-to-end.
- [ ] Migration/data caveat, capacity, stable/candidate boundary, pause/abort, dan cleanup ada.
- [ ] Secret dan raw artifact tidak masuk evidence.

## Troubleshooting

| Gejala | Tindakan |
|---|---|
| Build lulus tetapi digest tidak diketahui | Stop promotion; cari provenance/artifact evidence |
| Argo `Synced` tetapi rollout gagal | Klasifikasikan image, scheduling, readiness, dependency, dan traffic |
| Smoke sukses tetapi SLO buruk | Periksa telemetry window, eligible traffic, downstream, dan alert |
| Rollback code tidak memulihkan data | Stop; ikuti migration/data recovery caveat dan eskalasi |
| Canary NoData | Pause/abort dan investigate; NoData bukan sukses |

## Lanjut

Lanjutkan ke [LAB-02 — Telemetry, Alert, Backup](LAB-02-telemetry-alert-backup-evidence.md) dan [Modul 9.3](../../modul-9.3-game-day-graduation/README.md).
