# Fase 7 — Observability: Prometheus · Alloy · Grafana · Mimir · Loki · Tempo

> **Tujuan fase:** membangun jalur telemetry dari collection sampai query, dashboard, alert, notification, runbook, dan keputusan SLO dengan evidence yang dapat diaudit.

## Durasi dan Modul

Minggu 12–14 — lima modul dengan static review dan optional disposable runtime lane.

| Modul | Fokus | Status |
|---|---|---|
| 7.1 | Fondasi Metrics: Prometheus | ✅ Tersedia |
| 7.2 | Grafana Alloy & Pipeline Telemetry | ✅ Tersedia |
| 7.3 | Metrics Skala Production: Mimir | ✅ Tersedia |
| 7.4 | Logs & Traces: Loki + Tempo | ✅ Tersedia |
| 7.5 | Grafana, Alerting, dan SLO | ✅ Tersedia |

## Capaian Fase

- [ ] Menjelaskan pull/scrape Prometheus, TSDB, service discovery, target/job/instance, exporter, dan batas single-node.
- [ ] Menulis PromQL untuk CPU, memory, disk, request rate, error rate, dan p95 latency.
- [ ] Membuat recording rule dan alerting rule dengan label bounded, owner, severity, runbook, window, serta policy missing data.
- [ ] Menjelaskan Alertmanager grouping, routing, inhibition, silence, deduplication, dan kegagalan notification.
- [ ] Merancang Alloy sebagai collector metrics/logs/traces dengan redaction, retry, queue, backpressure, dan resource boundary.
- [ ] Menjelaskan OpenTelemetry receiver, propagation, sampling, correlation, dan OTLP.
- [ ] Menjelaskan arsitektur Mimir, remote-write, multi-tenancy, object storage, retention, HA, capacity, dan recovery.
- [ ] Menulis LogQL dengan label low-cardinality dan menghubungkan log, trace, dan metrics melalui trace ID.
- [ ] Mendesain dashboard USE untuk infra dan RED untuk service sebagai code.
- [ ] Menyusun SLO availability/latency/error rate, error budget, burn-rate alert, runbook, dan promotion gate.
- [ ] Membuktikan setiap tahap collection → storage → query → dashboard → alert → notification → SLO secara terpisah.

> Target `UP`, metric tersedia, Helm release `deployed`, dashboard dapat dibuka, rule `Firing`, Alertmanager menerima alert, dan notification diterima **bukan status yang sama**. Tanpa evidence runtime yang sesuai, statusnya **belum diverifikasi**.

## Alur Signal

```text
workload/exporter/OTel
  → Prometheus scrape atau Alloy collect
  → Mimir/Loki/Tempo storage
  → PromQL/LogQL/trace query
  → Grafana panel
  → rule evaluation
  → Alertmanager/contact point
  → notification
  → runbook/incident
  → SLO dan error-budget decision
```

## Rencana Belajar

| Hari | Materi | Praktik |
|---|---|---|
| 1–2 | [Modul 7.1](modul-7.1-prometheus/README.md) | [LAB-01](modul-7.1-prometheus/lab/LAB-01-prometheus-node-blackbox-scrape.md) + [LAB-02](modul-7.1-prometheus/lab/LAB-02-promql-rules-alertmanager.md) |
| 3–4 | [Modul 7.2](modul-7.2-alloy-telemetry-pipeline/README.md) | [LAB-01](modul-7.2-alloy-telemetry-pipeline/lab/LAB-01-alloy-metrics-remote-write.md) + [LAB-02](modul-7.2-alloy-telemetry-pipeline/lab/LAB-02-alloy-logs-otlp-pipeline.md) |
| 5–6 | [Modul 7.3](modul-7.3-mimir/README.md) | [LAB-01](modul-7.3-mimir/lab/LAB-01-mimir-remote-write-query.md) + [LAB-02](modul-7.3-mimir/lab/LAB-02-mimir-retention-capacity-recovery.md) |
| 7–8 | [Modul 7.4](modul-7.4-loki-tempo/README.md) | [LAB-01](modul-7.4-loki-tempo/lab/LAB-01-alloy-loki-log-pipeline.md) + [LAB-02](modul-7.4-loki-tempo/lab/LAB-02-tempo-trace-log-metrics-correlation.md) |
| 9–10 | [Modul 7.5](modul-7.5-grafana-alerting-slo/README.md) | [LAB-01](modul-7.5-grafana-alerting-slo/lab/LAB-01-dashboard-data-source-evidence.md) + [LAB-02](modul-7.5-grafana-alerting-slo/lab/LAB-02-alert-firing-notification-failure-injection.md) |

## Dua Lane Praktik

### Static lane

