# Modul 8.2 — Karakteristik Production On-Prem

> **Fokus:** memahami failure domain dan kontrol operasi ketika tidak ada cloud load balancer, cloud IAM, managed storage, atau managed upgrade.

## Tujuan dan Capaian

Peserta dapat:

- memetakan static IP pool, VLAN, DNS internal, ARP, MTU, dan MetalLB L2/BGP;
- membandingkan Local PV, NFS, SAN, StorageClass, IOPS, capacity, dan backup persistent data;
- menilai host monitoring, node_exporter, disk failure, bastion, SSH hardening, dan access recovery;
- menjelaskan boundary Ansible untuk patch OS dan controlled k3s upgrade;
- merancang Velero object/PV backup, etcd snapshot, restore order, consistency, retention, encryption/key custody, RPO/RTO;
- membuat security review RBAC, NetworkPolicy, secret rotation, image provenance, dan Trivy exception;
- menyusun DR path dari OpenTofu sampai SLO validation.

## Prasyarat

- [Fase 3 — OpenTofu](../../fase-3-opentofu/README.md) untuk metadata dan infrastructure boundary.
- [Fase 4 — Ansible](../../fase-4-ansible/README.md) untuk host configuration, patching, dan k3s.
- [Modul 2.3 — MetalLB](../../fase-2-kubernetes/modul-2.3-metallb/README.md) dan [Modul 2.4](../../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md).
- [Fase 5 — Helm](../../fase-5-helm/README.md) dan [Fase 7 — Observability](../../fase-7-observability/README.md).

## Rencana Belajar

| Sesi | Materi | Output |
|---|---|---|
| 1 | [Network, storage, hardware, access](01-network-storage-hardware-access.md) | Topology/dependency map dan capacity review |
| 2 | [Backup, upgrade, security, DR](02-backup-restore-upgrade-security-dr.md) | Backup/restore, patching, upgrade, dan DR plan |
| 3 | [LAB-01](lab/LAB-01-velero-etcd-backup-restore.md) | Restore procedure dan RPO/RTO evidence template |
| 4 | [LAB-02](lab/LAB-02-onprem-readiness-patching-trivy.md) | Readiness, patch, k3s, dan Trivy review |
| 5 | [Latihan dan kuis](evaluasi/latihan.md) | Minimal 80%, tanpa guardrail violation |

## Acceptance Criteria

- [ ] Network, storage, host, access, dan dependency failure domain terdokumentasi.
- [ ] Tidak ada cloud LB/IAM assumption yang tersembunyi.
- [ ] Backup membedakan objects, PV/application data, dan etcd; consistency/key custody dijelaskan.
- [ ] Restore memiliki order, RPO, RTO, retention, validation, dan evidence redaction.
- [ ] Patching/upgrades memiliki serial scope, maintenance window, quorum/PDB/capacity review, dan recovery.
- [ ] RBAC, NetworkPolicy, secret rotation, Trivy provenance, exception, dan remediation owner tersedia.
- [ ] DR path dan observability/SLO validation didefinisikan.

## Dua Lane

Static lane cukup untuk topology, diagrams, policy, commands placeholder, plan review, dan evidence schema. Runtime lane hanya target disposable dengan tool/context/target/storage/access preflight, approval, timeout, serial strategy, rollback, cleanup, dan evidence redacted.

Velero `Completed`, etcd snapshot tersedia, Trivy exit code, Ansible `changed=0`, atau node `Ready` bukan bukti aplikasi pulih atau production ready.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan mencetak backup credential, encryption key, kubeconfig, k3s token, raw archive, raw inventory secret, rendered Secret, SSH key, atau object-storage key. Jangan menjalankan restore pada cluster aktif, `cluster-reset`, `kubectl delete -A`, drain/upgrade tanpa quorum/PDB review, atau patch production tanpa approval dan recovery.

## Troubleshooting

- **MetalLB IP tidak reachable:** periksa VLAN, ARP, speaker, pool, MTU, switch policy, dan duplicate IP; jangan langsung mengubah pool.
- **Restore berhasil tetapi aplikasi gagal:** periksa PV/database consistency, secret references, DNS, dependency, migration, dan SLO.
- **Upgrade kehilangan capacity:** stop serial rollout, cek PDB/quorum/headroom, dan gunakan recovery plan.
- **Trivy menemukan vulnerability:** tetapkan severity, exploitability, owner, due date, upgrade/remediation, dan exception expiry.

## Kaitan

- [Fase 4 — Ansible](../../fase-4-ansible/README.md) adalah jalur perubahan OS yang diharapkan.
- [Modul 2.4 — Operasi](../../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md) menyediakan etcd, drain, PDB, dan rolling upgrade.
- [Fase 6 — GitOps](../../fase-6-gitops/README.md) melakukan rehydration desired state.
- [Fase 7 — Observability](../../fase-7-observability/README.md) memvalidasi telemetry dan SLO setelah recovery.
- [Fase 9 — Capstone](../../fase-9-capstone/README.md) menguji rebuild end-to-end dan recovery evidence.

## Status Runtime

Velero backup/restore, etcd snapshot/restore, OS patch, k3s upgrade, host monitoring, Trivy runtime, dan DR recovery: **belum diverifikasi** tanpa execution evidence.
