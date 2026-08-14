# LAB-01 — End-to-End GitOps Flow

## Tujuan

Melacak perubahan dari source commit sampai health/SLO decision dan mengidentifikasi titik stop sebelum promotion.

## Prasyarat dan Guardrail

- Modul 6.1 dan 6.2.
- GitOps repo disposable atau static simulation.
- Jika runtime, GitLab/registry/k3s/ArgoCD/access recovery/approval harus tersedia.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan melakukan production promotion, apply, sync, registry push, atau rollback tanpa scope, approval, target, maintenance consideration, dan evidence. Jangan menulis raw artifact, state, plan, Secret, atau log credential.

## Lane A — Static Simulation

### 1. Buat evidence ledger

Isi tabel berikut dengan reference placeholder:

| Link | Reference | Owner | Status |
|---|---|---|---|
| source commit | `<source-sha>` | app team | pending |
| pipeline/job | `<pipeline-job-id>` | CI owner | pending |
| tests/scan | `<summary-reference>` | security/release | pending |
| image | `<immutable-digest>` | registry owner | pending |
| GitOps MR/commit | `<gitops-reference>` | platform | pending |
| Argo revision | `<argo-revision>` | platform | pending |
| rollout/readiness | `<redacted-summary>` | runtime owner | pending |
| telemetry/SLO | `<metric-window-reference>` | SRE | pending |
| approval | `<approval-reference>` | release owner | pending |

### 2. Simulasikan promotion

Tulis decision gate staging dan production. Untuk tiap gate jawab:

- artifact apa yang dipromosikan;
- target cluster/namespace;
- compatibility/migration review;
- siapa reviewer/approver;
- telemetry window dan SLO signal;
- rollback reference;
- kapan change harus dihentikan.

### 3. Failure injection secara konseptual

Klasifikasikan tujuh failure: lint gagal, registry pull gagal, manifest invalid, Argo permission gagal, rollout stuck, endpoint error, SLO budget burn tinggi. Tuliskan owner, containment, evidence, dan recovery.

## Lane B — Optional Disposable Runtime

Jika semua dependency tersedia:

1. Buat perubahan aplikasi non-sensitive pada branch lab.
2. Jalankan CI dan catat pipeline/job ID serta image digest.
3. Buat reviewed GitOps change ke staging.
4. Amati Argo revision/sync, rollout, readiness, smoke test, dan telemetry.
5. Promosikan hanya setelah approval dan acceptance staging.
6. Jangan lanjut production kecuali environment benar-benar disposable/approved.
7. Revert GitOps commit sesuai runbook dan simpan evidence redacted.

Status setiap tahap harus berasal dari output aktual. Jika dependency unavailable, tulis **belum diverifikasi** dan jelaskan blocker; jangan menyimpulkan dari YAML saja.

## Stop Conditions

- source/artifact tidak immutable;
- GitOps diff mengubah resource di luar scope;
- Argo target salah;
- readiness/telemetry tidak tersedia;
- approval atau rollback reference hilang;
- migration compatibility belum dinilai;
- evidence memuat secret.

## Acceptance Criteria

- [ ] Evidence ledger complete dari source sampai health/SLO.
- [ ] Promotion gates dan owner jelas.
- [ ] Failure taxonomy dan recovery dapat dijelaskan.
- [ ] Runtime claim memakai job/revision/time/target evidence.
- [ ] Tidak ada credential/raw output pada repository.

## Kaitan

- [Promotion evidence](../01-promotion-flow-evidence.md)
- [Progressive delivery](../02-progressive-delivery-rollback.md)
- [LAB-02 progressive delivery](LAB-02-canary-blue-green-introduction.md)
