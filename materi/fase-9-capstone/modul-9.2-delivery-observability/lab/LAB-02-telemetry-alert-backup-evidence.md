# LAB-02 — Telemetry, Alert, dan Backup/Restore Evidence

> **Target:** membuktikan jalur signal → notification → runbook serta backup/restore pada disposable target, tanpa menyamakan status lokal dengan recovery aplikasi.

## Mode Lab

Static lane membuat alert matrix, query reference, dashboard, restore order, RPO/RTO contract, dan simulated evidence. Runtime lane memerlukan target disposable, approval, one bounded fault atau isolated restore, timeout, cleanup, dan evidence redaction.

## Prasyarat

- Fase 7 dan Fase 8 selesai.
- Prometheus/Alloy/Mimir/Loki/Tempo/Grafana/Alertmanager design tersedia.
- Lima alert memiliki owner, severity, query, window, missing-data policy, runbook, route, dan notification boundary.
- Velero/etcd procedure, retention, encryption/key custody, restore order, and application validation disetujui.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Failure injection dan restore hanya pada disposable target. Jangan restore ke cluster aktif, jangan mencetak webhook, password, object-storage key, encryption key, raw archive, raw alert payload, atau raw logs/traces.

## Evidence Contract

```yaml
lab: LAB-02
revision: <git-revision>
target: <disposable-target>
baseline_utc: <window>
fault_or_restore_scope: <bounded-scope>
alert_summary: <rule-state-summary>
notification_receipt: <redacted-reference>
runbook_decision: <runbook-id-and-action>
backup_id: <identifier-only>
restore_target: <isolated-target>
validation: <objects-data-endpoint-telemetry>
rpo_rto: <measured-or-unknown>
cleanup: <pass-fail-unknown>
known_gaps: [<gap>]
```

## Bagian A — Alert dan Notification

### 1. Baseline

Verifikasi target, namespace, owner, traffic, dashboard, existing alerts, and cleanup. Capture only redacted status summary.

### 2. Fault Bounded

Pilih satu fault yang disetujui, misalnya workload replica reduction pada disposable namespace atau controlled disk-pressure simulation. Tetapkan timeout dan stop condition sebelum injection. Jangan menyentuh production.

### 3. Trace Signal Chain

Ikuti:

```text
fault
→ metric/log/trace change
→ rule pending/firing
→ Alertmanager route
→ disposable notification receipt
→ runbook selection
→ mitigation
→ recovery/post-check
```

Catat UTC timestamp dan identifier, bukan raw payload atau credential. Missing data harus diperlakukan `unknown`/pause sesuai policy, bukan sukses.

### 4. Cleanup

Rollback fault, verify baseline, clear disposable notification, and confirm no shared resource changed. Bila alert tidak sampai notification, status evidence `failed`/`unknown`, bukan `passed`.

## Bagian B — Backup dan Isolated Restore

### 1. Backup Scope

Pisahkan Kubernetes objects, PV/application data, database consistency, dan etcd/control-plane snapshot. Catat backup ID, retention, checksum summary, encryption/key custody reference, dan owner.

### 2. Isolate Restore Target

Buat atau pilih target disposable terisolasi. Verifikasi namespace, storage, DNS, credentials reference, network, and cleanup. Jangan restore pada cluster aktif atau menimpa data existing.

### 3. Restore Order

Ikuti prosedur approved:

```text
cluster prerequisite
→ objects/namespaces
→ PV/application data
→ dependencies/secret references
→ endpoint/Ingress
→ telemetry
→ SLO and RPO/RTO validation
```

Velero `Completed` dan etcd snapshot tersedia bukan bukti aplikasi pulih. Validasi object count/summary, PV/application consistency, endpoint, logs, traces, metrics, and user-visible SLO.

### 4. Measure and Close

Catat waktu recovery hanya bila start/end didefinisikan dan dapat dibuktikan. Bandingkan actual RPO/RTO dengan objective. Cleanup isolated target dan simpan evidence redacted.

## Acceptance Criteria

- [ ] Baseline, target, owner, fault/restore scope, timeout, dan stop condition terdokumentasi.
- [ ] Minimal lima alert contract direview dan satu chain notification diuji atau diberi status belum diverifikasi.
- [ ] Runbook dan mitigation/rollback digunakan.
- [ ] Backup classes, restore order, data consistency, key custody, RPO/RTO, dan validation didefinisikan.
- [ ] Restore hanya pada target disposable terisolasi.
- [ ] Evidence tidak berisi raw payload, credential, PII, raw archive, atau raw log.

## Troubleshooting

| Gejala | Tindakan |
|---|---|
| Rule firing tetapi notification tidak diterima | Trace route/contact boundary; tandai paging belum terbukti |
| Dashboard kosong | Bedakan no traffic, scrape gap, remote-write, query, dan retention |
| Backup selesai tetapi app gagal setelah restore | Periksa PV/database consistency, dependencies, DNS, secret references, dan migration |
| RPO/RTO unknown | Jangan mengklaim tercapai; definisikan clock dan rerun pada disposable scope |
| Trace tidak punya correlation | Periksa propagation, sampling, collector, redaction, dan storage |

## Lanjut

Gunakan evidence dan runbook ini untuk [Modul 9.3 — Game Day](../../modul-9.3-game-day-graduation/README.md).
