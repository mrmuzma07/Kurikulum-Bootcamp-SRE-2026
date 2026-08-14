# Kuis dan Jawaban Modul 8.1

## Soal

1. Apa perbedaan SLI, SLO, SLA, dan error budget?
2. Mengapa numerator dan denominator harus mendefinisikan eligible traffic?
3. Apa yang harus dilakukan saat data SLO `NoData`?
4. Mengapa burn rate membutuhkan window dan baseline?
5. Apa ciri toil dan bagaimana memilih automation?
6. Mengapa automation memerlukan blast-radius control?
7. Sebutkan elemen minimum on-call contract.
8. Apa perbedaan IC, communications lead, dan operations lead?
9. Mengapa Git revert tidak selalu cukup sebagai rollback?
10. Apa fungsi change freeze dan CAB ringan?

## Jawaban

1. SLI adalah pengukuran; SLO adalah target; SLA adalah komitmen kontraktual; error budget adalah toleransi kegagalan dari SLO.
2. Agar angka reliability mewakili traffic yang menjadi tanggung jawab service dan tidak mudah dimanipulasi oleh exclusion.
3. Ikuti missing-data policy, biasanya pause/investigate; jangan menganggap `NoData` sebagai sukses atau nol error.
4. Window/baseline membedakan spike singkat dari konsumsi budget yang representatif.
5. Toil manual, repetitif, automatable, dan tanpa nilai durable; pilih automation dengan owner, guardrail, rollback, dan measurable benefit.
6. Bot dapat salah scope atau loop; allowlist, rate limit, timeout, audit, kill switch, dan approval membatasi blast radius.
7. Primary/secondary, coverage/timezone, ack, severity, escalation, paging boundary, fatigue control, handoff, dan access recovery.
8. IC mengatur incident/decision; communications lead mengelola stakeholder update; operations lead menjalankan diagnosis/mitigasi.
9. Migration, PVC, cache, queue, atau external side effect mungkin sudah berubah dan tidak dibalikkan oleh source revert.
10. Freeze mengurangi risiko saat reliability/operational state tidak aman; CAB ringan memberi peer risk review dan keputusan terdokumentasi.

## Penilaian

Soal 1–10 bernilai 2 poin. Total 20; lulus minimal 16 tanpa pelanggaran guardrail.

## Kaitan

Hubungkan output ke [Modul 8.2](../../modul-8.2-production-onprem/README.md), [Modul 8.3](../../modul-8.3-runbook-dokumentasi/README.md), [Fase 6](../../../fase-6-gitops/README.md), dan [Fase 7](../../../fase-7-observability/README.md).
