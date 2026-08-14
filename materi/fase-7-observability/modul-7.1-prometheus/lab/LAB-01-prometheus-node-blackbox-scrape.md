# LAB-01 — Prometheus, node_exporter, dan blackbox scrape

## Tujuan

Menyusun scrape contract untuk node metrics dan blackbox probe, lalu memisahkan static validation dari target/query runtime evidence.

## Prasyarat dan Guardrail

Baca [Arsitektur scrape](../01-arsitektur-scrape-exporter-promql.md). Runtime hanya target disposable yang disetujui.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menulis exporter credential, TLS private key, raw target inventory, atau production address ke repository/evidence.

## Lane A — Static Simulation

1. Tulis `job_name`, interval, timeout, target allowlist, ownership, environment, dan relabeling bounded.
2. Pilih node_exporter metrics untuk CPU/memory/disk/network.
3. Pilih blackbox module untuk HTTP/TCP/DNS dan definisikan expected success.
4. Review label cardinality, network policy, exporter RBAC, resource limit, and failure matrix.
5. Tulis 10 query PromQL: CPU, memory, disk, filesystem, load, network, target up, request rate, error ratio, p95.

Contoh target non-secret:

```yaml
job_name: <approved-node-job>
metrics_path: /metrics
static_configs:
  - targets: [<disposable-node>:<approved-port>]
    labels:
      environment: <lab>
      owner: <team-reference>
```

## Lane B — Optional Disposable Runtime

1. Verifikasi `kubectl` context aktif, namespace, cluster label, target ownership, dan tidak ada production target.
2. Pastikan Prometheus/node_exporter/blackbox_exporter/Helm tersedia; review diff dan approval.
3. Deploy atau update hanya pada namespace disposable dengan timeout dan stop condition.
4. Verifikasi Pod readiness, Service/target discovery, scrape status, exporter response class, dan resource usage.
5. Jalankan query dengan time range bounded; simpan summary series/value/timestamp yang sudah diredáksi.
6. Hentikan bila target di luar allowlist, exporter meminta secret di log, atau storage/CPU tidak cukup.
7. Rollback/uninstall hanya dengan scoped procedure dan cleanup evidence.

Tanpa langkah dan evidence tersebut, scrape runtime **belum diverifikasi**.

## Evidence Template

```text
lab: LAB-01
config_revision: <commit-or-chart-reference>
context: <redacted-context-reference>
target_scope: <approved-count-and-class>
scrape_status: <summary>
query_summary: <metric-names-and-window>
resource_summary: <redacted>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Target/job/instance dan exporter purpose benar.
- [ ] Allowlist, labels, resource, network, dan secret boundary direview.
- [ ] Query memiliki window/unit/aggregation.
- [ ] Runtime claim hanya dengan evidence aktual.

## Troubleshooting

Target down: cek DNS, route, TLS/auth reference, timeout, exporter process, dan clock. Metric kosong: cek discovery, relabel drop, time range, dan ingestion delay.

## Kaitan

Lanjutkan ke [LAB-02 rules](LAB-02-promql-rules-alertmanager.md) dan [Modul 7.2 Alloy](../../modul-7.2-alloy-telemetry-pipeline/README.md).
