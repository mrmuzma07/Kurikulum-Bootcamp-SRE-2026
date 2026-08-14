# Modul 4.1 — Fundamental Ansible

> **Tujuan:** memahami control node, managed node, SSH, inventory, playbook YAML, dan idempotency sebelum membuat role production.

## Capaian

- [ ] Menjelaskan arsitektur agentless control node → SSH → managed node.
- [ ] Menyusun inventory static dengan group dan variable non-secret.
- [ ] Menggunakan `ansible.cfg`, ad-hoc command, play, task, module, handler, loop, condition, dan tag.
- [ ] Menulis playbook idempotent dan menjelaskan hasil rerun.
- [ ] Memakai `--check`, `--diff`, `--limit`, dan preflight sebagai guardrail.

## Rencana 2 Hari

| Hari | Materi | Lab |
|---|---|---|
| 1 | [Arsitektur, Inventory, SSH](01-arsitektur-inventory-ssh.md), [Ad-hoc & Playbook](02-ad-hoc-playbook-yaml.md) | [LAB-01](lab/LAB-01-inventory-ssh-ad-hoc.md) |
| 2 | [Idempotency, Variables, Facts, Handlers](03-idempotency-variables-facts-handlers.md) | [LAB-02](lab/LAB-02-playbook-common-idempotent.md) + evaluasi |

## Prasyarat dan Lane

Linux, SSH, YAML, dan Modul 3.3. Static lane memakai placeholder; runtime lane membutuhkan `ansible` dan VM disposable yang disetujui. Tidak ada akses credential nyata yang diperlukan untuk belajar konsep.

## Deliverables

Inventory environment lab, `ansible.cfg`, playbook common sederhana, catatan first run/rerun, dan readiness evidence template.

## Guardrail

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menaruh private key atau password di inventory. Gunakan reference ke secret mechanism terpisah.
- Jangan menjalankan ad-hoc package/service pada host yang scope-nya tidak diverifikasi.
- `--check` bukan jaminan zero side effect; tetap review module dan target.
- Selalu gunakan `--limit` pada runtime lab dan berhenti jika identity/environment salah.

## Acceptance Criteria

- [ ] Inventory membedakan management address dan node address.
- [ ] Playbook memiliki handler, kondisi, dan module state yang idempotent.
- [ ] Rerun dibahas tanpa mengklaim eksekusi jika runtime tidak tersedia.
- [ ] Output contoh bebas credential dan raw diff sensitif.

## Status Runtime dan Kaitan

Status Ansible/SSH harus ditentukan oleh preflight, bukan diasumsikan. Modul ini meneruskan metadata dari [Modul 3.3](../../fase-3-opentofu/modul-3.3-konteks-onprem/README.md) dan menjadi prasyarat [Modul 4.2](../modul-4.2-pola-produksi-ansible/README.md).

## Catatan SRE

Inventory adalah boundary keselamatan. Nama group yang salah dapat menjalankan task pada node yang salah; karena itu identity check, `--limit`, dan evidence sebelum mutation sama pentingnya dengan sintaks YAML.
