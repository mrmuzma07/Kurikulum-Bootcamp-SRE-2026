# 01 — OpenTofu Handoff, Inventory, dan Host Readiness

## 1. Contract Handoff

OpenTofu menghasilkan metadata minimum:

```yaml
all:
  vars:
    environment: lab
    provisioning_ref: <commit-or-run-reference>
    module_version: <approved-module-version>
  hosts:
    cp1:
      ansible_host: <management-address-cp1>
      node_role: server
      hostname: <guest-hostname-cp1>
    worker1:
      ansible_host: <management-address-worker1>
      node_role: agent
      hostname: <guest-hostname-worker1>
```

Tidak ada password, SSH key, k3s token, kubeconfig, provider credential, atau secret value. Credential diambil melalui mechanism terpisah saat runtime.

## 2. Readiness Matrix

| Gate | Evidence minimum | Stop condition |
|---|---|---|
| Identity | hostname, environment, owner cocok | address/role tidak cocok |
| Network | management/node route dan CIDR | route atau IP collision |
| DNS | forward/reverse sesuai policy | resolve tidak konsisten |
| Time | sync service/status | skew di atas threshold |
| SSH/sudo | connection dan privilege test | access belum dapat dipulihkan |
| OS/kernel | supported release/resource | unsupported image/module |
| Storage | disk/mount/free space | kapasitas tidak cukup |
| Firewall/security | policy dan API path | akses cluster terblokir |
| Image/cloud-init | marker/log/first boot | initialization belum selesai |

`not-verified` berarti berhenti, bukan “lanjut dengan asumsi”.

## 3. Preflight Design

```yaml
- name: Host readiness gate
  hosts: k3s_nodes
  gather_facts: true
  any_errors_fatal: true
  tasks:
    - name: Pastikan environment sesuai
      ansible.builtin.assert:
        that: environment == 'lab'
        fail_msg: "Environment mismatch; hentikan play"

    - name: Pastikan facts minimum tersedia
      ansible.builtin.assert:
        that:
          - ansible_facts.os_family is defined
          - ansible_facts.architecture is defined
```

Assertion tidak menggantikan pemeriksaan network, disk, time, SSH, atau firewall. Tambahkan probe read-only yang relevan dan redacted.

## 4. Ownership dan Failure

OpenTofu menangani lifecycle VM/network/storage; Ansible tidak boleh diam-diam memperbaiki state provider. Jika host diganti, review state address, replacement, DNS, inventory, dan dampak terhadap node yang sudah masuk cluster.

## Acceptance Checklist

- [ ] Handoff hanya metadata non-secret.
- [ ] Semua gate memiliki evidence dan stop condition.
- [ ] Environment/identity diverifikasi sebelum mutation.
- [ ] Replacement host memiliki migration plan.

## Catatan SRE

Readiness adalah kontrak, bukan checklist kosmetik. Satu gate kritis yang belum diverifikasi dapat membuat task berikutnya gagal dengan blast radius lebih besar.

## Kaitan

Lanjutkan ke [k3s Install dan Upgrade Role](02-k3s-install-upgrade-role.md) dan [LAB-01](lab/LAB-01-handoff-k3s-multinode.md).
