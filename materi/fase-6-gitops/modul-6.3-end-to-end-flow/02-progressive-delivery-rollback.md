# 02 — Progressive Delivery dan Rollback

## Tujuan

Mengenalkan canary dan blue-green dengan Argo Rollouts tanpa mengklaim progressive delivery runtime sebelum traffic routing, metrics analysis, capacity, approval, dan rollback benar-benar dibuktikan.

## Canary

Canary menjalankan versi baru pada sebagian traffic atau capacity sebelum promotion penuh:

```text
stable workload/service
        +
canary workload/service
        → traffic split atau step weight
        → metric analysis
        → pause/approval
        → promote atau abort
```

Contoh desain non-secret:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: <approved-rollout-name>
spec:
  replicas: <approved-replica-count>
  strategy:
    canary:
      stableService: <stable-service-name>
      canaryService: <canary-service-name>
      steps:
        - setWeight: <approved-small-percentage>
        - pause: {}
        - setWeight: <approved-larger-percentage>
```

Yang harus ditentukan:

- traffic router atau service mechanism;
- metric query dan evaluation window;
- error/latency threshold;
- capacity headroom;
- pause, abort, dan owner approval;
- stable/canary service selectors;
- behavior saat node, image, dependency, atau telemetry gagal.

## Blue-Green

Blue-green mempertahankan environment stable (blue) dan kandidat (green), lalu memindahkan service/traffic setelah validation:

```text
blue stable → deploy green → smoke/metric validation
→ approval → switch active service → observe
→ rollback service ke blue bila gagal
```

Pertimbangkan kapasitas dua versi, database compatibility, connection draining, session state, cache, external side effects, dan rollback duration. Switch traffic tidak mengembalikan data yang telah berubah.

## Metrics Analysis dan Fase 7

Metric analysis harus mempunyai:

- sumber metric yang tersedia dan access boundary;
- query version/reference;
- baseline dan evaluation window;
- missing-data behavior;
- threshold dan abort policy;
- evidence query result yang sudah diredáksi.

Argo Rollouts `Healthy` atau step selesai bukan SLO. SLO gate memerlukan metrics/logs/traces dan definisi error budget dari Fase 7.

## Rollback dan Abort

| Kondisi | Tindakan awal |
|---|---|
| image pull/scheduling gagal | pause, perbaiki capacity/image, jangan promote |
| readiness gagal | abort/revert, ambil events redacted |
| latency/error naik | abort canary/traffic switch, cek telemetry |
| migration tidak compatible | stop; gunakan prosedur data recovery, bukan hanya Git revert |
| external side effect telah terjadi | koordinasikan compensating action |

Rollback reference harus berupa GitOps commit/Argo revision atau service switch yang dapat ditelusuri. `--atomic` atau controller rollback tidak menjamin data migration terbalik.

## Static Lane dan Runtime Lane

Static lane: review Rollout, service, analysis template, capacity, pause/abort, dan rollback. Runtime lane hanya pada disposable target dengan traffic generator, telemetry, approval, dan evidence.

Tanpa bukti tersebut, status canary/blue-green adalah **belum diverifikasi**.

## Acceptance Criteria

- [ ] Canary dan blue-green dibedakan.
- [ ] Stable/canary service, traffic, metric, pause, abort, dan capacity jelas.
- [ ] Rollback caveat migration/PVC/external side effect dijelaskan.
- [ ] SLO tidak disamakan dengan status rollout.
- [ ] Runtime claim hanya dibuat dengan evidence.

## Troubleshooting

- Canary tidak menerima traffic: cek router/service selector dan target platform.
- Analysis `NoData`: bedakan telemetry unavailable dari metric nol; default aman adalah pause.
- Blue-green overload: cek kapasitas dua environment dan resource quota.
- Abort tidak memulihkan aplikasi: periksa migration, external side effect, session, dan dependency.

## Kaitan

Praktikkan [LAB-02](lab/LAB-02-canary-blue-green-introduction.md), lalu hubungkan metrics ke Fase 7 Observability yang masih menyusul.
