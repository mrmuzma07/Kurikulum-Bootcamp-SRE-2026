# Fase 8 — Praktik SRE & Kesiapan Production On-Prem

> **Tujuan fase:** mengubah telemetry dan SLO dari [Fase 7](../fase-7-observability/README.md) menjadi kebijakan reliability, operasi on-call, incident command, change safety, backup/restore, runbook, dan keputusan production-readiness untuk environment on-prem.

## Durasi dan Modul

Minggu 15 — tiga modul dengan static review dan optional disposable runtime lane.

| Modul | Fokus | Status |
|---|---|---|
| 8.1 | Praktik SRE | ✅ Tersedia |
| 8.2 | Karakteristik Production On-Prem | ✅ Tersedia |
| 8.3 | Runbook & Dokumentasi | ✅ Tersedia |

## Boundary Ownership

```text
OpenTofu → VM/network/storage metadata
Ansible → OS/bootstrap/readiness/patching/k3s host configuration
Kubernetes/k3s → workload, scheduling, service, storage, cluster health
Helm → chart packaging and release rendering
GitLab CI/GitOps/ArgoCD → validation, desired state, promotion, reconciliation
Fase 7 Observability → metrics, logs, traces, dashboard, alert, SLO evidence
Fase 8 SRE → reliability policy, on-call, incident command, mitigation,
             backup/restore, change safety, runbook, postmortem, readiness decision
```

Fase 8 tidak menggantikan controller atau tooling fase sebelumnya. Ia menentukan kapan evidence dari layer tersebut cukup untuk mengambil keputusan, siapa owner-nya, dan bagaimana perubahan atau incident dikendalikan.

## Capaian Fase

- [ ] Menulis SLI/SLO dengan numerator, denominator, eligible traffic, exclusion, missing-data policy, owner, dan review cadence.
- [ ] Menghitung error budget dan burn rate serta menghubungkannya ke dashboard/alert Fase 7 dan promotion gate Fase 6.
- [ ] Mengidentifikasi toil, memilih automation yang aman, dan mengendalikan blast radius automation.
- [ ] Mendesain on-call primary/secondary, handoff, escalation, severity, acknowledgement, paging boundary, fatigue control, dan access recovery.
- [ ] Menjalankan model incident response: detection → declaration → triage → mitigation/rollback → resolution → monitoring → closure → blameless postmortem.
- [ ] Menilai change classification, risk/impact, peer review, maintenance window, change freeze, CAB ringan, dan emergency change review.
- [ ] Menjelaskan dependency on-prem: static IP, VLAN, DNS, ARP, MTU, MetalLB, Local PV/NFS/SAN, bastion, host monitoring, dan capacity.
- [ ] Merancang backup/restore, RPO/RTO, retention, encryption/key custody, restore order, dan disaster recovery path.
- [ ] Merencanakan OS patch melalui Ansible dan controlled k3s upgrade dengan serial strategy, quorum/PDB review, serta recovery.
- [ ] Menulis runbook operasional dan postmortem redacted yang dapat diuji serta memiliki evidence fields.
- [ ] Membuat keputusan readiness berdasarkan evidence chain, bukan status tunggal seperti `Healthy`, `Completed`, `Firing`, exit code, atau `changed=0`.

## Rencana Belajar

| Hari | Materi | Praktik |
|---|---|---|
| 1–2 | [Modul 8.1](modul-8.1-praktik-sre/README.md) | [LAB-01](modul-8.1-praktik-sre/lab/LAB-01-slo-error-budget-oncall.md) + [LAB-02](modul-8.1-praktik-sre/lab/LAB-02-incident-response-change-freeze.md) |
| 3–4 | [Modul 8.2](modul-8.2-production-onprem/README.md) | [LAB-01](modul-8.2-production-onprem/lab/LAB-01-velero-etcd-backup-restore.md) + [LAB-02](modul-8.2-production-onprem/lab/LAB-02-onprem-readiness-patching-trivy.md) |
| 5 | [Modul 8.3](modul-8.3-runbook-dokumentasi/README.md) | [LAB-01](modul-8.3-runbook-dokumentasi/lab/LAB-01-runbook-validation-drill.md) + [LAB-02](modul-8.3-runbook-dokumentasi/lab/LAB-02-postmortem-blameless.md) |

## Dua Lane Praktik

### Static lane

