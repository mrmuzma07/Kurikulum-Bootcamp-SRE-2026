# 03 — Chart Repository dan OCI di GitLab

## Tujuan

Memahami package chart, SemVer, provenance, dan alur OCI registry tanpa menaruh credential pada repository atau shell history.

## Chart Package

```bash
helm lint ./internal-app
helm package ./internal-app \
  --version <chart-semver> \
  --app-version <approved-app-version> \
  --destination <approved-package-dir>
```

Package adalah archive; package berhasil tidak membuktikan chart dapat di-install. Simpan checksum atau reference artifact yang tidak mengandung credential dan hubungkan dengan commit/chart version.

## Repository Index versus OCI

Repository klasik memakai index dan URL chart. OCI menyimpan chart sebagai artifact registry dan memakai reference seperti:

```text
oci://<gitlab-oci-registry>/<approved-project>/charts/internal-app
```

Bedakan:

- chart source di Git;
- package `.tgz`;
- registry artifact;
- credential mechanism protected;
- cluster release yang memakai chart version tertentu.

GitLab registry auth harus berasal dari mekanisme protected CI atau secret manager. Jangan menulis username, token, PAT, atau command login dengan nilai nyata.

## Pull dan Push Workflow Konseptual

```bash
helm registry login <gitlab-oci-registry> \
  --username <protected-registry-user> \
  --password-stdin <approved-protected-input>
helm push <chart-package-path> oci://<gitlab-oci-registry>/<approved-project>/charts
helm pull oci://<gitlab-oci-registry>/<approved-project>/charts/internal-app \
  --version <chart-semver>
```

Command di atas adalah workflow placeholder, bukan bukti login/push/pull berhasil. Jangan memasukkan credential pada command line, file values, README, atau log.

## Versioning dan Provenance

- Naikkan chart `version` ketika template/default/schema/dependency berubah.
- Naikkan `appVersion` mengikuti release aplikasi, tetapi tetap gunakan image tag/digest yang dapat ditelusuri.
- Pin dependency dan review `Chart.lock`.
- Hubungkan artifact dengan commit SHA, chart version, app version, dan approval.
- Promosi antar-environment harus memakai artifact yang sama atau reference immutable yang jelas; jangan rebuild diam-diam.

## Artifact Hygiene

Jangan menjadikan raw output berikut artifact umum:

- `helm get all`;
- rendered manifest yang memuat Secret;
- registry login output;
- raw CI debug;
- values hasil merge yang mengandung endpoint atau credential.

Buat evidence ringkas: chart version, digest/reference, commit, environment, command class, timestamp, status, dan redacted failure class.

## Static Registry Review

Jika registry belum tersedia, review:

1. chart name dan SemVer;
2. OCI path dan ownership;
3. dependency provenance;
4. protected secret injection design;
5. retention/immutability policy;
6. promotion approval;
7. rollback reference.

Jangan menyatakan “terpush ke GitLab” hanya karena package lokal berhasil dibuat.

## Troubleshooting

- `not found`: periksa OCI path, chart name, version, dan registry project scope.
- `unauthorized`: hentikan; jangan mencetak token. Periksa protected mechanism dan permission.
- Versi tertimpa: periksa immutable tag policy dan package retention.
- Dependency tidak konsisten: gunakan lock file dan review provenance.

## Acceptance Criteria

- [ ] Package, repository, dan OCI artifact dibedakan.
- [ ] Version/provenance/promotion dapat ditelusuri.
- [ ] Auth menggunakan protected mechanism tanpa credential literal.
- [ ] Push/pull yang belum dijalankan ditandai belum diverifikasi.

## Catatan SRE

Registry adalah bagian dari supply chain. Availability, immutability, provenance, retention, dan recovery chart harus dipikirkan sama seriusnya dengan deployment cluster.

## Kaitan

Image berasal dari [Fase 1](../../fase-1-container-orbstack/README.md); alur GitOps pada Fase 6 akan mengonsumsi artifact chart yang telah dipin dan direview.
