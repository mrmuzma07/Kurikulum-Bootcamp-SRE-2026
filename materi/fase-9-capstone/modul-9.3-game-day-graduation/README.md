# Modul 9.3 — Game Day dan Graduation

> **Fokus:** menjalankan simulasi on-call yang bounded, menutup setiap incident dengan postmortem, lalu mengambil keputusan readiness berbasis evidence.

## Tujuan dan Capaian

Peserta dapat:

- menetapkan primary/secondary on-call, incident commander, communications lead, dan operations lead;
- menjalankan detection → declaration → triage → mitigation/rollback → recovery → closure;
- memilih runbook berdasarkan symptom dan menjaga stop condition;
- menguji tujuh skenario resmi tanpa chaos pada production;
- menulis timeline UTC, evidence packet, postmortem blameless, action item, dan verification;
- memberi keputusan `ready`, `conditional`, atau `not ready` dengan known gaps yang eksplisit.

## Prasyarat

- [Modul 9.1 — Platform Rebuild](../modul-9.1-platform-rebuild/README.md).
- [Modul 9.2 — Delivery, Observability, Recovery](../modul-9.2-delivery-observability/README.md).
- [Modul 8.1 — Praktik SRE](../../fase-8-sre-practices/modul-8.1-praktik-sre/README.md).
- [Modul 8.3 — Runbook](../../fase-8-sre-practices/modul-8.3-runbook-dokumentasi/README.md).

## Rencana Sesi

| Sesi | Materi | Output |
|---|---|---|
| 1 | [Game Day dan on-call](01-game-day-oncall-scenarios.md) | scenario matrix dan incident roles |
| 2 | [Readiness dan postmortem](02-readiness-review-postmortem.md) | rubric, postmortem, action verification |
| 3 | [LAB-01](lab/LAB-01-game-day-drill.md) | drill packet dan timeline |
| 4 | [LAB-02](lab/LAB-02-destroy-rebuild-graduation.md) | rebuild review dan graduation packet |
| 5 | [Latihan dan kuis](evaluasi/latihan.md) | minimal 80%, tanpa guardrail violation |

## Deliverables

1. Game Day matrix untuk tujuh skenario.
2. Incident command card dan escalation matrix.
3. Minimal tiga runbook yang diuji atau ditinjau.
4. Postmortem redacted untuk setiap incident yang benar-benar dijalankan.
5. Graduation evidence packet dengan RTO/RPO, SLO/error budget, known gaps, dan keputusan.

## Acceptance Criteria

- [ ] Setiap skenario memiliki owner, severity, acknowledgement, escalation, timeout, stop condition, dan cleanup.
- [ ] Mitigation, rollback, recovery caveat, communication, dan post-check tercatat.
- [ ] Timeline menggunakan UTC dan merujuk evidence redacted.
- [ ] Action item memiliki owner, due date, dan verification.
- [ ] `ready` hanya dipilih bila evidence wajib lengkap dan gap tidak menghalangi operasi aman.
- [ ] Tanpa evidence wajib, keputusan minimal `conditional` atau `not ready`.

## Dua Lane

Static lane berupa tabletop, decision tree, readiness review, dan postmortem dari fault sintetis. Runtime lane hanya disposable, satu fault per iterasi, bounded scope, approval, timeout, rollback/cleanup, dan evidence redacted.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan melakukan failure injection pada production, mencetak raw log/alert payload, menaruh PII atau credential dalam postmortem, atau menghancurkan target tanpa allowlist, plan/diff, approval, backup, recovery path, dan cleanup.

## Troubleshooting

- **Incident roles tumpang tindih:** tetapkan satu incident commander dan tulis handoff UTC.
- **Operator langsung mengubah resource:** kembali ke read-only first checks dan approval.
- **Rollback cepat tetapi data rusak:** periksa migration/data caveat; rollback code bukan rollback data.
- **Banyak alert tanpa keputusan:** pilih alert owner/runbook dan catat missing signal.
- **Rebuild selesai tetapi readiness gagal:** pisahkan infrastructure, cluster, delivery, telemetry, backup, dan SLO gate.

## Kaitan

- [Fase 8](../../fase-8-sre-practices/README.md) menyediakan incident, change, runbook, dan postmortem contract.
- [Fase 7](../../fase-7-observability/README.md) menyediakan telemetry dan alert evidence.

## Status Runtime

Game Day, paging, mitigation, rollback kurang dari lima menit, rebuild, RPO/RTO, postmortem verification, dan graduation decision: **belum diverifikasi** tanpa execution evidence.