```text
telemetry design → non-secret config/query review → render/schema review
→ cardinality/retention/capacity review → dashboard/alert/SLO review
→ evidence schema dan runbook
```

Static lane dapat dilakukan tanpa cluster, Helm, backend, notification channel, atau credential. Hasilnya tidak boleh ditulis sebagai bukti scrape, ingestion, query data, dashboard data, alert firing, notification, atau SLO.

### Disposable runtime lane

```text
verify tools/context/namespace/storage/target ownership
→ review diff + approval → deploy/upgrade via approved Helm/GitOps path
→ verify collectors/backends/targets → generate bounded synthetic traffic
→ verify metrics/logs/traces/query/panel/rule/Alertmanager/notification
→ optional scoped failure injection → capture redacted evidence
→ rollback/cleanup pada disposable scope
```

Tidak ada runtime target atau context aktif pada preflight sebelumnya. Oleh karena itu deployment stack, dashboard data, alert firing, notification, dan SLO saat ini **belum diverifikasi**.

## Boundary Ownership

| Layer | Tanggung jawab |
|---|---|
| OpenTofu | VM, network, storage, metadata non-secret |
| Ansible | OS/bootstrap/readiness/k3s host configuration |
| Kubernetes/k3s | scheduling, service discovery, storage, cluster health |
| Helm | chart packaging, templating, release rendering |
| GitLab CI/GitOps/ArgoCD | validation, desired state, promotion, reconciliation |
| Prometheus | scrape, local TSDB, rules, alert generation |
| Alloy | collection, processing, forwarding, queue/retry |
| Mimir | long-term metrics storage/query |
| Loki | log storage/query |
| Tempo | trace ingestion/storage/query |
| Grafana/Alertmanager | visualization, alert routing, notification boundary |
| SRE/runbook | incident action, SLO/error-budget decision |

## Deliverables

1. Design Prometheus + node_exporter + blackbox_exporter dan 10 query PromQL penting.
2. Pipeline Alloy untuk metrics, logs, dan OTLP traces dengan redaction serta backpressure policy.
3. Model Mimir terpusat dengan remote-write, object storage, retention, tenancy, dan capacity plan.
4. Query LogQL, Tempo correlation, dashboard USE/RED, dan dashboard-as-code.
5. Alert error rate >5%, p95 >500ms, disk >85%, node down sebagai design/test scenario dengan threshold dan missing-data policy.
6. Evidence chain terpisah dari collection sampai SLO serta runbook failure injection disposable.
7. Nilai kuis minimal **80%** pada setiap modul tanpa pelanggaran guardrail.

## Guardrail Fase

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menulis Grafana credential, API token, webhook secret, object-storage key, TLS private key, exporter credential, OTLP credential, raw Kubernetes Secret, raw dashboard export yang memuat secret, atau raw notification payload ke repository, log, evidence, atau artifact.
- Jangan memasukkan PII, bearer token, authorization header, password, atau private key ke log, trace, label, dashboard variable, query result, atau alert notification.
- Metric label, Loki stream label, dan trace attribute harus bounded; jangan gunakan user ID, raw URL, request ID tak terbatas, token, atau arbitrary input sebagai label/index.
- `helm lint`, `helm template`, config parse, target `UP`, rule `Firing`, dan Grafana UI terbuka bukan bukti SLO.
- Verifikasi context, namespace, cluster, storage, ownership, dan target sebelum install/upgrade/rollback/uninstall. Jangan menjalankan failure injection atau chaos pada production.
- Failure injection hanya pada target disposable dengan approval, traffic/scope control, stop condition, rollback, access recovery, dan evidence redacted.
- Jangan memakai `--set` untuk secret. Reference secret manager atau mechanism yang disetujui; ciphertext tidak menggantikan key management, rotation, backup, dan recovery.
- Runtime observability stack, remote-write, Loki/Tempo ingestion, dashboard data, alert firing, notification, dan SLO berstatus **belum diverifikasi** tanpa evidence aktual.

## Kaitan

- [Fase 2 — Kubernetes](../fase-2-kubernetes/README.md) menyediakan target, context safety, node metrics, rollout, dan troubleshooting.
- [Fase 5 — Helm](../fase-5-helm/README.md) menyediakan chart, values, lifecycle, dan observability packaging.
- [Fase 6 — GitOps](../fase-6-gitops/README.md) menyediakan delivery/reconciliation; observability menjadi evidence gate.
- Fase 8 akan menggunakan alert, incident, error budget, dan runbook yang dirancang di sini.

## Status Runtime

Materi Fase7: **tersedia**. Prometheus/Alloy/Mimir/Loki/Tempo/Grafana/Alertmanager deployment, scrape semua node k3s, remote-write, query data, dashboard data, alert firing, notification, failure injection, dan SLO evidence: **belum diverifikasi** tanpa execution evidence.
