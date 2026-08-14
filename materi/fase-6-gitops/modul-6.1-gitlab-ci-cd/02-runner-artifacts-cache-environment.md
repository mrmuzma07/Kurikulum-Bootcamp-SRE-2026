# 02 — Runner, Artifact, Cache, dan Environment

## Tujuan

Memilih execution environment yang sesuai untuk GitLab CI on-prem simulation dan mengelola output job tanpa memperluas blast radius atau membocorkan secret.

## Shared dan Self-hosted Runner

| Runner | Kelebihan | Risiko yang harus dikendalikan |
|---|---|---|
| Shared GitLab runner | cepat digunakan, maintenance rendah | trust boundary, quota, architecture, ephemeral state |
| Self-hosted OrbStack | simulasi on-prem, kontrol image/network/cache | patching runner, privileged build, persistence, credential exposure |

Runner ARM64 di Mac Apple Silicon tidak otomatis menghasilkan artifact AMD64. Gunakan BuildKit/buildx atau runner architecture yang disetujui dan verifikasi manifest platform.

## Contoh Runner Design

```yaml
build:
  tags:
    - <approved-runner-tag>
  variables:
    BUILDKIT_PROGRESS: plain
  script:
    - ./scripts/build-multiarch --platform linux/amd64,linux/arm64
```

Jangan menaruh token runner, registry password, atau kubeconfig pada YAML. Protected/masked variable dan short-lived identity harus disediakan di platform yang disetujui.

## Privileged Build

Docker-in-Docker atau privileged BuildKit dapat memperbesar risiko:

- container escape dan host impact;
- cache poisoning;
- credential tersedia pada layer/log;
- network egress tanpa kontrol.

Pilih rootless/buildkit mode bila kompatibel, pin CI image, batasi network, hapus workspace, dan gunakan runner ephemeral untuk workload berisiko.

## Artifact Retention

Tetapkan:

- artifact apa yang dibutuhkan downstream;
- expiry dan retention exception;
- siapa yang boleh membaca;
- redaction sebelum upload;
- checksum/reference yang dapat diaudit;
- apa yang harus dihapus setelah incident.

Jangan meng-upload:

- `kubeconfig`;
- OpenTofu state atau plan mentah;
- decrypted SOPS/Secret;
- private key/PAT;
- raw `helm get all`;
- log yang memuat environment variable sensitif.

## Cache

Cache mempercepat dependency download tetapi bukan source of truth. Gunakan key yang memasukkan lockfile/architecture, batasi scope, dan jangan memasukkan secret ke cache. Jika cache dapat dipengaruhi fork atau untrusted MR, perlakukan sebagai input tidak terpercaya.

## Environment dan Protected Deployment

```yaml
deploy-staging:
  stage: publish
  environment:
    name: staging
  when: manual
  resource_group: staging

deploy-production:
  stage: publish
  environment:
    name: production
  when: manual
  resource_group: production
```

Environment protected branch, approval, freeze window, access recovery, dan rollback harus dikonfigurasi di GitLab, bukan hanya ditulis di README.

## Acceptance Criteria

- [ ] Architecture runner dan build platform terdokumentasi.
- [ ] Privilege, network, workspace, dan lifecycle runner direview.
- [ ] Artifact dan cache memiliki retention, permission, dan redaction.
- [ ] Staging/prod memiliki protected gate dan serialization.

## Troubleshooting

- Runner offline: bedakan registration, tags, capacity, network, dan executor.
- Build platform salah: inspect manifest secara read-only dan bandingkan dengan required node architecture.
- Cache stale: bump key berdasarkan lockfile atau clear melalui prosedur platform.
- Artifact bocor: stop downstream jobs, restrict access, revoke affected credential, dan simpan hanya evidence redacted.

## Kaitan

- [Fase 1 — OrbStack](../../fase-1-container-orbstack/README.md)
- [Modul 6.1](README.md)
- [LAB-01](lab/LAB-01-ci-lint-test-build-push.md)

## Catatan SRE

Runner adalah production infrastructure. Patch, capacity, disk pressure, queue, time synchronization, dan audit access-nya memengaruhi seluruh software supply chain.
