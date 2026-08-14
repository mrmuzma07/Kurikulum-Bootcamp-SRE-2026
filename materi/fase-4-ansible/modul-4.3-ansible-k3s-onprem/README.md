# Modul 4.3 — Ansible untuk k3s dan On-Prem

> **Tujuan:** mengotomasi host readiness, hardening, instalasi/upgrade k3s, dan rolling maintenance dengan boundary serta evidence yang aman.

## Capaian

- [ ] Mengubah metadata OpenTofu menjadi inventory Ansible non-secret.
- [ ] Membuat readiness gate untuk identity, network, DNS, time, SSH, OS, storage, firewall, dan security.
- [ ] Menjelaskan role k3s, server/agent join, version pin, quorum, token boundary, dan health check.
- [ ] Menyusun hardening common dengan blast radius dan rollback yang jelas.
- [ ] Merencanakan rolling patching/upgrade satu node per satu dengan `serial: 1`.
- [ ] Menyusun backup, rebuild, evidence, dan stop condition.

## Rencana 2 Hari

| Hari | Materi | Lab |
|---|---|---|
| 1 | [Handoff & Readiness](01-opentofu-inventory-readiness.md), [k3s Role](02-k3s-install-upgrade-role.md) | [LAB-01](lab/LAB-01-handoff-k3s-multinode.md) |
| 2 | [Hardening & Rolling](03-hardening-patching-rolling.md), [Backup & Evidence](04-backup-health-evidence-rebuild.md) | [LAB-02](lab/LAB-02-rolling-patching-readiness.md) + evaluasi |

## Prasyarat dan Lane

Selesaikan Modul 4.1–4.2, Modul 3.3, Modul 2.2, dan Modul 2.4. Static lane adalah default bila environment tidak tersedia. Runtime lane membutuhkan VM/cluster disposable, approval, access path, dan maintenance window.

## Deliverables

Inventory handoff, readiness matrix, role/runbook k3s, hardening plan, rolling patch runbook, health/evidence template, dan rebuild procedure.

## Boundary

OpenTofu memiliki VM/network/storage; Ansible memiliki host configuration; k3s memiliki cluster lifecycle. Credential, token, dan kubeconfig tidak menjadi output OpenTofu atau committed inventory.

## Guardrail

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menjalankan install/upgrade, firewall, SSH mutation, drain, restore, atau reset pada cluster aktif tanpa runbook dan approval.
- `kubectl delete -A` dan `k3s server --cluster-reset` dilarang sebagai shortcut.
- Backup etcd bukan backup aplikasi/PV/database.
- `changed=0` bukan bukti cluster sehat; lakukan health check context-safe.

## Acceptance Criteria

- [ ] Semua host memiliki stable identity dan readiness evidence sebelum mutation.
- [ ] Inventory hanya memuat metadata non-secret.
- [ ] k3s topology, quorum, network, version, backup, upgrade, dan rollback direview.
- [ ] Rolling task memiliki `serial`, preflight, post-check, dan stop condition.
- [ ] Runtime yang belum dijalankan ditandai belum diverifikasi.

## Status Runtime dan Kaitan

Tidak ada klaim SSH, Ansible, k3s, backup, atau hardening tanpa evidence. Setelah modul ini, materi berlanjut ke Fase 5 Helm yang masih menyusul.

## Catatan SRE

Automasi cluster harus mengutamakan availability dan recoverability. Satu node yang gagal bukan alasan untuk memaksa playbook lanjut; berhenti, pertahankan evidence, dan evaluasi quorum serta jalur pemulihan.
