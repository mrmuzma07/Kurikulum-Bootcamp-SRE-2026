# Latihan Modul 6.3 — End-to-End Flow

## Petunjuk

Nilai minimum lulus 80/100. Static lane cukup; runtime hanya pada disposable scope. Credential plaintext, raw secret/artifact, atau mutation tanpa approval berarti tidak lulus otomatis.

## Latihan 1 — Evidence ledger (25 poin)

Buat ledger source commit → pipeline/job → test/scan → image digest → GitOps commit → Argo revision → rollout → telemetry/SLO → approval. Sertakan owner, retention, redaction, dan status bila runtime belum tersedia.

## Latihan 2 — Promotion decision (20 poin)

Tentukan gate staging dan production serta kondisi stop: compatibility, migration, freeze, error budget, approval, rollback reference, dan incident owner.

## Latihan 3 — Failure taxonomy (20 poin)

Klasifikasikan CI, registry, manifest, Argo sync, rollout, application, dan SLO failure. Untuk tiap item tuliskan containment, owner, evidence, dan recovery.

## Latihan 4 — Rollback analysis (20 poin)

Bandingkan Git revert, Argo revision sync, service switch, dan data recovery. Jelaskan mengapa migration, CRD, PVC, hook, dan external side effect tidak otomatis dibalikkan.

## Latihan 5 — Progressive delivery (15 poin)

Review desain canary dan blue-green. Tentukan traffic, metric, capacity, pause, abort, NoData, stable/canary service, dan SLO dependency.

## Rubrik

- 90–100: evidence dan recovery production-oriented.
- 80–89: acceptance terpenuhi.
- 60–79: perlu remediasi.
- <60: belum lulus.
- Pelanggaran secret/destructive guardrail: tidak lulus otomatis.

## Pengumpulan

Diagram, ledger, failure matrix, rollback decision, Rollout placeholder, dan evidence redacted. Semua execution yang belum dijalankan harus diberi status **belum diverifikasi**.
