# 03 — Check, Diff, Limit, Error Handling, dan CI

## 1. Execution Guardrail

```bash
ansible-playbook -i inventories/lab/hosts.yml playbooks/common.yml \
  --check --diff --limit worker1
```

`--limit` bukan sekadar optimasi; ini kontrol blast radius. Pastikan limit cocok dengan inventory graph, environment, maintenance window, dan approval. `--diff` dapat mengungkap konfigurasi sensitif, jadi jangan simpan output mentah.

## 2. Serial dan Failure

```yaml
- name: Patch node secara rolling
  hosts: patchable_nodes
  serial: 1
  any_errors_fatal: true
  tasks:
    - name: Preflight marker
      ansible.builtin.assert:
        that:
          - environment == 'lab'
        fail_msg: "Environment bukan lab; stop sebelum mutation"
```

`assert` hanya satu gate; readiness harus lebih luas. `any_errors_fatal` menghentikan play ketika failure kritis, tetapi recovery dan evidence tetap perlu.

## 3. Block/Rescue/Always

```yaml
- name: Perubahan dengan evidence
  block:
    - name: Terapkan perubahan terbatas
      ansible.builtin.copy:
        content: "managed=true\n"
        dest: /var/lib/sre-marker
        mode: "0644"
  rescue:
    - name: Catat failure class tanpa detail sensitif
      ansible.builtin.debug:
        msg: "Perubahan gagal; hentikan promotion dan review target"
  always:
    - name: Simpan status non-secret
      ansible.builtin.set_fact:
        change_evidence: "completed-or-failed-redacted"
```

Jangan memakai rescue untuk melanjutkan seolah-olah sukses. Failure harus terlihat dan menghentikan langkah yang bergantung.

## 4. CI Gate

Urutan konseptual:

```text
YAML/lint → ansible-lint → syntax-check → inventory graph
→ check/diff pada environment terbatas → secret/artifact scan
→ approval → runtime apply dengan protected reference
```

CI runner tidak otomatis memiliki akses production. Protected variable/secret injection harus berasal dari sistem yang disetujui dan tidak dicetak ke log. Pipeline belum boleh diklaim berhasil tanpa job evidence.

## Troubleshooting

- `--check` berbeda dengan actual: baca dukungan check mode module.
- diff terlalu luas: batasi `--limit`, review variable precedence, dan pecah commit.
- task gagal setelah sebagian host berubah: hentikan, klasifikasikan partial state, jangan retry massal.
- lint lolos tetapi service gagal: parsing bukan integration/health check.

## Acceptance Checklist

- [ ] Check/diff/limit dipakai dalam urutan review.
- [ ] Rolling play memiliki serial dan stop condition.
- [ ] Failure tidak disamarkan oleh rescue.
- [ ] CI memisahkan lint/check dari approval/apply.
- [ ] Artifact dan log melalui redaction/secret scan.

## Catatan SRE

Pipeline yang cepat gagal sebelum perubahan lebih baik daripada pipeline yang “hijau” tetapi menyentuh host yang salah. Ukur keberhasilan dengan evidence dan health, bukan exit code tunggal.

## Kaitan

Materi ini dipraktikkan pada [LAB-02](lab/LAB-02-vault-safe-ci-validation.md) dan menjadi guardrail Modul 4.3.
