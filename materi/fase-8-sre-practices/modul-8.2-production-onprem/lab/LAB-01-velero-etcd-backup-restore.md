# LAB-01 — Velero dan etcd Backup/Restore

## Tujuan

Mereview serta, bila environment disposable tersedia, menguji backup/restore Kubernetes objects, PV/application data, dan etcd dengan RPO/RTO serta evidence chain yang benar.

## Prasyarat dan Guardrail

Gunakan [backup/restore theory](../02-backup-restore-upgrade-security-dr.md) dan [operasi etcd](../../../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan restore ke cluster aktif, jangan menjalankan `cluster-reset`, jangan mencetak kubeconfig, backup credential, encryption key, raw archive, rendered Secret, atau raw Velero/etcd output.

## Lane A — Static Simulation

### 1. Scope and contract

```text
cluster: <disposable-cluster>
namespace: <disposable-namespace>
application: <approved-application>
backup_location: <approved-reference>
velero_scope: <objects-and-pv-scope>
etcd_scope: <approved-control-plane-scope>
rpo: <approved-duration>
rto: <approved-duration>
retention: <approved-retention>
key_custody: <approved-key-boundary>
restore_order: <approved-order>
validation: <application-and-slo-check>
```

### 2. Evidence matrix

| Stage | Expected evidence | Not sufficient alone |
|---|---|---|
| preflight | context/namespace/target/approval | tool version |
| backup | backup ID, scope, timestamp, checksum summary | `Completed` |
| restore objects | resource summary/revision | object created |
| restore PV/data | mount/data consistency summary | PVC Bound |
| application | readiness, smoke, dependency check | pod Running |
| SLO | representative metric window | dashboard open |
| cleanup | scoped target cleanup | command exit 0 |

### 3. RPO/RTO calculation

Tulis start/end UTC, last valid backup, data loss estimate, restore duration, validation duration, and whether objective met. Jika tidak ada execution, isi `not-run` dan status **belum diverifikasi**.

## Lane B — Optional Disposable Runtime

1. Verifikasi `kubectl` context, namespace, cluster identity, storage, backup location, access recovery, and non-production target.
2. Review scoped Velero/etcd plan/diff and approval. Record command summary without credential values.
3. Create backup within approved scope; capture identifier, timestamp, result summary, checksum summary, and retention.
4. Inject or remove only a disposable test object/data according to plan.
5. Restore into isolated namespace/cluster per approved order; do not overwrite unrelated resources.
6. Validate object, PV/application/database consistency, endpoint, telemetry, and SLO window.
7. If etcd restore is tested, use disposable control plane and official procedure; do not use active production cluster.
8. Record RPO/RTO outcome, rollback/cleanup, owner, and redacted evidence.

Velero backup/restore, etcd restore, application recovery, and RPO/RTO status **belum diverifikasi** without all evidence stages.

## Stop Conditions

- Context/target/backup location ambiguous.
- Backup credential/key boundary unavailable.
- Persistent data/database consistency not defined.
- Restore would overwrite active or production resources.
- RPO/RTO, rollback, or cleanup cannot be measured.
- Raw secret or archive would enter logs/evidence.

## Acceptance Criteria

- [ ] Objects, PV/application data, and etcd are separated.
- [ ] Restore order, consistency, RPO/RTO, retention, and validation are explicit.
- [ ] Evidence captures backup ID/checksum summary without raw artifact.
- [ ] Runtime only disposable and status remains honest.
