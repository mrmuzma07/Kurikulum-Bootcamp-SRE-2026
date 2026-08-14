# 02 — On-Call, Incident Response, dan Change Management

## Tujuan

Membangun proses yang membuat detection, response, communication, change, dan recovery dapat diprediksi tanpa bergantung pada satu individu.

## 1. On-Call Contract

On-call matrix minimum:

| Elemen | Keputusan yang harus tertulis |
|---|---|
| primary/secondary | siapa menerima paging dan siapa backup |
| coverage/timezone | window, hari libur, handoff UTC |
| severity | impact, urgency, customer scope |
| acknowledgement | batas waktu dan tindakan bila lewat |
| escalation | team/role berikutnya dan expiry |
| paging boundary | alert mana yang page versus ticket |
| fatigue | dedup, grouping, quiet hours, rotation limit |
| access recovery | break-glass owner, audit, follow-up |

Gunakan placeholder `<approved-team>` dan `<redacted-contact-point>`. Jangan menyimpan nomor telepon, webhook, token, atau credential dalam repository.

Handoff harus menyebut incident terbuka, change aktif, error-budget state, dependency concern, dan action berikutnya. Primary tidak boleh menjadi single point of failure.

## 2. Incident Lifecycle

```text
detection → declaration → assign IC/comms/ops lead → triage
→ mitigate or rollback → monitor → resolve → close
→ timeline/postmortem/action verification
```

- **Detection:** signal berasal dari alert, user report, synthetic check, atau operator.
- **Declaration:** tetapkan incident ID, severity, scope, IC, channel, dan timestamp UTC.
- **Incident Commander (IC):** mengatur prioritas, decision log, dan stop condition; bukan harus mengerjakan semua command.
- **Communications lead:** membuat update terjadwal dan stakeholder boundary tanpa membocorkan PII/secret.
- **Operations lead:** menjalankan diagnosis dan mitigasi yang disetujui.
- **Triage:** read-only checks dahulu; klasifikasikan workload, node, network, storage, dependency, atau delivery.
- **Mitigation:** mengurangi impact dengan tindakan reversible dan bounded.
- **Rollback:** kembali ke revision/service state yang diketahui, setelah memeriksa migration, PVC, hook, dan external side effect.
- **Resolution:** symptom berhenti, recovery tervalidasi, dan monitoring window memenuhi acceptance.
- **Closure:** IC menyatakan selesai, evidence dikunci, follow-up dibuat.

Incident timeline harus memisahkan observed fact, hypothesis, decision, action, actor role, dan outcome. Referensi waktu gunakan UTC atau timezone yang eksplisit.

## 3. Blameless Postmortem

Postmortem minimum memuat impact, detection, timeline, contributing factors, what went well, what went poorly, missing signal, action item, owner, due date, verification, dan keputusan SLO/error budget. Blameless tidak berarti tanpa accountability; fokusnya memperbaiki system/process, bukan menyalahkan individu.

## 4. Change Management

Klasifikasikan perubahan:

| Kelas | Contoh | Kontrol |
|---|---|---|
| standard | perubahan berulang, low risk | peer review, automation, evidence |
| normal | chart/config/OS patch | impact, window, approval, rollback |
| high risk | schema, CRD, network, quorum | rehearsal, CAB ringan, communication |
| emergency | mitigasi active incident | IC/approver, scoped action, review setelahnya |

Change record harus berisi scope, dependency, risk/impact, owner, peer reviewer, plan/diff summary, maintenance window, pre-check, success criteria, rollback/recovery, communication, dan evidence location.

**Change freeze** berlaku ketika incident besar, budget terbakar, staffing rendah, dependency tidak sehat, atau maintenance boundary tertutup. Freeze memiliki owner, alasan, mulai/berakhir, exception path, dan review cadence.

**CAB ringan** bukan rapat formal tanpa keputusan. Minimal dua reviewer yang menilai risk, blast radius, observability, recovery, data migration, access recovery, dan customer communication.

Rollback code tidak otomatis membalikkan database migration, PVC data, cache, queue, webhook, atau external side effect. Emergency change tetap harus memiliki retrospective review.

## 5. Static dan Runtime Lane

Static lane: review matrix, incident template, decision tree, change record, freeze policy, CAB checklist, dan postmortem template.

Disposable runtime lane: inject fault terbatas pada target disposable dengan approval, timeout, paging boundary, mitigation, rollback, cleanup, dan evidence redacted. Tanpa alert/timeline/decision/recovery evidence, drill **belum diverifikasi**.

## Acceptance Criteria

- [ ] On-call primary/secondary, handoff, timezone, escalation, severity, acknowledgement, dan access recovery lengkap.
- [ ] Role IC/comms/ops dan incident lifecycle jelas.
- [ ] Timeline memakai UTC dan membedakan fact/hypothesis/decision/action.
- [ ] Change class, freeze, CAB, emergency review, rollback, dan migration caveat ada.
- [ ] Postmortem memiliki accountability tanpa blame.
- [ ] Tidak ada PII atau credential dalam template/evidence.
