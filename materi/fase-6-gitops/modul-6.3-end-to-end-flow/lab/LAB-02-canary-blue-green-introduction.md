# LAB-02 — Pengenalan Canary dan Blue-Green

## Tujuan

Mereview desain Argo Rollouts, traffic/capacity/metrics gate, dan rollback tanpa melakukan progressive delivery pada production.

## Prasyarat dan Guardrail

Gunakan [materi progressive delivery](../02-progressive-delivery-rollback.md). Runtime optional hanya disposable, memakai telemetry yang tersedia, traffic generator terkendali, dan approval.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menulis metric credential, kubeconfig, raw Secret, private key, atau decrypted SOPS. Jangan menganggap manifest valid sebagai runtime evidence.

## Lane A — Static Simulation

### 1. Canary review

Review placeholder berikut:

```yaml
strategy:
  canary:
    stableService: <stable-service-name>
    canaryService: <canary-service-name>
    steps:
      - setWeight: <small-percentage>
      - pause: {}
      - analysis:
          templates:
            - templateName: <approved-analysis-template>
      - setWeight: <full-promotion-percentage>
```

Jawab:

- bagaimana traffic split diterapkan;
- metric/error/latency threshold;
- kapan pause dan siapa approver;
- bagaimana abort memilih stable;
- apakah capacity cukup untuk dua versi;
- apa yang terjadi bila metric `NoData`.

### 2. Blue-green review

Rancang blue/green service, active/preview endpoint, validation window, connection drain, session/cache behavior, dan switch/rollback reference. Bahas database compatibility dan external side effect.

### 3. Failure matrix

| Failure | Signal | Action | Evidence |
|---|---|---|---|
| image pull | pod events | pause/fix | redacted events |
| readiness | rollout status | abort | revision/outcome |
| error/latency | metric window | abort | query reference |
| NoData | analysis status | pause, investigate | telemetry status |
| capacity | scheduling/quota | stop | resource summary |
| migration | app/data signal | data recovery runbook | incident reference |

## Lane B — Optional Disposable Runtime

1. Verifikasi target/namespace, Rollout, stable/canary service, router, metric source, dan capacity.
2. Deploy candidate dari immutable GitOps revision ke disposable target.
3. Jalankan traffic generator dengan rate/timeout yang disetujui.
4. Naikkan weight sesuai step; pause pada gate.
5. Evaluasi metrics dan readiness; abort jika threshold gagal.
6. Simpan rollout revision, weight, metric summary, decision, waktu, dan status redacted.
7. Bersihkan target melalui scoped procedure.

Canary/blue-green runtime **belum diverifikasi** tanpa evidence aktual dari rollout, traffic, metrics, dan rollback/abort.

## Stop Conditions

- router tidak dapat membedakan stable/canary;
- analysis query tidak tersedia atau `NoData` tidak punya policy;
- capacity tidak mencukupi;
- production target terpilih;
- database migration tidak backward-compatible;
- abort/recovery belum diuji atau tidak terdokumentasi.

## Acceptance Criteria

- [ ] Canary dan blue-green manifest memiliki stable/preview boundary.
- [ ] Traffic, metric, capacity, pause, abort, dan rollback jelas.
- [ ] `NoData`, migration, PVC, dan external side effect ditangani.
- [ ] Runtime hanya disposable dan evidence lengkap bila dijalankan.

## Kaitan

Lanjutkan ke evaluasi [Modul 6.3](../evaluasi/latihan.md) dan telemetry Fase 7 Observability yang masih menyusul.
