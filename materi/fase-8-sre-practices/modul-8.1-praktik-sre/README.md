# Modul 8.1 — Praktik SRE

> **Fokus:** membuat reliability dapat diukur, dioperasikan, dan dipulihkan melalui SLO, error budget, toil control, on-call, incident command, dan change management.

## Tujuan dan Capaian

Setelah modul ini peserta dapat:

- membedakan SLI, SLO, SLA, error budget, dan burn rate;
- menulis SLO contract dengan numerator/denominator yang dapat diaudit;
- menghubungkan SLO dengan dashboard/alert Fase 7 dan promotion gate Fase 6;
- membuat toil inventory dan automation proposal dengan blast-radius control;
- menyusun on-call rotation, escalation, handoff, access recovery, dan fatigue boundary;
- menjalankan incident lifecycle serta menulis postmortem blameless;
- menilai perubahan normal, high-risk, emergency, freeze, rollback, dan CAB ringan.

## Prasyarat

- [Fase 6 — GitOps](../../fase-6-gitops/README.md) untuk promotion dan rollback evidence.
- [Fase 7 — Observability](../../fase-7-observability/README.md) untuk metrics, alert, dashboard, dan SLO signal.
- Pemahaman operasi workload dari [Modul 2.4](../../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md).

## Rencana Belajar

| Sesi | Materi | Output |
|---|---|---|
| 1 | [SLI, SLO, error budget, toil](01-sli-slo-error-budget-toil.md) | SLO contract, math, toil inventory |
| 2 | [On-call, incident, change](02-oncall-incident-response-change-management.md) | On-call matrix, incident template, change record |
| 3 | [LAB-01](lab/LAB-01-slo-error-budget-oncall.md) | Dashboard/error-budget dan coverage review |
| 4 | [LAB-02](lab/LAB-02-incident-response-change-freeze.md) | Incident drill static/disposable plan |
| 5 | [Latihan dan kuis](evaluasi/latihan.md) | Minimal 80%, tanpa guardrail violation |

## Acceptance Criteria

- [ ] SLO memiliki service, objective, window, owner, numerator, denominator, eligibility, exclusion, missing-data policy, dan review cadence.
- [ ] Error budget dan burn rate dapat dihitung ulang dari angka yang disepakati.
- [ ] Alert memiliki owner, severity, runbook, query reference, evaluation window, dan paging boundary.
- [ ] On-call memiliki primary/secondary, handoff, escalation, timezone, acknowledgement, coverage, fatigue, dan access recovery.
- [ ] Incident role, severity, timeline UTC, mitigation, rollback, closure, dan postmortem terdefinisi.
- [ ] Change record memiliki risk/impact, peer review, maintenance window, freeze/CAB, rollback, dan migration caveat.
- [ ] Runtime claim tetap **belum diverifikasi** sampai execution evidence tersedia.

## Dua Lane

**Static lane:** review contract, math, policy, template, dashboard query, dan decision tree tanpa memicu incident.

**Disposable runtime lane:** hanya pada target disposable dengan preflight context/namespace/owner, approval, timeout, bounded fault injection, rollback/cleanup, dan evidence redacted. Jangan menguji chaos atau paging pada production.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menulis nomor telepon, webhook, token, raw alert payload, PII, authorization header, atau credential ke dokumen. Gunakan placeholder seperti `<approved-team>`, `<redacted-contact-point>`, dan `<incident-id>`. Jangan menyamakan alert firing dengan incident resolution atau SLO success.

## Troubleshooting

- **SLO tidak dapat dihitung:** periksa numerator, denominator, eligible traffic, dan missing-data policy sebelum mengubah query.
- **Burn rate terlalu tinggi:** validasi window dan traffic eligibility; jangan langsung menaikkan threshold.
- **Paging fatigue:** tinjau severity, grouping, inhibition, schedule, dan automation; jangan menonaktifkan alert tanpa owner/expiry.
- **Rollback tidak aman:** cek migration, PVC, external side effect, dan data recovery; Git revert tidak otomatis mengembalikan data.

## Kaitan

- Lanjutkan signal ke [Fase 7](../../fase-7-observability/README.md).
- Lanjutkan delivery gate ke [Fase 6](../../fase-6-gitops/README.md).
- Lanjutkan readiness dan Game Day ke Fase 9 (menyusul) bila tersedia.

## Status Runtime

SLO dashboard, alert/paging, incident drill, rollback, dan change-freeze execution: **belum diverifikasi** tanpa evidence runtime yang sesuai.
