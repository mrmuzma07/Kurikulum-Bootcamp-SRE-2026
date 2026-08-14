# 03 — Hardening, Patching, dan Rolling Maintenance

## 1. Common Hardening

Scope hardening dapat meliputi user non-root/sudo, SSH policy, firewall, fail2ban, automatic security updates, time sync, kernel/module, dan container runtime prerequisites. Setiap perubahan harus memiliki access recovery, compatibility review, dan rollback.

Contoh desain variable:

```yaml
hardening_manage_firewall: false
hardening_enable_fail2ban: false
hardening_automatic_updates: false
```

Default konservatif mencegah side effect, bukan pengganti policy. Mengaktifkan flag pada production memerlukan change review dan test path.

## 2. Rolling Patch

```yaml
- name: Patch node satu per satu
  hosts: k3s_nodes
  serial: 1
  tasks:
    - name: Gate environment
      ansible.builtin.assert:
        that: environment == 'lab'
    - name: Beri instruksi maintenance
      ansible.builtin.debug:
        msg: "Cordon/drain dan PDB harus disetujui oleh owner k3s sebelum patch"
```

Task drain tidak ditulis sebagai shortcut. Operator harus memeriksa replica, PDB, quorum, critical workloads, API access, dan maintenance window. Setelah patch: uncordon bila sesuai runbook, tunggu node Ready, lalu lakukan smoke test.

## 3. SSH dan Firewall

Perubahan SSH/firewall dapat memutus control node. Gunakan staged rollout satu node, alternate access path, config validation, dan rollback. Jangan mematikan host key checking atau membuka port luas sebagai debugging shortcut.

## 4. Failure Handling

Jika node pertama gagal:

1. stop play dan jangan lanjut massal;
2. pertahankan evidence redacted;
3. cek access, service, node condition, quorum, dan workload;
4. restore/rollback sesuai runbook;
5. minta approval sebelum melanjutkan node berikutnya.

## Acceptance Checklist

- [ ] Hardening flags memiliki default dan approval boundary.
- [ ] Rolling task memakai `serial: 1` dan health gate.
- [ ] PDB/replica/quorum serta access recovery dibahas.
- [ ] Failure pada satu node menghentikan promotion.

## Catatan SRE

Patching bukan lomba menyelesaikan semua host. Kecepatan yang tidak mempertimbangkan quorum dan recovery dapat mengubah maintenance terencana menjadi incident.

## Kaitan

Lanjutkan ke [Backup, Health, Evidence, Rebuild](04-backup-health-evidence-rebuild.md) dan [LAB-02](lab/LAB-02-rolling-patching-readiness.md).
