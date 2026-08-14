# Kuis dan Jawaban Modul 8.3

## Soal

1. Sebutkan field minimum runbook incident.
2. Mengapa read-only first checks penting?
3. Apa fungsi stop condition?
4. Apa perbedaan as-built dan desired-state topology?
5. Mengapa runbook memiliki `last-reviewed` dan `expires`?
6. Apa yang harus diredáksi dari evidence/postmortem?
7. Sebutkan elemen timeline incident yang baik.
8. Apa arti blameless dan mengapa tetap memerlukan accountability?
9. Mengapa MetalLB runbook harus membedakan L2 dan BGP?
10. Apa hubungan action-item verification dengan SLO/error budget?

## Jawaban

1. Symptom, scope, severity, owner/escalation, preconditions, read-only checks, query reference, decision tree, stop condition, mitigation, rollback, data caveat, communication, post-check, evidence, review, dan expiry.
2. Untuk membatasi blast radius, membedakan fact dari hypothesis, dan mencegah perubahan prematur.
3. Menghentikan tindakan saat target, risk, recovery, approval, atau ownership tidak aman/jelas.
4. As-built menunjukkan koneksi aktual; desired-state menunjukkan konfigurasi yang seharusnya dikelola oleh OpenTofu/Ansible/GitOps.
5. Agar drift atau dokumentasi tua tidak dipakai sebagai authoritative procedure tanpa review.
6. Credential, token, PII, raw logs, raw alert payload, private key, kubeconfig, Secret, dan backup artifact.
7. Timestamp UTC, observed fact, hypothesis, decision, action, role/owner, evidence reference, dan outcome.
8. Fokus pada system/process, bukan menyalahkan individu; accountability tetap melalui owner, due date, verification, dan follow-up.
9. Failure signal, ownership, mitigation, dan recovery route berbeda antara ARP advertisement dan BGP route/session.
10. Verification memastikan perbaikan benar-benar mengurangi risk dan dapat memicu revisi objective, alert, freeze, atau promotion policy.

## Penilaian

Soal 1–10 bernilai 2 poin. Total 20; lulus minimal 16 tanpa pelanggaran guardrail.

## Kaitan

Hubungkan output ke [Modul 8.1](../../modul-8.1-praktik-sre/README.md), [Modul 8.2](../../modul-8.2-production-onprem/README.md), [Fase 6](../../../fase-6-gitops/README.md), dan [Fase 7](../../../fase-7-observability/README.md).
