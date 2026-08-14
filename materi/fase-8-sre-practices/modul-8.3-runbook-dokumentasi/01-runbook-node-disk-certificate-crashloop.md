# 01 — Runbook Node, Disk, Certificate, dan CrashLoopBackOff

## Runbook Contract

Semua contoh di bawah adalah template. Ganti placeholder setelah owner dan approval ditetapkan; jangan menambahkan credential atau raw payload.

```text
runbook_id: <approved-runbook-id>
revision: <git-revision>
last-reviewed: <utc-timestamp>
expires: <utc-timestamp>
owner/escalation: <approved-team>/<approved-escalation>
```

## 1. Node Down

- **Symptom:** node exporter/heartbeat unavailable atau node `NotReady`.
- **Scope/severity:** `<approved-cluster>/<approved-severity>` berdasarkan workload impact.
- **Preconditions:** context, cluster, namespace, incident ID, maintenance status, and access recovery verified.
- **Read-only first checks:** node conditions, events summary, workload placement, recent changes, host telemetry, network reachability melalui approved path.
- **Decision tree:**
  1. Jika telemetry collector saja down, eskalasi observability dan jangan drain.
  2. Jika host unreachable, cek power/network/bastion dan capacity failover.
  3. Jika workload impact tinggi, IC memilih mitigation bounded dan owner host.
- **Stop condition:** target bukan disposable/approved, quorum/PDB tidak aman, identity/context ambigu, atau recovery belum jelas.
- **Safe mitigation:** cordon/drain hanya setelah PDB, replica, capacity, maintenance window, dan approval diperiksa.
- **Rollback/recovery:** uncordon hanya setelah health; host rebuild/rejoin via Ansible/k3s runbook. Jangan cluster-reset spontan.
- **Data caveat:** Local PV mungkin tidak pindah; validasi application/database consistency.
- **Communication/post-check:** update `<redacted-status-channel>`; cek workload, storage, telemetry, alert, dan SLO window.
- **Evidence:** revision, node summary, timestamps UTC, decision, action summary, before/after health, recovery ID.

## 2. Disk Full

- **Symptom:** filesystem/inode threshold, pod eviction, write errors, atau log backlog.
- **Read-only checks:** filesystem/inode summary, top-level usage tanpa mencetak secret, container/log retention, PV usage, node events, and affected workloads.
- **Decision tree:** bedakan host filesystem, container layer, log, PV, database, atau backup staging.
- **Stop condition:** path berisi database/PV tanpa recovery plan; jangan `rm -rf` atau menghapus data berdasarkan ukuran saja.
- **Safe mitigation:** pause noisy workload/log source, rotate melalui policy, expand approved storage, atau move workload setelah capacity review.
- **Rollback/data caveat:** deletion/compaction dapat irreversible; backup dan application consistency harus diverifikasi.
- **Post-check:** inode/space, write path, workload readiness, error rate, alert recovery, dan SLO.

## 3. Certificate Expired

- **Symptom:** TLS handshake/authentication failures atau expiry alert.
- **Read-only checks:** certificate subject/issuer/expiry summary, affected endpoint, clock/NTP, trust chain, recent rotation, and dependency.
- **Stop condition:** private key tidak tersedia melalui approved secret mechanism, issuer tidak dikenal, atau renewal scope ambiguous.
- **Safe mitigation:** renew melalui approved automation/secret manager; jangan menempelkan private key ke shell, README, atau evidence.
- **Rollback/data caveat:** certificate rollback tidak boleh mengembalikan expired trust; perhatikan clients/cache/propagation.
- **Post-check:** endpoint handshake, workload, logs redacted, alert resolution, and representative SLO window.

## 4. CrashLoopBackOff

- **Symptom:** pod repeatedly restarts atau readiness tidak pernah tercapai.
- **Read-only checks:** workload revision, pod status/events summary, container exit reason, probes, config references, resource/quota, dependency health, and recent GitOps change.
- **Decision tree:** image/config/probe/resource/dependency/data migration. Jangan langsung menghapus pod sebagai diagnosis.
- **Stop condition:** secret value/raw environment, production target tanpa incident approval, migration/data risk, atau rollback tidak kompatibel.
- **Safe mitigation:** pause promotion, scale/traffic boundary secara bounded, revert reviewed GitOps revision, atau apply approved config fix.
- **Rollback/data caveat:** Git revert tidak membalikkan migration/PVC/external side effect.
- **Post-check:** stable restarts, readiness, logs summary, error/latency, alert, and SLO.

## Evidence dan Metadata

Simpan hanya summary redacted: incident/runbook revision, query reference, UTC timestamps, status, action/decision, owner, and post-check. Runbook expired atau belum direview harus mengarah ke stop condition, bukan action improvisasi.
