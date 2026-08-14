# Kuis dan Jawaban Modul 9.3

## Soal

1. Sebutkan role minimum pada incident command.
2. Mengapa timeline harus memakai UTC dan evidence reference?
3. Apa urutan lifecycle incident yang disarankan?
4. Mengapa satu drill hanya menguji satu fault?
5. Apa syarat rollback <5 menit agar dapat diklaim?
6. Sebutkan tujuh scenario Game Day resmi.
7. Apa isi minimum postmortem blameless?
8. Apa perbedaan `ready`, `conditional`, dan `not ready`?
9. Mengapa pipeline green atau Argo `Healthy` tidak cukup untuk graduation?
10. Apa yang harus dilakukan terhadap known gap setelah graduation review?

## Jawaban

1. Primary/secondary on-call, incident commander, communications lead, operations lead, dan service/platform owner.
2. UTC menghindari ambiguity lintas timezone; evidence reference membuat setiap fact/decision dapat diaudit tanpa menyalin raw data.
3. Detection, declaration/severity, read-only triage, runbook, bounded mitigation, rollback/recovery, telemetry/post-check, closure, postmortem, action verification.
4. Untuk membatasi blast radius, menjaga causal reasoning, dan membuat rollback/cleanup dapat dikendalikan.
5. Waktu harus diukur dari declaration/decision sampai traffic, telemetry, dan recovery outcome sehat; command selesai saja tidak cukup.
6. Worker down, disk full, bad release, certificate ingress expired, latency 10x, MetalLB/ARP failure, dan destroy/rebuild.
7. Impact/detection redacted, timeline UTC, contributing factors, what went well/poorly, missing signal, action items owner/due/verification, dan SLO/error-budget decision.
8. `ready` memiliki evidence wajib lengkap dan risiko terkendali; `conditional` memiliki gap dengan risk acceptance/owner/expiry; `not ready` memiliki evidence wajib hilang atau risiko tidak dapat diterima.
9. Keduanya hanya signal lokal; graduation memerlukan rebuild, traffic, telemetry, alerts, recovery, Game Day, SLO, RPO/RTO, dan action verification.
10. Beri owner, due date UTC, risk/expiry, dan verification test; review ulang sampai action benar-benar terbukti.

## Penilaian

Soal 1–10 bernilai 4 poin, total 40. Lulus minimal 32/40 tanpa pelanggaran guardrail.

## Kaitan

Fase9 selesai secara dokumentasi setelah seluruh modul dan evaluasi ditinjau. Runtime graduation tetap **belum diverifikasi** tanpa evidence aktual.
