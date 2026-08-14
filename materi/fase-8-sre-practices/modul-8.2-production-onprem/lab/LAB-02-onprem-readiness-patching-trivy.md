# LAB-02 — On-Prem Readiness, Patching, k3s Upgrade, dan Trivy

## Tujuan

Membuat readiness review untuk network/storage/host/access/security, lalu menyusun execution gate untuk Ansible patch, controlled k3s upgrade, dan Trivy CI scan.

## Prasyarat dan Guardrail

Gunakan [network/storage/access](../01-network-storage-hardware-access.md), [backup/upgrade/security/DR](../02-backup-restore-upgrade-security-dr.md), [Ansible](../../../fase-4-ansible/README.md), dan [GitLab CI](../../../fase-6-gitops/README.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan mencetak inventory secret, SSH private key, registry credential, CI token, raw Trivy report containing sensitive paths, atau kubeconfig. `--check`, `--diff`, dan Trivy exit code bukan full readiness evidence.

## Lane A — Static Review

### 1. Readiness matrix

| Area | Check | Owner | Evidence | Stop condition |
|---|---|---|---|---|
| network | VLAN/DNS/ARP/MTU/MetalLB | `<team>` | `<summary>` | duplicate/unknown IP |
| storage | capacity/IOPS/PV/backup | `<team>` | `<summary>` | no restore path |
| host | CPU/memory/disk/node_exporter | `<team>` | `<summary>` | no headroom |
| access | bastion/SSH/recovery | `<team>` | `<summary>` | no break-glass |
| cluster | quorum/PDB/version | `<team>` | `<summary>` | unsafe drain |
| security | RBAC/NetworkPolicy/Trivy | `<team>` | `<summary>` | critical unowned |

### 2. Ansible patch plan

```text
inventory: <approved-inventory-reference>
limit: <approved-host-allowlist>
serial: <approved-batch-size>
maintenance_window: <utc-window>
precheck: <health-and-backup-reference>
change: <patch-scope>
rollback/recovery: <approved-reference>
postcheck: <host-k3s-workload-telemetry-check>
approval: <approved-role>
```

`--limit` membatasi blast radius. Review diff, package impact, reboot, quorum, PDB, capacity, and access recovery. Manual SSH hanya break-glass.

### 3. k3s upgrade plan

Tentukan release compatibility, backup, node order, cordon/drain, serial one-node-at-a-time, quorum/PDB, capacity, rollback/recovery, and post-check. Jangan mengklaim upgrade berhasil dari package version saja.

### 4. Trivy policy

| Severity | Action | Owner | Exception expiry |
|---|---|---|---|
| `<approved-severity>` | block/remediate/review | `<team>` | `<utc-date>` |

Tambahkan image digest/provenance, scanner version, database freshness summary, SBOM reference, remediation ticket, and compensating control. Jangan menyimpan raw report jika berisi path/credential/PII.

## Lane B — Optional Disposable Runtime

1. Verify target host/cluster/namespace, inventory allowlist, context, storage, backup, access recovery, and maintenance window.
2. Review Ansible plan/diff, k3s release, Trivy scope/image digest, approval, and rollback.
3. Run Trivy against approved image in CI or disposable environment; capture summary/severity counts, digest, scanner version, and decision.
4. Run Ansible patch with `--limit` and serial strategy; capture before/after health summary.
5. Upgrade one disposable k3s node at a time; stop on quorum/PDB/capacity or health failure.
6. Verify workloads, storage, network, observability, alerts, and SLO window.
7. Cleanup or recover target; redact logs/evidence.

Patching, k3s upgrade, Trivy runtime, and on-prem readiness **belum diverifikasi** without full chain.

## Stop Conditions

- Production host or broad inventory selected.
- No backup/restore/access recovery or maintenance window.
- Quorum/PDB/headroom unsafe.
- Critical vulnerability has no owner or exception.
- SSH/manual change would be the only desired-state path.

## Acceptance Criteria

- [ ] Readiness matrix memiliki owner, evidence, dan stop condition.
- [ ] Ansible patch memakai bounded `--limit`, serial, approval, dan recovery.
- [ ] k3s upgrade memiliki quorum/PDB/capacity review.
- [ ] Trivy policy memiliki digest/provenance, severity, owner, remediation, dan exception expiry.
- [ ] Runtime claim tetap **belum diverifikasi** bila evidence aktual belum lengkap.
