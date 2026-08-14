# 01 — SLI, SLO, Error Budget, Burn Rate, dan Toil

## Tujuan

Mendefinisikan reliability sebagai objective yang dapat diukur, ditinjau, dan dipakai untuk mengambil keputusan delivery serta operasi.

## 1. Istilah Dasar

- **SLI (Service Level Indicator):** pengukuran perilaku layanan, misalnya request yang berhasil dibagi seluruh request eligible.
- **SLO (Service Level Objective):** target SLI pada window yang disepakati, misalnya availability `99.9%` selama kalender 30 hari.
- **SLA:** komitmen eksternal dengan konsekuensi kontraktual; jangan menganggap semua SLO otomatis SLA.
- **Error budget:** toleransi kegagalan yang tersedia, `1 - objective`. SLO `99.9%` memberi budget `0.1%`.
- **Burn rate:** laju konsumsi budget dibanding laju konsumsi normal pada window tersebut. Burn rate `1` menghabiskan budget tepat sesuai objective window.
- **Toil:** pekerjaan manual, repetitif, automatable, dan tidak memberi nilai durable pada sistem.

## 2. SLO Contract

Gunakan contract yang disetujui, bukan angka yang dipilih setelah incident:

```yaml
service: <approved-service>
owner: <approved-team>
window: <approved-calendar-window>
objective: <approved-percentage>
availability:
  numerator: <successful-eligible-requests>
  denominator: <eligible-requests>
  exclusions:
    - <approved-maintenance-or-test-traffic-policy>
latency:
  percentile: <approved-percentile>
  threshold: <approved-duration>
  denominator: <eligible-requests-with-valid-duration>
missing_data_policy: <pause-alert-or-review-policy>
review_cadence: <approved-cadence>
runbook: <approved-runbook-reference>
```

### Numerator dan denominator

Denominator harus merepresentasikan traffic yang memang berada dalam tanggung jawab service. Dokumentasikan:

- request yang eligible dan yang dikecualikan;
- health check, synthetic traffic, retry, timeout, dan cancellation;
- status code yang dianggap berhasil atau gagal;
- sampling atau aggregation;
- timezone/window boundary;
- missing telemetry: apakah dianggap error, `NoData`, atau memicu review terpisah.

Exclusion tidak boleh dipakai untuk menghapus incident yang tidak nyaman. Setiap exclusion memiliki owner, alasan, expiry, dan audit trail.

## 3. Error-Budget Math

Jika objective availability `99.9%` pada 30 hari dan denominator eligible adalah `1,000,000` request:

```text
budget fraction = 1 - 0.999 = 0.001
budget requests = 1,000,000 × 0.001 = 1,000 request
```

Gunakan angka placeholder atau angka sintetis dalam materi. Untuk latency, definisikan apakah budget menghitung request di atas threshold, bukan mencampurnya dengan availability budget.

Burn rate harus memakai window yang representatif. Window pendek cepat mendeteksi spike tetapi mudah noisy; window panjang stabil tetapi lambat. Alert burn-rate wajib mempunyai threshold, evaluation window, missing-data policy, severity, owner, runbook, dan notification boundary.

## 4. Hubungan dengan Fase 7 dan Fase 6

```text
Fase 7 metric/query → dashboard dan recording rule → burn-rate alert
→ Alertmanager boundary → incident/runbook
→ error-budget decision → Fase 6 promotion/change gate
```

SLO gate dapat menghentikan promotion ketika budget habis atau burn rate tidak sehat. Gate harus menyebut window, query revision, baseline, approval, dan tindakan bila telemetry unavailable. Dashboard green atau Argo `Healthy` tidak membuktikan SLO.

## 5. Toil Inventory dan Automation

Catat pekerjaan dengan format:

| Field | Isi |
|---|---|
| aktivitas | `<repetitive-activity>` |
| frekuensi/durasi | `<synthetic-frequency-and-duration>` |
| dampak | engineer time, risk, fatigue |
| trigger | alert, ticket, schedule |
| kandidat automation | script, controller, policy, runbook |
| guardrail | allowlist, dry-run, rate limit, approval |
| blast radius | namespace/host/service |
| rollback | disable/revert/restore |
| owner/expiry | `<approved-owner>` / `<review-date>` |

Automation bukan selalu aman. Aksi idempotent dengan allowlist, rate limit, concurrency limit, audit log, timeout, circuit breaker, dan kill switch lebih aman daripada bot yang bebas mengubah seluruh cluster. `--check` atau dry-run tidak menjamin zero side effect.

## 6. Static dan Runtime Lane

**Static:** review contract, math, query reference, dashboard panel, alert metadata, toil inventory, dan automation guardrail.

**Disposable runtime:** generate synthetic eligible traffic pada target disposable, ambil metric window, evaluasi alert, dan kaitkan ke incident timeline. Tanpa metric, notification, dan decision evidence aktual, SLO/error-budget outcome tetap **belum diverifikasi**.

## Acceptance Criteria

- [ ] SLI/SLO contract lengkap dan dapat dihitung ulang.
- [ ] Eligibility, exclusion, missing-data, window, owner, dan cadence eksplisit.
- [ ] Error budget dan burn rate dibedakan.
- [ ] Hubungan dashboard/alert/promotion dijelaskan.
- [ ] Toil inventory dan automation guardrail memiliki blast-radius control.
- [ ] Tidak ada credential, PII, raw alert payload, atau secret.
