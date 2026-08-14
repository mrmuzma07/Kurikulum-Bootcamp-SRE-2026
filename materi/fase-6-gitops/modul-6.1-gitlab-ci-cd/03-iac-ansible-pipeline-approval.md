# 03 — Pipeline OpenTofu, Ansible, dan Approval

## Tujuan

Membuat pipeline infrastructure/configuration yang memisahkan plan, review, approval, apply, readiness, dan evidence.

## OpenTofu Pipeline

```text
format → validate → plan pada MR
→ plan review/approval → apply pada protected main
→ output summary redacted → readiness evidence
```

Contoh job desain:

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
    name: <approved-environment>
  when: manual
  resource_group: <approved-state-lock-group>
```

`tofu plan` dapat memuat nilai sensitif atau resource detail. Gunakan backend/identity yang disetujui, lock, artifact access control, dan summary redacted. Jangan menyimpan raw state, raw plan, provider credential, atau `-auto-approve` sebagai default.

## Ansible Pipeline

```text
inventory/reference validation → ansible-lint
→ syntax-check → check/diff pada --limit lab
→ approval → playbook run serial
→ readiness/health evidence
```

Contoh command placeholder:

```bash
ansible-lint <approved-playbook-path>
ansible-playbook <approved-playbook-path> \
  --syntax-check
ansible-playbook <approved-playbook-path> \
  --check --diff --limit <approved-lab-group>
```

`--check` dan `--diff` tidak menjamin zero side effect. Module tertentu tetap dapat berinteraksi dengan target. Review `--limit`, inventory, privilege, `no_log`, `serial`, maintenance window, dan recovery.

## Handoff

OpenTofu menghasilkan metadata non-secret:

```text
stable key → management address → role → environment → approved version → secret reference
```

Ansible memakai metadata tersebut untuk readiness dan configuration. Credential transport harus berasal dari protected mechanism; tidak boleh ikut dalam output OpenTofu atau inventory commit.

## Approval dan Stop Conditions

Hentikan apply/run bila:

- target atau state lock tidak cocok;
- plan berubah setelah approval;
- inventory membawa credential atau host di luar scope;
- `--limit` tidak membatasi target;
- backup/access recovery/rollback tidak tersedia;
- readiness atau maintenance condition gagal.

## Acceptance Criteria

- [ ] Plan MR dapat dibandingkan dengan perubahan yang disetujui.
- [ ] Apply hanya pada protected branch/environment dan state lock yang benar.
- [ ] Ansible lint/check/diff/run memiliki limit dan approval.
- [ ] Output dan artifact sensitif diredáksi.

## Troubleshooting

- Plan berbeda saat apply: lock dependency, state, commit, dan plan artifact; jangan apply ulang tanpa review.
- State lock gagal: jangan bypass lock; cari owner/job aktif dan prosedur recovery.
- Ansible changed terlalu banyak: stop, periksa inventory/variable precedence, dan jalankan ulang static review.
- Readiness gagal: hentikan handoff ke k3s; klasifikasikan host, network, time, storage, atau service.

## Kaitan

- [Modul 3.3 — handoff](../../fase-3-opentofu/modul-3.3-konteks-onprem/README.md)
- [Modul 4.3 — Ansible k3s](../../fase-4-ansible/modul-4.3-ansible-k3s-onprem/README.md)
- [Modul 6.1](README.md)

## Catatan SRE

Approval bukan formalitas. Reviewer harus mampu menjawab target apa yang berubah, mengapa, siapa yang menyetujui, cara berhenti, dan bagaimana memulihkan akses.
