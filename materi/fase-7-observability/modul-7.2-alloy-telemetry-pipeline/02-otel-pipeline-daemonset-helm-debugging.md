# 02 — OTel Pipeline, DaemonSet, Helm, dan Debugging

## Tujuan

Mendesain Alloy DaemonSet melalui Helm dan OpenTelemetry pipeline yang dapat diperiksa tanpa mengekspos credential atau data sensitif.

## 1. OTLP Contract

Dokumentasikan protocol (`grpc`/`http`), endpoint reference, timeout, retry, batch size, compression, auth/TLS reference, resource attributes, sampling, dan expected backend. `service.name`, `service.namespace`, `deployment.environment`, dan bounded cluster labels membantu correlation; jangan memasukkan secret atau arbitrary input.

## 2. DaemonSet Design

DaemonSet cocok untuk node-local scrape/log collection tetapi memiliki blast radius per node. Review:

- nodeSelector/toleration dan architecture ARM64/AMD64;
- ServiceAccount/RBAC minimal;
- hostPath hanya path yang dibutuhkan dan read-only;
- CPU/memory request/limit dan priority;
- liveness/readiness/collector health;
- PodSecurity/network policy;
- config reload dan rollback;
- queue storage/WAL bila digunakan.

Contoh values non-secret:

```yaml
controller:
  type: daemonset
alloy:
  configMap:
    name: <approved-alloy-config>
resources:
  requests:
    cpu: <approved-cpu>
    memory: <approved-memory>
  limits:
    cpu: <approved-cpu-limit>
    memory: <approved-memory-limit>
securityContext:
  runAsNonRoot: true
```

## 3. Helm/GitOps Boundary

`helm lint` dan `helm template` memeriksa packaging/rendering. GitOps review memeriksa chart version, image digest, values, namespace, RBAC, hostPath, and destination. Install/upgrade harus pada disposable target dengan approval, timeout, stop condition, and cleanup. Jangan gunakan `--set` untuk secret.

## 4. Debugging Tanpa Data Leakage

Ambil status component, queue depth, drop counter, backend response class, config hash, dan revision; redaksi payload/log lines. Jangan menjalankan debug yang mencetak environment variable, OTLP headers, decrypted values, atau raw application logs. Korelasikan timestamp dan component ID, bukan credential.

## Acceptance Criteria

- [ ] OTLP contract lengkap dan secret-safe.
- [ ] DaemonSet memiliki ARM64/AMD64, RBAC, hostPath, resource, and network review.
- [ ] Helm/GitOps path dan rollback jelas.
- [ ] Debug evidence hanya metadata redacted.

## Troubleshooting dan Catatan SRE

Config valid tetapi collector crash dapat disebabkan component dependency, permission, protocol mismatch, atau resource limit. Collector ready tetapi no data dapat berarti source/processor/exporter path. Bedakan config load, component health, transport, backend ingestion, dan query.

## Kaitan

Praktikkan [LAB-02](lab/LAB-02-alloy-logs-otlp-pipeline.md), lalu lanjutkan ke [Loki + Tempo](../modul-7.4-loki-tempo/README.md).

## Status Runtime

DaemonSet install, config reload, log/trace forwarding, dan backend query **belum diverifikasi**.
