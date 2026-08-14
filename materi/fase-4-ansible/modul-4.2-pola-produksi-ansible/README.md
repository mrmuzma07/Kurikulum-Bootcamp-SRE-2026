# Modul 4.2 — Pola Produksi Ansible

> **Tujuan:** menyusun repository Ansible dengan role, template, variable contract, Vault boundary, dan execution guardrail yang dapat dipromosikan.

## Capaian

- [ ] Menjelaskan struktur role, collections, dependency, dan version pinning.
- [ ] Menggunakan Jinja2 template, defaults, validation, handler, dan variable precedence.
- [ ] Membatasi blast radius dengan `--check`, `--diff`, `--limit`, `serial`, dan approval.
- [ ] Menangani failure memakai `block/rescue/always` tanpa menutupi root cause.
- [ ] Menjelaskan Vault encrypted-at-rest dan protected secret injection tanpa membocorkan nilai.
- [ ] Mendesain CI lint/syntax/check gate tanpa klaim pipeline berhasil.

## Rencana 2 Hari

| Hari | Materi | Lab |
|---|---|---|
| 1 | [Roles, Collections, Repository](01-roles-collections-repository.md), [Jinja2, Variables, Vault](02-jinja-variables-vault.md) | [LAB-01](lab/LAB-01-role-common-hardening.md) |
| 2 | [Check, Diff, Limit, Error, CI](03-check-diff-limit-error-handling-ci.md) | [LAB-02](lab/LAB-02-vault-safe-ci-validation.md) + evaluasi |

## Prasyarat dan Lane

Selesaikan Modul 4.1 dan Git workflow. Static lane memeriksa struktur serta contoh; runtime lane hanya pada host disposable. Vault, CI, dan collections boleh dipelajari tanpa menyimpan credential.

## Deliverables

Skeleton role `common`, template non-secret, molecule/validation design, CI gate, dan evidence checklist yang redacted.

## Guardrail

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menulis vault password, encrypted payload contoh yang berasal dari sistem nyata, atau decrypted output ke Git.
- `no_log: true` hanya untuk data sensitif, bukan untuk menyembunyikan task yang gagal.
- `--diff` dapat menampilkan isi sensitif; review dan redaction sebelum menyimpan output.
- `--limit` dan approval wajib untuk mutation. Jangan mengubah SSH/firewall pada host yang tidak disetujui.

## Acceptance Criteria

- [ ] Role memiliki input contract dan handler yang dapat direview.
- [ ] Template memakai defaults/validation dan tidak memuat secret.
- [ ] CI gate memisahkan lint/syntax/check dari apply dan memiliki protected secret boundary.
- [ ] Failure path, rollback, dan evidence dijelaskan.

## Status Runtime dan Kaitan

Parsing atau desain role tidak membuktikan execution. Modul ini menerima fondasi Modul 4.1 dan menyiapkan [Modul 4.3](../modul-4.3-ansible-k3s-onprem/README.md) untuk host serta k3s.

## Catatan SRE

Role yang rapi bukan hanya folder yang rapi. Input, ownership, failure semantics, dan evidence harus stabil agar perubahan dapat dipromosikan tanpa kejutan address atau blast radius.
