# LAB-02 — Observability, OCI, Test, dan Rollback

## Tujuan

Menyusun pipeline review chart production dari package sampai test/rollback tanpa menganggap artifact lokal atau desain sebagai execution evidence.

## Static Lane

1. Review chart lint, schema, rendered output, dependency, `Chart.lock`, SemVer, dan provenance.
2. Rancang package dan OCI reference:

```bash
helm package <chart-path> --version <chart-semver> --destination <approved-package-dir>
helm push <chart-package-path> oci://<gitlab-oci-registry>/<approved-project>/charts
```

3. Tetapkan protected registry authentication tanpa menulis token/password ke command line atau repository.
4. Rancang `helm test` untuk Service/health path dan NOTES yang hanya memuat instruksi safe.
5. Review observability footprint: replicas, resources, storage, retention, labels, scrape path, and upgrade behavior.
6. Tulis rollback matrix untuk failed hook/test, readiness timeout, error rate, resource saturation, and migration incompatibility.
7. Tentukan evidence minimum: chart/app version, digest, commit, environment, namespace, revision, test status, health summary, approval.

## Runtime Lane Opsional

Hanya pada registry dan cluster disposable/approved:

```text
preflight context → lint/template → package → protected push
→ install/upgrade --wait → helm test → health evidence
→ controlled rollback or uninstall
```

Setiap command harus memiliki scope, approval, timeout, access recovery, dan redacted evidence. Jangan login/push atau deploy observability tanpa mekanisme credential yang disetujui.

## Failure Drill

- OCI unauthorized;
- package version conflict;
- test Job gagal;
- hook timeout;
- rollback terhalang migration;
- observability stack kekurangan storage/resource.

## Acceptance Criteria

- [ ] OCI package workflow dan provenance dapat diaudit.
- [ ] `helm test`, NOTES, health gate, dan rollback memiliki desain evidence.
- [ ] Observability dependency dan resource footprint direview.
- [ ] Tidak ada registry token, kubeconfig, Secret value, atau raw release output.
- [ ] Push/test/rollback yang tidak dijalankan diberi status belum diverifikasi.

## Catatan SRE

Artifact yang tersedia di registry belum berarti dapat dipromosikan. Promotion membutuhkan compatibility, target readiness, approval, dan health evidence.
