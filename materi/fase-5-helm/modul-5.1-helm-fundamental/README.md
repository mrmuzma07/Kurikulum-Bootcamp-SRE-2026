# Modul 5.1 — Helm Fundamental

> **Tujuan akhir:** memahami chart dan release, menulis template Go yang aman, merender manifest, serta merancang lifecycle install/upgrade/rollback tanpa mencampurkan static proof dengan runtime proof.

## Capaian Modul

- [ ] Menjelaskan chart, release, values, repository, dan OCI registry.
- [ ] Membaca struktur `Chart.yaml`, `values.yaml`, `templates/`, helper, schema, test, dan `NOTES.txt`.
- [ ] Menggunakan `include`, `required`, `default`, `quote`, `toYaml`, `nindent`, `if`, `with`, dan `range`.
- [ ] Menyusun values contract non-secret dan image immutable.
- [ ] Melakukan lint, template, package, dan rendered-manifest review.
- [ ] Menjelaskan install, upgrade, history, rollback, uninstall, `--set`, multiple values, `--wait`, `--atomic`, dan timeout.
- [ ] Menjelaskan chart repository dan OCI package di GitLab tanpa menyimpan credential.

## Rencana 2 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [Struktur chart dan template](01-struktur-chart-values-template.md), [render/lifecycle](02-render-install-upgrade-rollback.md) | [LAB-01](lab/LAB-01-chart-skeleton-render-lint.md) |
| 2 | [Repository dan OCI](03-repository-chart-oci-gitlab.md) | [LAB-02](lab/LAB-02-release-lifecycle-values.md) + [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 2.1 dan 2.4: Deployment, Service, ConfigMap, Secret boundary, rollout, context safety.
- Fase 1: image repository dan tag semver/multi-arch.
- YAML dan dasar command line.
- Helm hanya diperlukan untuk runtime lane; static lane dapat dilakukan dengan review.

## Cara Belajar

Mulai dari rendered output, bukan dari asumsi bahwa template terlihat benar. Tetapkan values contract, render setiap environment, periksa selector dan security fields, baru susun release runbook. `helm lint` dan `helm template` hanya membuktikan chart dapat diperiksa atau dirender.

## Acceptance Criteria

- [ ] Struktur chart dan values contract terdokumentasi.
- [ ] Template memakai helper, defaults, validation, dan indentation yang konsisten.
- [ ] Image memakai `<immutable-image-digest>` atau reference yang disetujui.
- [ ] Lint/render/package dilakukan atau ditandai belum diverifikasi jika tool tidak tersedia.
- [ ] Lifecycle runbook memverifikasi context, namespace, release, diff, timeout, approval, dan rollback.
- [ ] Tidak ada credential, token, kubeconfig, atau Secret value literal.

## Guardrail

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan memakai `--set` untuk secret. Redact rendered Secret, `helm get all`, history, hook logs, dan CI artifacts. Jangan install/upgrade/rollback/uninstall pada context yang tidak diverifikasi atau cluster production.

## Troubleshooting

- Template gagal: periksa delimiters, values path, tipe schema, `include`, `toYaml`, dan `nindent`.
- YAML hasil render invalid: simpan output redacted, cari indentation/quote/list type, lalu render ulang dengan values minimal.
- Release Pending/timeout: bedakan failure chart, scheduling, image pull, readiness probe, storage, dan dependency eksternal.
- Rollback gagal: periksa revision, CRD, migration, hook, immutable field, PDB, dan data compatibility sebelum tindakan berikutnya.

## Kaitan

- Fase 1 menyediakan image.
- Modul 2.1 menyediakan objek yang ditulis template.
- Modul 2.4 menyediakan rollout dan troubleshooting.
- [Modul 5.2](../modul-5.2-chart-production/README.md) melanjutkan ke custom chart production.
- Fase 6 memakai chart dalam GitOps.

## Catatan SRE

Release yang berhasil dibuat belum tentu workload sehat. Health harus ditentukan dari status Pod, readiness, Service/Ingress, smoke test, telemetry, dan SLO yang relevan.

## Status Runtime

Materi tersedia. Helm binary, chart lint/render, registry push, dan release lifecycle belum diverifikasi tanpa evidence runtime.
