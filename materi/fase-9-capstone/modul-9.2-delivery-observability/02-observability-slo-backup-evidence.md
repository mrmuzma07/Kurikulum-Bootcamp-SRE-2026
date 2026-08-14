# 9.2.2 — Telemetry, SLO, Alert, dan Recovery Evidence

## Observability Flow

```text
workload/exporter/OTel
→ Prometheus scrape atau Alloy collect
→ Mimir metrics / Loki logs / Tempo traces
→ PromQL/LogQL/trace query
→ Grafana RED/USE/SLO panel
→ rule evaluation
→ Alertmanager route/notification
→ runbook dan incident decision
```

Labels harus bounded. Jangan memasukkan user ID, request ID tak terbatas, raw URL, token, authorization header, atau arbitrary input sebagai label. Log dan trace wajib memiliki redaction, PII policy, sampling, storage cost, dan retention.

## Alert Contract

Minimal lima alert meaningful dapat memakai kelas berikut sebagai contoh; query aktual harus disesuaikan service:

| Alert | Owner | Severity | Signal | Runbook |
|---|---|---|---|---|
| HighErrorRate | app owner | page | error ratio melewati objective | runbook app error |
| HighLatency | app owner | page | latency percentile melewati window | runbook latency |
| NodeDiskPressure | platform | ticket/page | filesystem headroom rendah | runbook disk full |
| KubeNodeNotReady | platform | page | node Ready false selama window | runbook worker down |
| MetalLBAddressPoolExhausted | network/platform | page | available IP pool rendah | runbook MetalLB |

Setiap alert wajib memiliki owner, severity, query reference, evaluation window, missing-data policy, runbook, route, notification boundary, dan evidence receipt. `Firing` tanpa receipt bukan bukti paging.

## SLO dan Error Budget

SLO contract harus mendefinisikan service, objective, window, eligible traffic, numerator, denominator, exclusion, missing-data policy, owner, dan review cadence. Untuk objective 99.9%:

```text
error budget fraction = 1 - 0.999 = 0.001
budget requests = eligible requests × 0.001
```

Angka contoh bukan credential. Keputusan promotion harus mengaitkan burn rate dan window dengan risk, bukan sekadar panel hijau.

## Backup Classes

Pisahkan:

1. Kubernetes objects dan metadata.
2. PV/application data serta database consistency.
3. etcd/control-plane state.

Catat retention, encryption/key custody, restore order, owner, RPO, RTO, dan post-restore validation. Velero `Completed` atau etcd snapshot tersedia hanya membuktikan pembuatan artifact, bukan recovery aplikasi.

## Restore Order

```text
verify disposable target and isolation
→ restore control-plane/cluster prerequisite as approved
→ restore objects and namespaces
→ restore PV/application data with consistency check
→ restore dependencies and secret references through approved mechanism
→ validate service/Ingress/MetalLB
→ validate metrics/logs/traces
→ validate SLO and RPO/RTO
→ capture redacted evidence and cleanup
```

Jangan restore ke cluster aktif, jangan mencetak archive, encryption key, credential, atau raw restore output. Database restore harus mempertimbangkan point-in-time consistency dan migration compatibility.

## Evidence Record

```yaml
revision: <git-revision>
telemetry_window_utc: <window>
alert_summary: <rule-status-and-receipt-summary>
notification_boundary: <redacted-reference>
slo: <objective-and-outcome>
backup_id: <identifier-only>
checksum_summary: <redacted-summary>
restore_target: <disposable-target>
rpo_rto: <measured-or-unknown>
validation: <objects-data-endpoint-telemetry>
```

## Static dan Runtime

Static review membuktikan desain query, alert, dashboard, backup order, dan redaction. Runtime membutuhkan query result, notification receipt, restore validation, dan outcome SLO/RPO/RTO pada target yang disetujui.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menaruh Grafana password, webhook secret, metric credential, Loki/Mimir/Tempo credential, TLS private key, object-storage key, PII, raw alert payload, atau raw backup pada materi dan evidence.

## Kaitan

- Promotion dipraktikkan di [LAB-01](lab/LAB-01-end-to-end-promotion.md).
- Alert dan restore dipraktikkan di [LAB-02](lab/LAB-02-telemetry-alert-backup-evidence.md).
- Game Day memakai signal ini melalui [Modul 9.3](../modul-9.3-game-day-graduation/README.md).
