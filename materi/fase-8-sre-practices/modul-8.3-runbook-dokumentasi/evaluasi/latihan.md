# Latihan Modul 8.3 — Runbook dan Dokumentasi

## Hari 1 — Runbook

1. Tulis runbook node down dan disk full dengan read-only first checks.
2. Tulis runbook certificate expired dan CrashLoopBackOff dengan data recovery caveat.
3. Tulis runbook MetalLB IP failure yang membedakan L2, BGP, VLAN, ARP, MTU, endpoint, dan NetworkPolicy.

## Hari 2 — Topology dan Evidence

1. Buat as-built topology dan desired-state topology secara terpisah.
2. Buat dependency map serta access/recovery boundary.
3. Buat evidence retention/redaction policy tanpa raw log, payload, Secret, token, PII, atau backup artifact.

## Hari 3 — Tabletop

1. Jalankan tabletop dua scenario bersama reviewer.
2. Catat timeline UTC, decision, mitigation, rollback, communication, post-check, dan gap.
3. Ubah gap menjadi action item dengan owner, due date, dan verification.

## Hari 4 — Postmortem

1. Tulis postmortem blameless dari scenario sintetis.
2. Pisahkan fact, hypothesis, contributing factor, missing signal, dan action.
3. Kaitkan hasilnya ke perubahan SLO/error budget, alert, runbook, atau change policy.

## Hari 5 — Review

Periksa `last-reviewed`, `expires`, revision ownership, stop condition, dan status runtime. Dokumentasi expired tidak boleh digunakan untuk perubahan.

**Lulus:** minimal 80% dan tidak ada pelanggaran guardrail evidence atau secret.
