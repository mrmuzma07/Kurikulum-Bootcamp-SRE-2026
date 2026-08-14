# 01 — Pipeline Stages, Jobs, Rules, dan Gate

## Tujuan

Memahami pipeline GitLab sebagai directed graph yang memvalidasi perubahan sebelum artifact dan environment disentuh.

## Model Pipeline

```text
source commit/MR
  → lint + schema + chart render
  → unit/integration test
  → build multi-arch
  → scan/sign/provenance
  → push immutable artifact
  → update GitOps repository via reviewed change
```

`stages` memberi urutan kasar. `needs` membuat dependency graph lebih tepat dan dapat mengurangi waktu tunggu. `rules` harus ditulis eksplisit berdasarkan pipeline source, branch, file change, dan approval state.

## Contoh Non-Secret

```yaml
stages:
  - validate
  - test
  - build
  - publish

workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

lint:
  stage: validate
  image: <approved-ci-image>
  script:
    - ./scripts/lint
  rules:
    - changes:
        - "**/*.go"
        - "**/*.yaml"
        - ".gitlab-ci.yml"

unit-test:
  stage: test
  needs: [lint]
  script:
    - ./scripts/test

build-image:
  stage: build
  needs: [unit-test]
  script:
    - ./scripts/build-multiarch --tag "${CI_COMMIT_SHA}"
  artifacts:
    reports:
      dotenv: build-metadata.env

publish-image:
  stage: publish
  needs:
    - job: build-image
      artifacts: true
  script:
    - ./scripts/publish-by-digest
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
```

Contoh hanya desain. Tidak ada pipeline yang dianggap berjalan tanpa job ID, target, waktu, log redacted, dan artifact evidence.

## Rules dan Gate

Review:

- MR wajib lint/test dan chart render;
- branch protected mencegah push langsung;
- publish hanya dari branch/tag yang disetujui;
- production environment memerlukan approval;
- perubahan `.gitlab-ci.yml`, Dockerfile, chart, atau IaC memicu gate yang sesuai;
- cancel/retry tidak boleh menghasilkan dua promotion yang tidak dapat dibedakan;
- gunakan `resource_group` atau concurrency policy untuk target yang serialized.

## Artifacts versus Cache

- **Artifact:** output job yang ditelusuri, memiliki retention dan permission; jangan menyimpan secret/raw state.
- **Cache:** akselerasi dan dapat hilang/terkontaminasi; bukan evidence atau source of truth.
- `dotenv` hanya memuat metadata non-secret seperti digest/reference yang aman.

## Evidence

```text
commit SHA → pipeline ID → job ID → test summary → image digest
→ GitOps change reference → approval → target environment
```

Redact URL privat, token, kubeconfig, raw logs, dan rendered Secret.

## Troubleshooting

- Rules overlap: gunakan pipeline lint dan uji source branch/MR/tag secara terpisah.
- Job duplicate: periksa `workflow:rules`, `rules`, dan merge train behavior.
- Artifact missing: periksa `needs:artifacts`, expiry, dan job dependency.
- Publish terpicu dari MR: tambahkan protected branch/tag dan manual gate.

## Acceptance Criteria

- [ ] Graph valid untuk MR dan default branch.
- [ ] Publish tidak otomatis berjalan dari source yang tidak dipercaya.
- [ ] Artifact/cache boundary jelas.
- [ ] Pipeline evidence tidak membawa credential.

## Kaitan

Lanjutkan ke [runner/artifact/cache](02-runner-artifacts-cache-environment.md) dan [LAB-01](lab/LAB-01-ci-lint-test-build-push.md).

## Catatan SRE

Pipeline duration dan queue time adalah bagian dari reliability delivery. Gate yang tidak deterministic akan mendorong bypass manual dan memperbesar risiko perubahan.
