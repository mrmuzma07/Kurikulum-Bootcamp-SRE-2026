# Modul 6.1 — GitLab CI/CD

> **Tujuan akhir:** menyusun pipeline yang dapat memvalidasi, menguji, membangun, mempublikasikan, dan mempromosikan artifact dengan gate yang terlihat serta credential boundary yang aman.

## Capaian Modul

- [ ] Menjelaskan stages, jobs, `rules`, `workflow`, `needs`, artifacts, cache, environments, dan manual approval.
- [ ] Merancang pipeline lint → test → build multi-arch → scan/sign → push registry.
- [ ] Membedakan shared runner GitLab dan self-hosted OrbStack runner.
- [ ] Menyusun pipeline OpenTofu plan/apply dan Ansible lint/check/run yang dibatasi.
- [ ] Menyimpan evidence tanpa credential, raw plan, raw state, atau raw secret.

## Rencana 2 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [Pipeline stages dan job graph](01-pipeline-stages-jobs-rules.md), [runner/artifact/cache](02-runner-artifacts-cache-environment.md) | [LAB-01](lab/LAB-01-ci-lint-test-build-push.md) |
| 2 | [IaC, Ansible, dan approval](03-iac-ansible-pipeline-approval.md) | [LAB-02](lab/LAB-02-iac-plan-approval-lab-run.md) + [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Fase 0.3 untuk branch, merge request, dan review.
- Fase 1 untuk image multi-arch dan registry.
- Fase 3 untuk OpenTofu plan/state.
- Fase 4 untuk Ansible lint, limit, readiness, dan k3s handoff.
- Fase 5 untuk Helm lint/template/package.

## Acceptance Criteria

- [ ] Job graph memiliki gate dan failure behavior yang jelas.
- [ ] Build memakai image/digest reference yang dapat ditelusuri.
- [ ] Artifact/cache memiliki expiry, scope, access, dan redaction policy.
- [ ] IaC apply dan Ansible run dilindungi approval, limit, dan recovery.
- [ ] Tidak ada credential literal atau raw sensitive artifact.

## Guardrail

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Pipeline green hanya membuktikan job pada scope tersebut berhasil. Ia tidak membuktikan cluster sync, readiness, health, atau SLO.

## Troubleshooting

- Job tidak berjalan: periksa `workflow`, `rules`, protected branch, dan pipeline source.
- `needs` gagal: periksa dependency job, artifact availability, dan stage graph.
- Build ARM64 gagal: periksa runner architecture, buildx emulation, base image, dan registry manifest.
- Artifact bocor: hentikan promotion, revoke credential yang terdampak, redaksi artifact, dan review retention.
- Apply/run tidak aman: stop; verifikasi target, plan, `--limit`, approval, dan rollback.

## Kaitan

- [Modul 3.2 — OpenTofu production](../../fase-3-opentofu/modul-3.2-modul-pola-produksi/README.md)
- [Modul 4.2 — pola Ansible](../../fase-4-ansible/modul-4.2-pola-produksi-ansible/README.md)
- [Fase 5 — Helm](../../fase-5-helm/README.md)
- [Modul 6.2 — ArgoCD](../modul-6.2-argocd/README.md)

## Catatan SRE

Pipeline adalah sistem produksi: concurrency, queue, runner capacity, artifact retention, approval latency, dan observability-nya perlu dipikirkan, bukan hanya isi YAML.

## Status Runtime

Materi tersedia. GitLab runner, pipeline execution, image build/push, OpenTofu apply, dan Ansible run belum diverifikasi tanpa evidence.
