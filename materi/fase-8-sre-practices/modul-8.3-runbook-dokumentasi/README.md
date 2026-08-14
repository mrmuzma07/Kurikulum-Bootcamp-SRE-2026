# Modul 8.3 — Runbook & Dokumentasi

> **Fokus:** membuat pengetahuan operasional yang dapat dipakai saat incident, memiliki stop condition, dan dapat diaudit melalui evidence serta postmortem.

## Tujuan dan Capaian

Peserta dapat:

- menulis runbook node down, disk full, certificate expired, CrashLoopBackOff, dan MetalLB IP failure;
- memulai dari read-only checks dan membangun decision tree yang bounded;
- mendefinisikan symptom, scope, severity, owner/escalation, mitigation, rollback, recovery caveat, communication, post-check, dan evidence fields;
- membedakan as-built topology, desired-state topology, dependency map, access/recovery boundary, dan documentation drift;
- menulis incident timeline dengan referensi UTC dan evidence redaction;
- menulis postmortem blameless dengan action item, owner, due date, verification, dan SLO/error-budget follow-up.

## Prasyarat

- [Modul 8.1 — Praktik SRE](../modul-8.1-praktik-sre/README.md) untuk incident roles dan SLO.
- [Modul 8.2 — Production On-Prem](../modul-8.2-production-onprem/README.md) untuk failure domain dan recovery.
- [Fase 2.4 — Operasi](../../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md), [Fase 6](../../fase-6-gitops/README.md), dan [Fase 7](../../fase-7-observability/README.md).

## Rencana Belajar

| Sesi | Materi | Output |
|---|---|---|
| 1 | [Runbook workload dan node](01-runbook-node-disk-certificate-crashloop.md) | Empat runbook dengan decision tree |
| 2 | [MetalLB, topology, evidence, postmortem](02-runbook-metallb-topology-evidence-postmortem.md) | MetalLB runbook, topology map, postmortem contract |
| 3 | [LAB-01](lab/LAB-01-runbook-validation-drill.md) | Review dan tabletop validation |
| 4 | [LAB-02](lab/LAB-02-postmortem-blameless.md) | Postmortem redacted + action verification |
| 5 | [Latihan dan kuis](evaluasi/latihan.md) | Minimal 80%, tanpa guardrail violation |

## Runbook Contract Minimum

Setiap runbook wajib berisi:

```text
symptom
scope
severity
owner/escalation
preconditions
read-only first checks
query/dashboard reference
decision tree
stop condition
safe mitigation
rollback
data recovery caveat
communication
post-check
evidence fields
last-reviewed
expires
```

`last-reviewed` dan `expires` mencegah dokumentasi tua dianggap authoritative. Runbook expired harus dihentikan atau direview sebelum action.

## Acceptance Criteria

- [ ] Lima failure scenario memiliki runbook atau pola yang dapat dieksekusi.
- [ ] Read-only first checks mendahului perubahan dan memiliki batas waktu.
- [ ] Stop condition, rollback, data recovery caveat, dan escalation jelas.
- [ ] Topology as-built/desired-state serta dependency/access boundary terdokumentasi.
- [ ] Timeline UTC, evidence retention/redaction, owner, revision, dan expiry tersedia.
- [ ] Postmortem fokus pada system/process, tetapi action item tetap memiliki accountability.

## Dua Lane

Static lane: tabletop review, runbook lint, decision tree, topology, evidence template, dan postmortem dari fault scenario sintetis. Disposable runtime lane: drill terbatas dengan approval, scoped fault, timeout, telemetry, paging boundary, mitigation, rollback/cleanup, dan evidence redacted. Jangan melakukan failure injection pada production.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menulis raw log, raw alert payload, PII, token, private key, password, kubeconfig, atau backup artifact ke postmortem. Simpan query reference, timestamp, status summary, checksum summary, dan link ke sistem evidence dengan access control.

Blameless bukan berarti tanpa accountability: owner, due date, verification, dan follow-up tetap wajib; yang dihindari adalah menyalahkan individu sebagai root cause.

## Troubleshooting

- **Runbook terlalu luas:** pecah berdasarkan symptom dan failure domain, lalu tambahkan stop condition.
- **Operator langsung mengubah resource:** pindahkan read-only checks dan approval ke awal.
- **Postmortem menjadi kronologi tanpa perbaikan:** tambahkan contributing factors, missing signal, action item, owner, due date, dan verification.
- **Topology drift:** beri revision owner, last-reviewed, expiry, dan proses update setelah perubahan.

## Kaitan

- [Fase 7](../../fase-7-observability/README.md) memberikan query/dashboard/alert reference.
- [Fase 6](../../fase-6-gitops/README.md) memberikan revision, promotion, dan rollback reference.
- [Fase 2.4](../../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md) memberikan troubleshooting dan evidence discipline.
- [Fase 9 — Capstone](../../fase-9-capstone/README.md) menggunakan minimal tiga runbook dan postmortem Game Day.

## Status Runtime

Runbook validation drill, paging, mitigation, rollback, recovery, dan action-item verification: **belum diverifikasi** tanpa evidence runtime yang lengkap.
