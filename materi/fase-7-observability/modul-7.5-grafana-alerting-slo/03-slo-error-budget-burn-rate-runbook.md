# 03 — SLO, Error Budget, Burn Rate, dan Runbook

## Tujuan

Mengubah telemetry yang tervalidasi menjadi objective layanan, error budget, alert burn-rate, dan tindakan operasional yang dapat diaudit.

## 1. SLO Contract

Definisikan:

```text
service: <approved-service>
window: <approved-calendar-window>
objective: <approved-percentage>
availability_numerator: successful_requests
availability_denominator: eligible_requests
latency_objective: <approved-percentile-and-threshold>
owner: <approved-owner>
```

Jangan memakai seluruh request tanpa mengecualikan traffic yang memang tidak eligible; exclusion harus terdokumentasi dan tidak menjadi cara menyembunyikan failure. Availability, latency, dan error rate dapat memiliki SLO terpisah.

## 2. Error Budget dan Burn Rate

Jika objective 99.9% pada window 30 hari, budget kegagalan adalah 0.1% dari eligible events. Burn rate membandingkan laju budget consumption terhadap laju normal. Gunakan fast/slow windows yang disetujui, misalnya short window untuk incident cepat dan long window untuk trend; angka harus dikalibrasi traffic, detection latency, dan false positive.

Alert burn-rate adalah symptom. Runbook harus mengarahkan cek deployment, dependency, saturation, logs, traces, and rollback/traffic control. `Healthy`, `UP`, atau dashboard green tidak otomatis berarti SLO tercapai.

## 3. Runbook Contract

```text
symptom: <approved-alert>
owner/escalation: <approved-team-reference>
query/dashboard: <approved-reference>
first_checks: <redacted-check-list>
stop_condition: <approved-condition>
mitigation: <approved-safe-action>
rollback: <approved-revision-or-traffic-action>
data_recovery_caveat: <approved-caveat>
post_check: <approved-slo-window>
```

Runbook harus membedakan mitigation, rollback code/config, dan data recovery. Semua tindakan mutatif memerlukan scope, approval, access recovery, dan stop condition.

## 4. Promotion dan Failure Injection

SLO gate untuk promotion membutuhkan representative telemetry window, query revision, target, traffic context, error budget decision, and reviewer. Failure injection hanya disposable: bounded traffic, approved fault, observation window, rollback, cleanup, and evidence. Tidak ada chaos/failure injection pada production.

## Acceptance Criteria

- [ ] SLO numerator/denominator/objective/window/owner jelas.
- [ ] Error budget dan burn-rate mempunyai window dan threshold yang terkalibrasi.
- [ ] Runbook memiliki query, escalation, stop, mitigation, rollback, data caveat.
- [ ] Promotion/failure injection menggunakan evidence, bukan status controller.

## Troubleshooting dan Catatan SRE

Budget turun tanpa alert: cek query denominator, missing data, evaluator, and route. Burn rate noisy: cek traffic volume, short window, aggregation, and deploy correlation. Runbook tidak efektif: tambah stop condition dan evidence field, bukan sekadar daftar command.

## Kaitan

Lanjutkan ke [Fase 8 — Praktik SRE](../../fase-8-sre-practices/README.md) untuk incident response dan error-budget practice. Hubungkan bukti delivery dari [Fase 6](../../fase-6-gitops/README.md).

## Status Runtime

SLO calculation, burn-rate alert, promotion gate, failure injection, dan runbook execution **belum diverifikasi**.
