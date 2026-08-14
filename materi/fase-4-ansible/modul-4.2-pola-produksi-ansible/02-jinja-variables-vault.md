# 02 — Jinja2, Variable Precedence, dan Vault Boundary

## 1. Template Non-Secret

```jinja2
# templates/sre-marker.conf.j2
environment={{ environment | default('lab', true) }}
node_role={{ node_role | default('unknown', true) }}
managed_by=ansible
```

Gunakan `default` dan validasi input. Hindari interpolasi string yang dapat menghasilkan konfigurasi invalid. Template yang lolos syntax bukan bukti service menerima konfigurasi tersebut.

## 2. Variable Precedence

Ansible memiliki banyak sumber variable: role defaults, inventory/group vars, host vars, play vars, extra vars, dan facts. Tetapkan kebijakan sederhana: environment menentukan inventory, role menyediakan defaults, dan override berisiko memerlukan review. Jangan menyimpan secret pada `group_vars` biasa.

## 3. Secret Boundary

Kredensial hanya direferensikan secara abstrak:

```yaml
k3s_token_ref: <secret-reference>
ssh_credential_ref: <secret-reference>
```

`ansible-vault` dapat mengenkripsi file yang memang disetujui untuk disimpan, tetapi password Vault, decrypted content, token, kubeconfig, dan private key tidak boleh masuk repository atau log. Vault bukan pengganti access policy, rotation, audit, dan backup.

Contoh workflow placeholder:

```bash
ansible-vault view <approved-encrypted-file>
ansible-playbook -i inventories/lab/hosts.yml playbooks/k3s.yml --ask-vault-pass
```

Jangan menganggap command di atas telah dijalankan.

## 4. no_log dan Redaction

`no_log: true` mencegah sebagian output, tetapi dapat mengurangi observability. Gunakan hanya pada task yang memproses secret dan tetap catat status non-secret, reference, serta failure class.

## 5. Template Validation

Sebelum reload, gunakan `validate` bila module/service mendukung. Jalankan validasi terhadap target disposable atau lint static; jangan menyalin output konfigurasi sensitif ke evidence.

## Acceptance Checklist

- [ ] Template hanya berisi data non-secret.
- [ ] Precedence dan ownership variable jelas.
- [ ] Vault workflow tidak berisi password/payload nyata.
- [ ] Redaction dan `no_log` tidak menutupi failure.
- [ ] Default serta validation mencegah input kosong/berbahaya.

## Catatan SRE

Secret leakage sering terjadi lewat diff, exception, debug, dan artifact CI—bukan hanya file variable. Perlakukan seluruh execution output sebagai data yang harus diklasifikasikan.

## Kaitan

Lanjutkan ke [Check, Diff, Limit, Error Handling, CI](03-check-diff-limit-error-handling-ci.md).
