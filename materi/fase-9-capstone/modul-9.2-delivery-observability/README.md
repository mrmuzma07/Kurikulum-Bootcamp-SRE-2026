# Modul 9.2 — Delivery, Observability, dan Recovery Evidence

> **Fokus:** menghubungkan commit sampai SLO, lalu membuktikan alert, notification, backup, dan restore pada disposable target.

## Tujuan dan Capaian

Peserta dapat:

- memetakan commit → CI → multi-arch digest → GitOps revision → ArgoCD → rollout → telemetry;
- memisahkan staging k3d dari production k3s dan menetapkan protected promotion;
- mendefinisikan MetalLB/Ingress evidence serta immutable artifact/provenance;
- membuat minimal lima alert meaningful dengan owner, severity, query, window, missing-data policy, runbook, route, dan notification boundary;
- menghubungkan Prometheus/Mimir, Alloy/Loki/Tempo, Grafana, dan Alertmanager ke SLO/error budget;
- membedakan backup completed dari restore dan application recovery yang tervalidasi.

## Prasyarat

- [Fase 6 — GitOps](../../fase-6-gitops/README.md).
- [Fase 7 — Observability](../../fase-7-observability/README.md).
- [Modul 9.1 — Platform Rebuild](../modul-9.1-platform-rebuild/README.md).
- [Modul 8.2 — Backup/Restore On-Prem](../../fase-8-sre-practices/modul-8.2-production-onprem/README.md).

## Rencana Sesi

| Sesi | Materi | Output |
|---|---|---|
| 1 | [CI/GitOps promotion](01-gitops-ci-promotion.md) | delivery chain dan rollback contract |
| 2 | [Telemetry, SLO, backup](02-observability-slo-backup-evidence.md) | alert matrix, SLO, restore order |
| 3 | [LAB-01](lab/LAB-01-end-to-end-promotion.md) | promotion evidence packet |
| 4 | [LAB-02](lab/LAB-02-telemetry-alert-backup-evidence.md) | alert/notification dan restore evidence |
| 5 | [Latihan dan kuis](evaluasi/latihan.md) | minimal 80%, tanpa guardrail violation |

## Deliverables

1. CI/GitOps evidence chain dengan digest dan revision.
2. MetalLB/Ingress traffic boundary dan smoke evidence.
3. Alert matrix minimal lima alert serta route/notification contract.
4. RED/USE/SLO dashboard dan error-budget decision.
5. Velero/etcd backup-restore plan, RPO/RTO, dan post-restore validation.

## Acceptance Criteria

- [ ] Production delivery menggunakan GitOps; tidak ada manual `kubectl apply` sebagai jalur normal.
- [ ] Digest, provenance, GitOps commit, Argo revision, rollout, smoke, telemetry, dan SLO decision saling terhubung.
- [ ] Lima alert memiliki metadata operasional dan notification evidence.
- [ ] Logs/traces memiliki redaction, bounded labels, sampling, dan retention policy.
- [ ] Restore dilakukan pada target disposable dan memvalidasi objects, PV/application, endpoint, telemetry, serta RPO/RTO.
- [ ] `Synced`, `Healthy`, `Firing`, atau `Completed` tidak dipakai sebagai bukti tunggal.

## Dua Lane

Static lane meninjau pipeline, query, alert, dashboard, backup contract, dan evidence schema. Runtime lane memerlukan target disposable, approval, fault/restore scope, timeout, rollback/cleanup, dan evidence redacted.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan memasukkan token, webhook, password, TLS key, object-storage key, raw log/trace, PII, rendered Secret, atau raw backup ke repository, dashboard, alert, atau artifact. Jangan memakai `--set` untuk secret, production chaos, atau restore ke cluster aktif.

## Troubleshooting

- **Pipeline green tetapi app gagal:** cek target, digest, GitOps revision, rollout, dependency, traffic, dan telemetry.
- **Argo `Healthy` tetapi SLO buruk:** cek request eligibility, error/latency window, dependency, dan missing-data policy.
- **Alert firing tanpa notifikasi:** telusuri rule → Alertmanager route → contact boundary → receipt; jangan menandai paging sukses dari status rule saja.
- **Backup completed tetapi restore gagal:** cek restore order, PV/database consistency, secret references, DNS, dan application validation.
- **Trace tidak berkorelasi:** cek propagation, sampling, collector pipeline, redaction, dan retention.

## Kaitan

- [Modul 9.3](../modul-9.3-game-day-graduation/README.md) memakai alert, SLO, runbook, dan recovery evidence.
- [Fase 8](../../fase-8-sre-practices/README.md) menyediakan policy, incident command, backup, dan readiness decision.

## Status Runtime

CI runner/registry, promotion, Argo sync, MetalLB traffic, telemetry correlation, alert notification, Velero restore, etcd restore, dan SLO outcome: **belum diverifikasi** tanpa execution evidence.
