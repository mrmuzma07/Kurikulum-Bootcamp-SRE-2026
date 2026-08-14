# Kuis dan Jawaban Modul Grafana/Alerting/SLO

## Soal

1. Apa boundary utama komponen modul ini?
2. Mengapa status `UP`, `Healthy`, atau config valid bukan bukti SLO?
3. Sebutkan dua failure mode utama dan evidence yang perlu dikumpulkan.
4. Bagaimana mencegah cardinality/volume/retention tidak terkendali?
5. Apa yang wajib dilakukan terhadap credential, PII, dan raw payload?
6. Apa perbedaan static lane dan disposable runtime lane?
7. Mengapa collection, storage, query, dashboard, alert, dan notification harus dipisahkan?
8. Apa policy aman saat no-data/backpressure/transport failure?
9. Kapan runtime boleh dilakukan?
10. Apa acceptance evidence minimum untuk menyatakan lab verified?

## Jawaban

1. Collector/storage/query/dashboard/alert/runbook harus dipisah sesuai ownership; komponen tidak boleh diklaim mengambil tanggung jawab layer lain.
2. Status tersebut hanya signal lokal controller/config/resource; tidak membuktikan query data, notification delivery, user impact, atau SLO.
3. Contoh: backend unavailable dibuktikan queue/retry/drop/status; query kosong dibuktikan tenant/time range/ingestion/component summary. Evidence harus redacted.
4. Gunakan label/attribute allowlist bounded, sampling/filtering, scrape/ingest limits, retention, query limits, dan capacity headroom.
5. Jangan commit/cetak; gunakan secret manager/reference, redaction sebelum storage/export, access control, rotation, backup, dan recovery.
6. Static lane hanya review/design/render; runtime memerlukan context/target verification, approval, disposable scope, execution, rollback/cleanup, dan evidence.
7. Karena setiap tahap dapat sukses/gagal terpisah; transport accepted tidak sama dengan durable/queryable, panel, firing, receipt, atau action.
8. Pause/investigate, bounded retry/queue, alert on degradation, dan jangan menganggap no-data sebagai healthy atau drop sebagai nol.
9. Hanya pada target disposable yang scope, owner, access recovery, approval, timeout, stop condition, dan rollback-nya jelas; failure injection tidak boleh production.
10. Reference/revision, target/time/window, command/outcome summary, stage-by-stage evidence, redaction, dan status yang dapat diaudit.

## Penilaian

Soal 1–10 masing-masing 4 poin; total 40. Lulus minimal 32 poin dan tanpa pelanggaran guardrail.

## Kaitan

Setelah lulus, hubungkan evidence modul ini ke alur Fase 7 dan [Fase 6 GitOps](../../fase-6-gitops/README.md). Runtime yang tidak memiliki evidence tetap **belum diverifikasi**.
