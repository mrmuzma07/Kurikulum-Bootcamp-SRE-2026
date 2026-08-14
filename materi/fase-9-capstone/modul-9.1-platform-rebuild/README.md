# Modul 9.1 — Platform Rebuild

> **Fokus:** membangun ulang platform disposable dari metadata OpenTofu, bootstrap Ansible, dan handoff k3s tanpa kehilangan ownership atau access recovery.

## Tujuan dan Capaian

Peserta dapat:

- membedakan desired-state topology, as-built topology, dan evidence topology;
- menetapkan ownership OpenTofu, Ansible, k3s, storage, network, dan access recovery;
- merancang state/lock, inventory handoff, static IP, DNS, VLAN, MetalLB pool, dan dependency map;
- menjalankan preflight, plan/diff review, approval, serial bootstrap, readiness, rollback, dan cleanup;
- menjelaskan quorum, PDB, capacity headroom, bastion, dan recovery path sebelum rebuild.

## Prasyarat

- [Fase 3 — OpenTofu](../../fase-3-opentofu/README.md).
- [Fase 4 — Ansible](../../fase-4-ansible/README.md).
- [Modul 2.2 — k3s Production](../../fase-2-kubernetes/modul-2.2-k3s-production/README.md).
- [Modul 2.3 — MetalLB](../../fase-2-kubernetes/modul-2.3-metallb/README.md).
- [Fase 8 — Production On-Prem](../../fase-8-sre-practices/modul-8.2-production-onprem/README.md).

## Rencana Sesi

| Sesi | Materi | Output |
|---|---|---|
| 1 | [Arsitektur dan boundary](01-capstone-arsitektur-boundary.md) | desired/as-built topology dan dependency map |
| 2 | [OpenTofu, Ansible, k3s rebuild](02-opentofu-ansible-k3s-rebuild.md) | rebuild contract dan handoff schema |
| 3 | [LAB-01](lab/LAB-01-infrastructure-rebuild.md) | preflight, plan, rebuild, cleanup evidence |
| 4 | [LAB-02](lab/LAB-02-k3s-bootstrap-handoff.md) | readiness dan access recovery review |
| 5 | [Latihan dan kuis](evaluasi/latihan.md) | minimal 80%, tanpa guardrail violation |

## Deliverables

1. Arsitektur target dan as-built evidence template.
2. OpenTofu/Ansible/k3s ownership matrix.
3. Target allowlist, plan/diff approval, maintenance window, dan recovery contract.
4. Inventory handoff non-secret dan dependency/access map.
5. Readiness report yang tidak menyamakan `Ready` atau `changed=0` dengan production readiness.

## Acceptance Criteria

- [ ] Scope rebuild hanya disposable dan target allowlist terdokumentasi.
- [ ] State/lock, backup, recovery, dan access boundary jelas.
- [ ] OpenTofu menghasilkan metadata yang dapat dikonsumsi Ansible tanpa secret.
- [ ] Ansible menjalankan bootstrap secara serial/scoped dengan rollback atau recovery.
- [ ] k3s readiness mencakup quorum, CNI/DNS, storage, MetalLB dependency, capacity, dan access recovery.
- [ ] Before/after evidence memiliki revision, UTC timestamp, target, dan redaction.
- [ ] Runtime tetap **belum diverifikasi** sampai execution evidence tersedia.

## Dua Lane

Static lane meninjau topology, contracts, inventory schema, dependency graph, plan example, dan readiness rubric. Disposable runtime lane harus melalui preflight, approval, bounded action, post-check, cleanup, dan evidence redaction.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan mencetak raw state/plan, k3s token, kubeconfig, SSH private key, password, registry credential, atau inventory secret. Jangan memakai `-auto-approve` sebagai default, destroy target tanpa allowlist, atau `k3s server --cluster-reset` pada cluster aktif.

## Troubleshooting

- **Plan menyentuh target tak dikenal:** berhenti; cocokkan workspace, target allowlist, owner, dan approval.
- **Ansible host tidak reachable:** lakukan read-only connectivity/access recovery check; jangan mem-bypass host key atau menaruh key di repo.
- **Node belum Ready:** pisahkan OS, network, kubelet/k3s, CNI/DNS, storage, dan capacity evidence.
- **Kubeconfig tidak dapat dipakai:** gunakan jalur recovery yang disetujui; jangan menyalin atau mencetak isinya ke evidence.
- **Rebuild kehilangan state:** stop, verifikasi backup/lock/recovery order; jangan mengarang resource dari memory.

## Kaitan

- [Modul 9.2](../modul-9.2-delivery-observability/README.md) memakai cluster hasil handoff untuk delivery dan recovery evidence.
- [Modul 9.3](../modul-9.3-game-day-graduation/README.md) memakai topology dan access boundary untuk Game Day.

## Status Runtime

VM rebuild, Ansible bootstrap, k3s readiness, storage, MetalLB, dan access recovery: **belum diverifikasi** tanpa evidence runtime yang lengkap.
