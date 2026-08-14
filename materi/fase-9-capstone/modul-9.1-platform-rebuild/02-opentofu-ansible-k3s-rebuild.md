# 9.1.2 — Contract OpenTofu → Ansible → k3s

## Prinsip Handoff

Handoff adalah data contract, bukan salin-tempel output terminal. OpenTofu menghasilkan metadata non-secret; Ansible mengonsumsi inventory dan melakukan bootstrap; k3s mengembalikan readiness summary; delivery layer baru berjalan setelah handoff disetujui.

```text
OpenTofu plan/apply pada disposable scope
→ metadata VM/network/storage non-secret
→ Ansible inventory + host readiness
→ k3s server/agent bootstrap dan quorum check
→ storage/MetalLB/access validation
→ handoff approval ke Modul 9.2
```

## Contoh Handoff Non-Secret

```yaml
capstone_revision: <git-revision>
target:
  environment: disposable-production-simulation
  workspace: <approved-workspace>
  allowlist: [<vm-id-1>, <vm-id-2>]
nodes:
  - name: <control-plane-1>
    role: server
    address: <redacted-or-approved-address>
network:
  node_cidr: <approved-cidr>
  service_cidr: <approved-service-cidr>
  pod_cidr: <approved-pod-cidr>
  metallb_pool: <dedicated-approved-range>
access:
  bastion_alias: <approved-alias>
  recovery_reference: <access-runbook-id>
readiness:
  quorum_required: <n>
  storage_class: <approved-class>
  capacity_headroom_percent: <n>
```

File tersebut tidak boleh berisi token, kubeconfig, private key, password, registry credential, atau object-storage key. Secret hanya direferensikan melalui identifier secret manager yang telah disetujui.

## Preflight

Sebelum plan atau bootstrap, verifikasi read-only:

- repository revision dan working tree;
- tool version dan platform ARM64;
- workspace/state backend dan lock boundary;
- target allowlist, ownership, maintenance window, dan backup;
- access recovery/bastion;
- resource budget OrbStack;
- network range, DNS, VLAN/MTU, storage, dan MetalLB pool;
- quorum, PDB, capacity, dan rollback/recovery.

Jika context kosong, target ambigu, lock tidak sehat, atau access recovery hilang, berhenti. Jangan mengganti preflight yang gagal dengan asumsi.

## Plan, Approval, dan Action

Gunakan plan/diff yang disimpan sementara pada lokasi terlindungi dan jangan commit raw plan. Review mencakup create/update/destroy, dependency, blast radius, dan external side effect. `--check` atau `--diff` membantu review tetapi bukan jaminan zero side effect.

OpenTofu apply, destroy, import, state move/remove, Ansible patch, k3s install, dan network change membutuhkan approval eksplisit. Jangan memakai `-auto-approve` sebagai default. Ansible selalu dibatasi dengan `--limit` dan serial strategy yang sesuai.

## Readiness Handoff

Readiness harus mencakup:

1. OS, time sync, package, disk, dan access.
2. k3s control-plane quorum serta API/etcd health.
3. worker scheduling, CNI/DNS, taint/label, dan capacity.
4. StorageClass/PV behavior dan backup boundary.
5. MetalLB speaker/controller, dedicated pool, ARP/L2, dan Ingress traffic.
6. Observability prerequisite dan access recovery.

Node `Ready` hanya signal lokal. Handoff baru dapat diajukan bila seluruh dependency memiliki status dan evidence yang dapat diaudit.

## Destroy dan Recovery Contract

Destroy hanya untuk disposable target yang telah diverifikasi. Sebelum destroy:

```text
confirm target allowlist
→ capture redacted before-state summary
→ verify backup and restore path
→ review destroy plan/diff
→ approval + maintenance window
→ execute scoped destroy
→ verify cleanup
→ rebuild dari repository
→ compare desired/as-built
```

Jangan menjalankan destroy pada production atau target ambigu. Jangan melakukan cluster reset aktif atau restore snapshot langsung pada cluster aktif. Bila recovery gagal, stop dan eskalasi sesuai runbook.

## Evidence Record

```yaml
revision: <git-revision>
target: <approved-disposable-target>
window_utc: <start-end>
approval_ref: <change-or-approval-id>
plan_summary: <counts-only-redacted>
action_summary: <status-and-identifiers>
readiness_summary: <passed-failed-unknown-by-domain>
recovery_summary: <redacted>
next_owner: <role>
```

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

K3s token dan kubeconfig hanya boleh digunakan melalui access mechanism yang disetujui, tidak dicetak atau ditulis ke Git. `sensitive = true` bukan enkripsi state.

## Kaitan

- [LAB-01](lab/LAB-01-infrastructure-rebuild.md) mempraktikkan rebuild contract.
- [LAB-02](lab/LAB-02-k3s-bootstrap-handoff.md) memvalidasi handoff.
- Delivery baru dimulai pada [Modul 9.2](../modul-9.2-delivery-observability/README.md).
