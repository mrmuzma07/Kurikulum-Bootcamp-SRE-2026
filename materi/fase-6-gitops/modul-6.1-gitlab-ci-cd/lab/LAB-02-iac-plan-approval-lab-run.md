# LAB-02 — OpenTofu Plan, Approval, dan Ansible Lab Run

## Tujuan

Membuat pipeline IaC/configuration yang memisahkan validasi, plan, review, approval, apply, readiness, dan evidence dengan blast radius terbatas.

## Prasyarat

- [Pipeline OpenTofu, Ansible, dan approval](../03-iac-ansible-pipeline-approval.md).
- [Fase 3 OpenTofu](../../../fase-3-opentofu/README.md).
- [Fase 4 Ansible](../../../fase-4-ansible/README.md).

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Lab runtime hanya boleh memakai environment disposable yang disetujui. Jangan menjalankan apply, SSH mutation, firewall change, k3s change, destroy, import, `state rm`, atau `state mv` pada target yang tidak diverifikasi. Jangan memakai `-auto-approve` sebagai default.

## Lane A — Static Simulation

### 1. OpenTofu pipeline

Rancang job berikut:

```text
fmt → validate → plan pada merge request
→ review diff redacted → protected approval
→ apply approved plan pada environment lab
→ readiness summary
```

Gunakan desain non-secret:

```yaml
tofu-plan:
  stage: validate
  script:
    - tofu fmt -check
    - tofu init -backend=false
    - tofu validate
    - tofu plan -out=<approved-plan-path>
  artifacts:
    expire_in: <short-retention>
    paths:
      - plan-summary-redacted.txt
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

tofu-apply:
  stage: publish
  script:
    - ./scripts/apply-approved-plan
  environment:
    name: <approved-lab-environment>
  when: manual
  resource_group: <approved-state-lock-group>
```

Jelaskan mengapa raw plan/state tidak boleh di-upload ke artifact terbuka. Bedakan `tofu plan` dari bukti apply. Pastikan backend, lock, identity, state access, backup, dan recovery berada di boundary platform.

### 2. Ansible validation

Rancang urutan:

```bash
ansible-lint <approved-playbook-path>
ansible-playbook <approved-playbook-path> --syntax-check
ansible-playbook <approved-playbook-path> --check --diff --limit <approved-lab-group>
```

Tentukan penggunaan `serial`, `no_log`, privilege boundary, inventory reference, readiness check, dan stop condition. Ingat bahwa `--check` dan `--diff` bukan jaminan zero side effect.

### 3. Handoff OpenTofu → Ansible

Buat metadata non-secret:

```text
stable_key → management_address → role → environment
→ approved_version → secret_reference → readiness_contract
```

Metadata tidak boleh membawa password, private key, token, kubeconfig, atau raw inventory secret. Ansible memperoleh transport credential melalui protected mechanism di luar Git.

### 4. Approval checklist

Reviewer harus menjawab:

- target dan environment apa yang berubah;
- commit dan dependency lock yang digunakan;
- apakah plan sama dengan yang disetujui;
- state lock siapa yang memegang;
- apakah `--limit` benar-benar membatasi target;
- apakah backup dan access recovery tersedia;
- bagaimana menghentikan run dan melakukan rollback;
- evidence apa yang akan disimpan setelah readiness.

## Lane B — Optional Disposable Runtime

Dengan approval dan target lab:

1. Verifikasi context, inventory scope, state backend, lock, dan target address secara read-only.
2. Jalankan fmt/validate/lint/syntax-check.
3. Jalankan plan/check/diff terbatas dan simpan summary redacted.
4. Minta approval yang merujuk commit, plan hash/reference, target, dan expiry.
5. Jalankan apply/playbook secara serial pada scope lab.
6. Verifikasi readiness host/service secara read-only dan redacted.
7. Catat job ID, plan reference, run ID, target group, waktu, dan outcome.

Jika runtime tidak tersedia, status OpenTofu apply/Ansible run adalah **belum diverifikasi**. Jangan mengganti dengan klaim “berhasil” dari syntax-check atau contoh dokumentasi.

## Stop Conditions

- plan berubah setelah approval;
- state lock tidak cocok atau bypass diminta;
- host di luar `--limit` muncul;
- inventory atau log memuat credential;
- target bukan disposable/approved;
- readiness gagal atau access recovery tidak tersedia;
- apply membutuhkan destructive shortcut.

## Evidence Template

```text
lab: LAB-02
source_commit: <redacted-sha>
plan_reference: <redacted-plan-reference>
plan_hash: <approved-hash-reference>
apply_job_id: <job-id-or-not-run>
ansible_run_id: <run-id-or-not-run>
target_scope: <approved-lab-group>
readiness: <summary-redacted>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] OpenTofu plan/apply dan approval terpisah.
- [ ] State lock/backend/plan retention dan access boundary dijelaskan.
- [ ] Ansible lint, syntax-check, check/diff, `--limit`, serial, dan `no_log` digunakan.
- [ ] Handoff hanya metadata non-secret.
- [ ] Evidence tidak memuat raw plan/state/inventory secret.
- [ ] Runtime tanpa evidence dilaporkan **belum diverifikasi**.

## Troubleshooting

- Plan/apply berbeda: hentikan, bandingkan commit/provider/lock, dan minta review ulang.
- Lock gagal: jangan bypass; cari owner run dan prosedur recovery.
- Ansible menyentuh host salah: stop segera, periksa inventory/limit/variable precedence.
- Readiness gagal: klasifikasikan host, network, storage, time sync, service, atau permission.

## Kaitan

- [Modul 3.3](../../../fase-3-opentofu/modul-3.3-konteks-onprem/README.md)
- [Modul 4.3](../../../fase-4-ansible/modul-4.3-ansible-k3s-onprem/README.md)
- [Modul 6.3](../../modul-6.3-end-to-end-flow/README.md)
