# LAB-02 — Handoff ke Ansible dan k3s

> **Mode utama:** desain dan static review. **Tidak ada klaim Ansible atau k3s berjalan** tanpa host, command, log, dan health evidence yang nyata.

## Tujuan

- mengubah metadata provisioning menjadi inventory contract non-secret;
- menjalankan readiness review sebelum configuration management;
- menyusun sequencing OpenTofu → Ansible → k3s;
- menetapkan stop condition, evidence, dan rollback decision.

## Guardrail

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Jangan menaruh SSH password/private key, k3s token, kubeconfig, access key, atau password di inventory, HCL, README, log, output, evidence, atau Git. Jangan melakukan mutation pada host production. Jangan menjalankan cluster reset, restore snapshot aktif, atau chaos test production.

## Bagian A — Static Handoff Contract (Wajib)

### 1. Buat metadata contoh

Gunakan hanya placeholder:

```yaml
hosts:
  cp1:
    hostname: <approved-control-plane-hostname>
    address: <approved-management-address>
    role: server
    environment: lab
    module_version: <approved-module-version>
    provisioning_ref: <non-secret-provider-reference>
  worker1:
    hostname: <approved-worker-hostname>
    address: <approved-management-address>
    role: agent
    environment: lab
    module_version: <approved-module-version>
    provisioning_ref: <non-secret-provider-reference>
```

Jangan mengisi placeholder dengan credential atau alamat produksi. Pastikan stable key, role, environment, version, dan reference konsisten.

### 2. Bentuk inventory konseptual

```yaml
all:
  vars:
    environment: lab
    k3s_version: <approved-version>
  hosts:
    cp1:
      ansible_host: <approved-management-address>
      node_role: server
    worker1:
      ansible_host: <approved-management-address>
      node_role: agent
```

SSH identity disediakan runner/secret mechanism terpisah. Inventory ini tidak membuktikan konektivitas.

### 3. Readiness matrix

Isi status `pass`, `fail`, atau `not-verified`:

| Gate | Pertanyaan | Status | Evidence/reference |
|---|---|---|---|
| Identity | hostname, role, environment benar? |  |  |
| Network | runner ke host dan node-to-node reachable? |  |  |
| DNS | forward/reverse lookup sesuai? |  |  |
| Time | NTP/chrony dan clock skew siap? |  |  |
| SSH | check connectivity memakai identity eksternal? |  |  |
| OS | distro/kernel/resource sesuai runbook? |  |  |
| Storage | path/capacity/durability siap? |  |  |
| Firewall | port matrix k3s disetujui? |  |  |
| Security | patch/user/sudo/hardening baseline siap? |  |  |

Jika salah satu gate kritis `fail` atau `not-verified`, stop sebelum Ansible/k3s.

## Bagian B — Sequencing Review

Tulis approval dan owner untuk setiap tahap:

```text
1. OpenTofu fmt/validate/plan
2. plan review dan approval
3. apply scope disposable/staging yang disetujui
4. collect metadata non-secret
5. host readiness
6. Ansible check mode/limited bootstrap
7. readiness ulang
8. k3s server bootstrap
9. k3s agent/server join sesuai quorum
10. kubectl context/node/workload health check
11. evidence dan handoff berikutnya
```

K3s cluster topology harus menjawab server count, quorum, datastore, agent count, API endpoint, CNI/ingress/load balancer, backup, upgrade, dan rollback.

## Bagian C — Runtime Lane (Hanya Jika Environment Disetujui)

Tidak ada provider on-prem, Ansible, atau k3s yang boleh diarahkan ke production untuk lab ini. Jika hanya ingin menguji bentuk contract, gunakan static lane. Bila Fase 4 dan lab disposable tersedia di masa depan, jalankan read-only/check mode lebih dahulu:

```bash
set -eu

# Jalankan dari repository/playbook yang telah diverifikasi.
ansible-inventory --graph
ansible-playbook --check --limit <approved-lab-host>
```

`<approved-lab-host>` adalah placeholder. Jangan menjalankan command tersebut tanpa inventory, identity, scope, dan approval yang benar. Check mode tidak selalu mensimulasikan semua side effect.

K3s health check setelah instalasi nyata harus mengikuti runbook yang disetujui dan setidaknya memeriksa context safety, node readiness, version, API access, dan workload smoke test. Jangan mencetak kubeconfig/token dan jangan menganggap command return code tunggal sebagai bukti cluster sehat.

## Evidence Handoff

Evidence minimum tanpa secret:

```text
source commit/module version
provider lock/checksum reference
backend/state key class
approved plan summary
host metadata checksum/reference
readiness matrix
Ansible job/check-mode result
k3s health summary
owner, timestamp, change/approval reference
runtime status
```

Tulis `belum diverifikasi` bila binary, host, endpoint, atau cluster tidak tersedia. Raw state, raw plan, token, kubeconfig, private key, dan password bukan evidence yang boleh di-commit.

## Failure Drill (Static)

### Skenario 1 — Provisioning sukses, SSH gagal

Stop Ansible. Periksa metadata address, DNS, route, firewall, guest readiness, dan identity. Jangan destroy otomatis.

### Skenario 2 — Ansible satu host gagal

Limit retry ke host yang disetujui, simpan redacted evidence, perbaiki root cause, dan ulangi readiness. Jangan rerun seluruh cluster tanpa review.

### Skenario 3 — k3s server join gagal

Stop join berikutnya. Periksa version, time, API route, token lifecycle melalui secret mechanism, hostname, dan quorum. Jangan reset cluster aktif.

### Skenario 4 — Plan ingin replace server yang sudah masuk cluster

Hentikan promotion. Buat migration plan: cordon/drain sesuai PDB, quorum check, replacement readiness, Ansible bootstrap, k3s rejoin, dan validation. Plan OpenTofu saja tidak cukup.

## Acceptance Criteria

- [ ] Metadata dan inventory hanya non-secret.
- [ ] Readiness matrix terisi dan memiliki stop condition.
- [ ] Sequencing setiap boundary memiliki owner.
- [ ] Quorum, backup, version, dan rollback k3s dibahas.
- [ ] Failure drill tidak memakai destructive shortcut.
- [ ] Runtime yang tidak dijalankan dinyatakan belum diverifikasi.

## Kaitan Berikutnya

Fase 4 — Ansible akan melatih inventory, role, idempotency, check mode, dan k3s bootstrap nyata. Modul 2.2/2.4 tetap menjadi rujukan topologi, context safety, backup, quorum, drain, dan rolling maintenance.

## Catatan SRE

Handoff yang aman bukan sekadar meneruskan IP. Handoff harus membawa metadata minimum, readiness evidence, owner, version, approval, dan keputusan berhenti. Credential mengikuti secret boundary yang tepat, bukan aliran output IaC.
