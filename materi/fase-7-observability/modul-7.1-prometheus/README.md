# Modul 7.1 — Fondasi Metrics: Prometheus

> **Tujuan:** memahami bahasa metrics dari exporter dan scrape sampai PromQL, rules, Alertmanager, dan batas Prometheus single-node.

## Capaian

- [ ] Menjelaskan pull/scrape, TSDB, target/job/instance, service discovery, relabeling, dan target health.
- [ ] Membedakan node_exporter untuk host metrics dan blackbox_exporter untuk probe.
- [ ] Memilih counter, gauge, histogram, atau summary dengan benar.
- [ ] Menulis query CPU, memory, disk, request rate, error rate, dan p95 latency.
- [ ] Mendesain recording/alerting rules dan Alertmanager route tanpa secret.
- [ ] Menghitung concern retention, cardinality, scrape failure, dan missing data.

## Materi dan Praktik

| Hari | Materi | Lab |
|---|---|---|
| 1 | [Arsitektur, exporter, PromQL](01-arsitektur-scrape-exporter-promql.md) | [LAB-01](lab/LAB-01-prometheus-node-blackbox-scrape.md) |
| 2 | [Rules, Alertmanager, retention](02-rules-alertmanager-retention-cardinality.md) | [LAB-02](lab/LAB-02-promql-rules-alertmanager.md) |

## Prasyarat

Modul Kubernetes 2.1/2.4, Helm Fase 5, dasar label/Service, dan pemahaman bahwa target Kubernetes tidak otomatis berarti service sehat. `helm`/cluster hanya diperlukan untuk runtime lane.

## Acceptance Criteria

- [ ] Scrape contract memiliki job, target, interval, timeout, relabeling, ownership, dan stop condition.
- [ ] Query menjelaskan unit, range window, aggregation, dan perilaku no-data.
- [ ] Rule memiliki owner, severity, `for`, runbook reference, dan bounded labels.
- [ ] Target `UP`, metric tersedia, rule evaluated, Alertmanager receipt, dan notification dipisahkan.

## Guardrail dan Status Runtime

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Jangan mencetak exporter credential, webhook, TLS key, atau raw alert payload. Prometheus install, node scrape, blackbox probe, alert firing, dan notification **belum diverifikasi** tanpa execution evidence.

## Troubleshooting dan Catatan SRE

Target `DOWN` dapat berarti network, DNS, TLS, auth, timeout, exporter, atau clock issue; jangan langsung menaikkan timeout. Metric ada tidak berarti query benar atau SLO tercapai. Cardinality dan retention adalah capacity decision sejak awal.

## Kaitan

Lanjutkan ke [Modul 7.2 Alloy](../modul-7.2-alloy-telemetry-pipeline/README.md), [Modul 7.3 Mimir](../modul-7.3-mimir/README.md), dan [Modul 7.5 Alerting/SLO](../modul-7.5-grafana-alerting-slo/README.md).
