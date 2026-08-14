# Modul 7.2 — Grafana Alloy & Pipeline Telemetry

> **Tujuan:** merancang collector metrics, logs, dan traces yang dapat melakukan processing, redaction, retry, queue, dan forwarding secara aman.

## Capaian

- [ ] Menjelaskan tiga pilar observability dan konsep OpenTelemetry.
- [ ] Memahami component graph Alloy, debug, health, dan failure boundary.
- [ ] Mengirim Prometheus metrics ke Mimir, logs ke Loki, dan OTLP traces ke Tempo.
- [ ] Mendesain DaemonSet/Helm, permissions, resource limit, queue, retry, dan backpressure.
- [ ] Membatasi labels/attributes serta meredaksi PII dan credential.

## Materi dan Praktik

| Hari | Materi | Lab |
|---|---|---|
| 1 | [Komponen dan tiga pilar](01-alloy-components-metrics-logs-traces.md) | [LAB-01](lab/LAB-01-alloy-metrics-remote-write.md) |
| 2 | [OTel, DaemonSet, Helm, debugging](02-otel-pipeline-daemonset-helm-debugging.md) | [LAB-02](lab/LAB-02-alloy-logs-otlp-pipeline.md) |

## Prasyarat

Modul 7.1, dasar Kubernetes DaemonSet/ServiceAccount, Helm Fase 5, dan konsep OTLP. Runtime harus disposable serta memiliki backend yang disetujui.

## Acceptance Criteria

- [ ] Pipeline memiliki input, processing, output, queue/retry, failure metric, dan ownership.
- [ ] Host log access dan network policy dibatasi; secret hanya direferensikan.
- [ ] Backpressure dan dropped telemetry mempunyai policy dan evidence field.
- [ ] Config parse/Helm render tidak disamakan dengan data ingestion.

## Guardrail dan Status Runtime

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Jangan memasukkan OTLP endpoint credential, TLS key, PII, bearer token, atau log mentah ke file/evidence. Alloy deployment dan forwarding **belum diverifikasi** tanpa runtime evidence.

## Troubleshooting dan Catatan SRE

Jika backend lambat, queue dapat penuh dan data drop; retry tanpa bound dapat menghabiskan memory. Pisahkan collector health, transport success, backend ingestion, dan query availability. Sampling trace mengurangi biaya tetapi dapat menghilangkan detail.

## Kaitan

Hubungkan ke [Mimir](../modul-7.3-mimir/README.md), [Loki + Tempo](../modul-7.4-loki-tempo/README.md), dan GitOps [Fase 6](../fase-6-gitops/README.md).
