# LAB-01 — CI Lint, Test, Build Multi-Arch, dan Publish

## Tujuan

Merancang pipeline GitLab CI yang menghasilkan artifact aplikasi secara reproducible tanpa memberi pipeline aplikasi akses langsung ke cluster production.

## Prasyarat

- [Modul 6.1](../README.md).
- [Pipeline stages, jobs, dan rules](../01-pipeline-stages-jobs-rules.md).
- [Runner, artifact, cache, dan environment](../02-runner-artifacts-cache-environment.md).
- Dasar container dari [Fase 1](../../../fase-1-container-orbstack/README.md) dan Helm dari [Fase 5](../../../fase-5-helm/README.md).

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Gunakan placeholder untuk image CI, registry, runner tag, dan identity. Jangan mencetak `CI_JOB_TOKEN`, credential registry, private key, kubeconfig, atau environment variable lengkap. Jangan push ke registry nyata tanpa target disposable, approval, dan bukti yang dapat diredáksi.

## Lane A — Static Simulation

### 1. Buat graph pipeline

Gambarkan graph berikut dan jelaskan dependency setiap job:

```text
lint/schema → unit-test → chart-template
                         ↓
                  build-amd64/arm64
                         ↓
                 scan/sign/provenance
                         ↓
                  publish-by-digest
                         ↓
             reviewed GitOps change
```

Tentukan job mana yang boleh berjalan pada merge request dan job mana yang hanya boleh berjalan dari protected branch/tag.

### 2. Tulis desain `.gitlab-ci.yml`

Gunakan nilai non-secret:

```yaml
stages: [validate, test, build, verify, publish]

workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

lint:
  stage: validate
  image: <approved-ci-image>
  script: ["./scripts/lint"]

unit-test:
  stage: test
  needs: [lint]
  script: ["./scripts/test"]

build-image:
  stage: build
  needs: [unit-test]
  script:
    - ./scripts/build-multiarch --platform linux/amd64,linux/arm64 --tag "${CI_COMMIT_SHA}"
  artifacts:
    reports:
      dotenv: build-metadata.env

publish-image:
  stage: publish
  needs:
    - job: build-image
      artifacts: true
  script: ["./scripts/publish-by-digest"]
  environment:
    name: <approved-registry-environment>
  resource_group: <approved-registry-group>
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

Nilai dotenv hanya boleh memuat metadata non-secret seperti digest, commit, platform, dan chart reference yang sudah disetujui.

### 3. Review rules dan gate

Buat tabel yang menjawab:

| Kondisi | Pipeline/job | Gate |
|---|---|---|
| MR mengubah source | lint/test/template | wajib |
| MR mengubah Dockerfile/CI | security review tambahan | wajib |
| default branch | build/scan | protected branch |
| publish registry | publish-by-digest | manual approval |
| update GitOps | merge request terpisah | reviewer release |

Periksa kemungkinan duplicate pipeline dari `workflow:rules`, pipeline source, merge train, dan retry. Tentukan `interruptible` hanya untuk job yang aman dibatalkan.

### 4. Review multi-architecture

Jelaskan:

- perbedaan host Mac ARM64 dan target `linux/amd64,linux/arm64`;
- cara memverifikasi manifest platform secara read-only;
- risiko emulasi dan privileged BuildKit;
- alasan image tag commit/digest lebih aman daripada `latest`;
- cara menyimpan checksum/reference tanpa menyimpan credential.

### 5. Review artifact dan cache

Klasifikasikan setiap output sebagai artifact atau cache:

- test summary;
- SBOM/provenance;
- image digest;
- dependency download;
- rendered Secret;
- raw build log;
- kubeconfig.

Output terakhir harus ditolak atau diredáksi jika memuat credential, raw secret, private key, kubeconfig, raw Helm output, atau environment dump.

## Lane B — Optional Disposable Runtime

Hanya lakukan bila runner dan registry disposable telah disediakan, target, waktu, scope, approval, dan recovery path terdokumentasi.

1. Jalankan pipeline lint/template validation pada branch lab.
2. Pastikan job ID, commit SHA, runner architecture, dan waktu dicatat.
3. Build ke registry disposable dengan digest immutable.
4. Verifikasi manifest platform secara read-only.
5. Simpan hanya evidence redacted: job ID, digest, status, platform, dan expiry artifact.
6. Cabut atau expire artifact dan credential sesuai prosedur setelah lab.

Tanpa evidence aktual, status build/push adalah **belum diverifikasi**. Pipeline hijau tidak membuktikan deployment, readiness, atau SLO.

## Stop Conditions

Hentikan lab bila:

- pipeline dari fork/untrusted MR memperoleh credential protected;
- job berjalan pada runner atau architecture yang tidak disetujui;
- cache berasal dari scope tidak dipercaya;
- tag mutable menjadi promotion reference;
- log memuat token, kubeconfig, atau registry credential;
- publish terjadi tanpa approval atau resource serialization.

## Evidence Template

```text
lab: LAB-01
source_commit: <redacted-sha>
pipeline_id: <pipeline-id>
job_ids: <job-id-list>
runner_architecture: <amd64-or-arm64>
image_digest: <immutable-digest-reference>
platforms: <approved-platform-list>
gitops_change: <merge-request-or-commit-reference>
status: <verified-or-belum-diverifikasi>
redaction_review: <review-reference>
```

## Acceptance Criteria

- [ ] Graph dan `needs` dapat dijelaskan.
- [ ] MR, protected branch, manual publish, dan environment gate dipisahkan.
- [ ] Image reference immutable dan multi-arch diverifikasi secara tepat.
- [ ] Artifact/cache boundary dan retention ditentukan.
- [ ] Tidak ada credential literal atau raw secret pada repository/evidence.
- [ ] Runtime, bila tidak memiliki evidence, ditulis **belum diverifikasi**.

## Troubleshooting

- Job tidak muncul: periksa `workflow:rules`, `rules:changes`, dan source pipeline.
- Job duplicate: periksa kombinasi MR dan branch pipeline.
- Manifest hanya ARM64: periksa builder platform dan target manifest.
- Publish gagal: bedakan auth, permission, network, quota, dan digest mismatch; jangan mencetak credential saat debug.
- Artifact tidak tersedia: periksa `needs:artifacts`, expiry, dan permission.

## Kaitan

Lanjutkan ke [LAB-02 IaC plan dan approval](LAB-02-iac-plan-approval-lab-run.md) dan [Modul 6.2](../../modul-6.2-argocd/README.md).
