# Modul 7.5 — Grafana, Alerting, dan SLO

> **Tujuan:** mengubah telemetry menjadi dashboard, alert route, notification evidence, runbook, dan keputusan error budget.

## Capaian

- [ ] Mendesain dashboard USE infra dan RED service sebagai code.
- [ ] Mengonfigurasi data source, variable bounded, panel query, unit, legend, ownership, dan RBAC.
- [ ] Membedakan Prometheus rule, Grafana rule, Alertmanager route, contact point, receipt, dan notification.
- [ ] Menulis alert error rate >5%, p95 >500ms, disk >85%, dan node down dengan policy no-data.
- [ ] Menghitung availability/latency/error SLO, error budget, fast/slow burn rate, dan runbook.
- [ ] Menguji failure injection hanya pada disposable target dan menghubungkannya ke evidence.

## Materi dan Praktik

| Hari | Materi | Lab |
|---|---|---|
| 1 | [Dashboard USE/RED as code](01-grafana-use-red-dashboard-as-code.md) | [LAB-01](lab/LAB-01-dashboard-data-source-evidence.md) |
| 2 | [Alert route dan notification](02-alerting-contact-point-routing-notification.md) | [LAB-02](lab/LAB-02-alert-firing-notification-failure-injection.md) |
| 3 | [SLO, budget, burn rate, runbook](03-slo-error-budget-burn-rate-runbook.md) | evaluasi dan evidence review |

## Prasyarat

Modul 7.1–7.4, GitOps Fase 6, Helm Fase 5, dan dasar incident/change management. Notification channel nyata bersifat optional dan tidak boleh diasumsikan tersedia.

## Acceptance Criteria

- [ ] Dashboard panel menampilkan query, unit, window, owner, dan data evidence.
- [ ] Alert memiliki query, threshold, window, severity, owner, runbook, no-data/error policy, dan route.
- [ ] Alert firing, Alertmanager receipt, notification delivered, dan incident action dipisahkan.
- [ ] SLO decision menyebut numerator/denominator, objective, budget, burn windows, serta caveat.

## Guardrail dan Status Runtime

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Webhook/Telegram/Slack token, Grafana credential, raw alert payload, PII, dan private key dilarang di repository/log/evidence. Dashboard, alert, notification, failure injection, dan SLO **belum diverifikasi** tanpa execution evidence.

## Troubleshooting dan Catatan SRE

Dashboard kosong bukan selalu backend down: cek datasource, tenant, time range, label, query, ingestion, dan clock. Alert `Firing` tidak membuktikan notification delivered. Burn-rate alert adalah symptom gate; runbook harus mengarahkan diagnosis dan mitigasi.

## Kaitan

- [Fase 6 GitOps](../fase-6-gitops/README.md) menggunakan alert/SLO sebagai promotion evidence.
- Fase 8 menggunakan notification, incident, error budget, dan runbook.
- [Fase 5 Helm](../fase-5-helm/README.md) menyediakan chart untuk dashboard/alert provisioning.
