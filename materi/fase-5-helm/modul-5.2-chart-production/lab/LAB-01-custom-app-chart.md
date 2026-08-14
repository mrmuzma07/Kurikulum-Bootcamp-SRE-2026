# LAB-01 — Custom Application Chart

## Tujuan

Menghasilkan chart aplikasi internal dengan values staging/prod, schema, security baseline, probes, resources, PDB, dan test design.

## Static Lane

1. Tetapkan chart name, chart SemVer, app version, owner, dan `<approved-image-repository>`.
2. Buat Deployment, Service, optional Ingress, ConfigMap non-secret, ServiceAccount, PDB, test Job, helper, dan NOTES.
3. Gunakan image digest placeholder; jangan gunakan `latest`.
4. Tambahkan securityContext non-root bila image mendukung, resource requests/limits, startup/readiness/liveness probe, dan termination behavior.
5. Buat `values.schema.json` yang mewajibkan image repository/digest dan membatasi replica/resource type.
6. Render staging dan production. Review selector stability, capacity, endpoint, PDB, and environment-only differences.
7. Tulis evidence chain dari commit → chart version → values revision → rendered review.

## Runtime Lane Opsional

Jika cluster disposable tersedia, jalankan lint/render terlebih dahulu, lalu mutation setelah approval. Validasi rollout dan Service; jangan menggunakan production context.

## Failure Drill

- schema production tidak memiliki required field;
- selector berubah saat upgrade;
- requests melebihi capacity node;
- readiness path salah;
- PDB terlalu ketat untuk jumlah replica.

## Acceptance Criteria

- [ ] Chart dapat dirender untuk dua environment.
- [ ] Values tidak menyimpan secret.
- [ ] Image immutable dan labels/selectors stabil.
- [ ] Probes, resources, security context, PDB, dan Service direview.
- [ ] Test dan NOTES tidak mencetak credential.

## Catatan SRE

Default chart harus aman ketika operator lupa override; tetapi default tidak boleh menyembunyikan input wajib atau masalah capacity.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
