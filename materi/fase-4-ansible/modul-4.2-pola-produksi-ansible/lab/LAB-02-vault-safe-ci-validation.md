# LAB-02 — Vault Boundary dan CI Validation

## Tujuan

Mendesain pipeline yang melakukan lint, syntax check, inventory review, check mode, dan secret scan tanpa mencetak secret.

## Pipeline Konseptual

```text
lint → syntax-check → inventory graph → check/diff terbatas
→ secret/artifact scan → approval → protected runtime
```

## Langkah

1. Tulis job names dan input/output non-secret.
2. Tetapkan inventory environment dan `--limit`.
3. Tandai job yang membutuhkan protected secret reference.
4. Pastikan raw diff, decrypted Vault file, dan verbose log tidak menjadi artifact.
5. Tambahkan failure gate untuk environment mismatch dan secret scan.
6. Dokumentasikan approval sebelum mutation.

Contoh placeholder, bukan eksekusi:

```bash
ansible-playbook -i inventories/lab/hosts.yml playbooks/common.yml --syntax-check
ansible-playbook -i inventories/lab/hosts.yml playbooks/common.yml --check --limit <approved-lab-host>
```

## Acceptance Criteria

- [ ] Lint/check dipisahkan dari apply.
- [ ] Secret reference tidak dicetak.
- [ ] Diff/log melalui redaction.
- [ ] Approval dan stop condition jelas.
- [ ] Pipeline tidak diklaim berhasil tanpa job evidence.

## Catatan SRE

CI adalah control plane perubahan, tetapi runner bukan otomatis trusted production host. Verify identity dan policy di setiap stage.
