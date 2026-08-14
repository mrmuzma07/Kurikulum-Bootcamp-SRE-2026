# 9.1.1 — Arsitektur Capstone dan Boundary

## Tujuan

Capstone dimulai dari kontrak arsitektur, bukan dari perintah destroy. Peserta harus dapat menjelaskan resource mana yang dimiliki setiap layer, bagaimana akses dipulihkan, dan evidence apa yang membuktikan topology yang dibangun benar-benar sesuai desired state.

## Topologi Target

```text
MacBook Apple Silicon
└── OrbStack
    ├── infra VM
    │   ├── state/lock object storage lokal
    │   ├── GitLab runner lokal
    │   └── Ansible control node
    ├── servers VM
    │   └── k3s production multi-node
    └── k3d staging
```

Production workload memiliki jalur:

```text
Git commit → GitLab CI → immutable image digest → reviewed GitOps commit
→ ArgoCD reconciliation → Helm release → MetalLB L2 → Ingress
→ Prometheus/Alloy → Mimir/Loki/Tempo → Grafana/Alertmanager
```

Diagram tersebut adalah desired state. As-built state harus dicatat dari output yang sudah disaring, bukan disalin sebagai raw dump.

## Boundary Ownership

| Layer | Memiliki | Tidak boleh mengambil alih |
|---|---|---|
| OpenTofu | VM, network, metadata, storage dan disposable lifecycle | package OS, workload, secret value |
| Ansible | OS, SSH/bootstrap, readiness, patching, k3s host config | desired application release |
| k3s/Kubernetes | scheduling, Service, storage, cluster health | VM provisioning dan Git promotion |
| Helm | chart, values non-secret, rendering | production approval dan incident decision |
| GitLab CI/GitOps/ArgoCD | validation, artifact provenance, desired state, reconciliation | manual credential injection atau bypass review |
| MetalLB/Ingress | service exposure dan traffic boundary | SLO decision |
| Observability | signal, dashboard, alert, evidence | menggantikan incident command |
| SRE/Capstone | policy, on-call, recovery, Game Day, graduation | mengklaim runtime tanpa evidence |

## Design Review Minimum

Review harus menjawab:

1. Apakah VM `infra`, `servers`, dan k3d staging berada pada scope yang berbeda?
2. Di mana state dan lock disimpan, siapa owner-nya, bagaimana backup dan recovery-nya?
3. Bagaimana inventory non-secret berpindah dari OpenTofu ke Ansible?
4. Bagaimana API k3s, bastion, DNS, VLAN, MTU, dan access recovery bekerja?
5. Apakah quorum etcd, PDB, capacity headroom, storage consistency, dan failure domain cukup untuk skenario yang dipilih?
6. Apakah dedicated MetalLB pool tidak tumpang tindih dengan DHCP, node IP, Service CIDR, atau Pod CIDR?
7. Bagaimana desired-state drift ditemukan tanpa manual edit production?

## Desired, As-Built, dan Evidence Topology

- **Desired topology:** kontrak repository yang direview; berisi role, range, dependency, ownership, dan policy.
- **As-built topology:** hasil actual build; berisi identifier dan status summary yang telah direduksi.
- **Evidence topology:** paket bukti yang menghubungkan revision, target, waktu UTC, action, health, dan outcome.

IP, hostname, atau identifier yang memang diperlukan untuk diagram boleh dicatat bila bukan credential. Jangan mencatat kubeconfig, token, private key, password, atau endpoint yang mengandung secret.

## Dependency dan Recovery Boundary

Contoh urutan dependency:

```text
OrbStack/resource budget
→ VM/network/storage metadata
→ OS/SSH readiness
→ k3s control-plane quorum
→ worker/storage/MetalLB
→ Helm/GitOps/ArgoCD
→ application/telemetry
→ backup/restore validation
```

Setiap dependency memiliki owner, precondition, timeout, stop condition, dan recovery path. Access recovery harus tersedia sebelum action yang dapat memutus akses; break-glass access dicatat, dibatasi, dan ditindaklanjuti melalui desired state.

## Static dan Runtime Lane

Static review dapat membuktikan diagram, contract, dan dependency logic. Runtime rebuild hanya boleh pada target disposable yang diverifikasi, dengan target allowlist, plan/diff, approval, backup, recovery path, dan cleanup. `kubectl get nodes`, `Ansible changed=0`, atau OpenTofu plan yang sukses tidak sendirian membuktikan readiness.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

`Sensitive` pada output hanya menyamarkan tampilan dan tidak mengenkripsi state. Jangan menyimpan raw state, raw plan, raw inventory secret, atau rendered Secret di repository maupun evidence.

## Latihan Design Review

Buat satu diagram desired state dan satu tabel boundary. Untuk setiap dependency, tambahkan owner, precondition, failure signal, stop condition, dan evidence reference. Tandai bagian yang masih `belum diverifikasi`.

## Kaitan

- Implementasi contract ada di [9.1.2](02-opentofu-ansible-k3s-rebuild.md).
- Praktik rebuild ada di [LAB-01](lab/LAB-01-infrastructure-rebuild.md) dan handoff di [LAB-02](lab/LAB-02-k3s-bootstrap-handoff.md).
- Boundary on-prem dilanjutkan dari [Modul 8.2](../../fase-8-sre-practices/modul-8.2-production-onprem/README.md).