Static lane digunakan untuk review SLO contract, error-budget math, toil inventory, on-call matrix, escalation tree, incident template, change record, CAB/freeze policy, topology, backup/restore plan, RTO/RPO, patching strategy, k3s upgrade plan, Trivy policy, runbook, dan postmortem. Static success hanya membuktikan design/documentation review.

### Disposable runtime lane

Runtime hanya boleh dilakukan pada target disposable yang diverifikasi, bukan production:

```text
verify tools/context/namespace/target/storage/access recovery
→ review plan/diff + scope + approval + maintenance window
→ scoped action or bounded fault injection
→ capture before/after health, alert/timeline, backup ID/checksum summary
→ rollback/restore/cleanup
→ post-check SLO/error-budget outcome
→ redact and retain evidence
```

Tanpa evidence execution aktual, Velero backup/restore, etcd snapshot/restore, Trivy runtime scan, patching, k3s upgrade, incident drill, notification, dan production-readiness tetap **belum diverifikasi**.

## Evidence Chain

```text
policy/runbook revision
→ preflight/scope/approval
→ action or injected fault
→ telemetry/alert/paging
→ incident timeline/decision
→ mitigation/rollback/recovery
→ post-check/SLO/error-budget outcome
→ postmortem/action-item verification
```

Dashboard green, alert firing, `kubectl` Healthy, Velero `Completed`, snapshot tersedia, Trivy exit code, dan Ansible `changed=0` hanya membuktikan tahap lokal. Semuanya harus dikaitkan dengan chain dan outcome yang relevan.

## Guardrail Fase

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menyimpan password, k3s token, SSH private key, registry/object-storage credential, webhook, raw Secret, raw state/plan, raw backup, raw CI log, atau PII dalam materi, log, evidence, shell history, atau artifact.
- `sensitive = true`, `.gitignore`, ciphertext SOPS, atau secret reference tidak menggantikan encryption, key custody, rotation, backup, dan recovery.
- Jangan memakai `kubectl delete -A`, cluster reset, restore snapshot pada cluster aktif, atau chaos/failure injection pada production.
- Apply, destroy, patch, drain, upgrade, restore, Helm lifecycle, firewall/SSH changes, dan notification membutuhkan scope, approval, access recovery, stop condition, rollback/recovery, serta post-check.
- Manual SSH hanya break-glass dengan approval, audit, dan follow-up automation melalui Ansible.
- `--check`, `--diff`, `--atomic`, status Helm/Argo, pipeline green, dan alert firing bukan jaminan zero side effect, health, recovery, atau SLO.
- Evidence harus redacted dan menyimpan summary/identifier, bukan credential atau raw artifact.

## Kaitan

- [Modul 2.4 — Operasi Kubernetes](../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md) menyediakan context safety, etcd, PDB, drain, upgrade, dan incident evidence.
- [Fase 4 — Ansible](../fase-4-ansible/README.md) menyediakan perubahan OS, patching, readiness, dan k3s host configuration.
- [Fase 5 — Helm](../fase-5-helm/README.md) menyediakan chart lifecycle dan rollback caveat.
- [Fase 6 — GitOps](../fase-6-gitops/README.md) menyediakan promotion evidence dan desired-state reconciliation.
- [Fase 7 — Observability](../fase-7-observability/README.md) menyediakan telemetry, alert, dashboard, dan SLO signal.
- [Fase 9 — Capstone](../fase-9-capstone/README.md) menggabungkan fondasi ini dalam rebuild, delivery evidence, recovery, dan Game Day.

## Deliverables

1. SLO dan dashboard/error-budget design untuk aplikasi sendiri.
2. On-call matrix, escalation policy, incident template, change record, freeze/CAB policy.
3. Backup/restore plan Velero + etcd, RPO/RTO, DR path, dan evidence template.
4. Trivy CI policy, Ansible patching plan, controlled k3s upgrade plan, serta production on-prem readiness checklist.
5. Dua runbook operasional, topology/dependency map, dan satu postmortem blameless.
6. Evidence chain redacted yang menghubungkan policy sampai verification action item.
7. Nilai kuis minimal **80%** setiap modul tanpa pelanggaran guardrail.

## Status Runtime

Materi Fase8: **tersedia**. Velero backup/restore, etcd snapshot/restore, Trivy scan runtime, OS patching, k3s upgrade, incident drill, notification, dan production-readiness decision: **belum diverifikasi** tanpa execution evidence yang lengkap dan redacted.
