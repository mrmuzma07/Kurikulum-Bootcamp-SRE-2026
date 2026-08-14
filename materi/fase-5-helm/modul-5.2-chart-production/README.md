# Modul 5.2 — Chart untuk Production

> **Tujuan akhir:** menghasilkan chart aplikasi internal yang dapat dipromosikan antar-environment dengan schema, tests, hooks yang terkendali, image immutable, dan strategi upgrade/rollback yang dapat diaudit.

## Capaian Modul

- [ ] Membuat chart custom dengan Deployment, Service, Ingress, configuration, probes, resources, security context, dan PDB.
- [ ] Memisahkan values staging dan production tanpa secret plain text.
- [ ] Menambahkan `values.schema.json`, helper, `NOTES.txt`, dan `helm test`.
- [ ] Menjelaskan hook lifecycle, weight, deletion policy, failure behavior, dan migration boundary.
- [ ] Menyusun dependency pinning, `Chart.lock`, SemVer, dan OCI promotion.
- [ ] Merancang upgrade/rollback dengan rendered diff, history, PDB, CRD, migration, timeout, dan stop condition.
- [ ] Menyiapkan chart observability tanpa mengklaim stack aktif tanpa evidence.

## Rencana 3 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [Custom chart dan values](01-custom-chart-environment-values.md) | [LAB-01](lab/LAB-01-custom-app-chart.md) |
| 2 | [Hooks, tests, NOTES](02-hooks-tests-notes-upgrade.md) | [LAB-02](lab/LAB-02-observability-oci-test-rollback.md) |
| 3 | [Security dan reliability](03-production-chart-security-reliability.md) | [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 5.1 selesai.
- Pemahaman Deployment/Service/Ingress, probes, resources, PDB, dan rollout.
- Fase 4 readiness k3s untuk runtime lane.
- Akses registry/cluster hanya diperlukan bila runtime disposable disetujui.

## Acceptance Criteria

- [ ] Chart custom memiliki version SemVer dan app version yang dapat ditelusuri.
- [ ] Staging/prod values dirender terpisah dan divalidasi schema.
- [ ] Selector stabil, image immutable, resources/probes/security context terdokumentasi.
- [ ] Hooks tidak menyembunyikan failure dan tidak membawa credential.
- [ ] `helm test`, NOTES, rendered diff, upgrade, dan rollback memiliki acceptance/evidence design.
- [ ] Dependency dan OCI artifact memiliki provenance/version review.
- [ ] Observability footprint dan storage/retention concern dibahas.

## Guardrail

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

`values-prod.yaml` tidak boleh menjadi secret store. Jangan memakai `--set` untuk secret. Hook migration, CRD, uninstall, rollback, dan observability deployment membutuhkan scope, approval, backup/recovery, dan stop condition.

## Troubleshooting

- Schema menolak values: periksa required field, tipe, enum, dan perbedaan staging/prod.
- Upgrade gagal: periksa rendered diff, immutable field, migration compatibility, CRD lifecycle, hook logs yang sudah diredáksi, dan revision history.
- `helm test` gagal: bedakan test Pod/Job, readiness, dependency eksternal, dan application/SLO health.
- Chart terlalu berat: ukur requests/limits, storage, retention, replicas, dan resource footprint sebelum observability deployment.

## Kaitan

- [Modul 5.1](../modul-5.1-helm-fundamental/README.md) menyediakan chart/lifecycle foundation.
- Modul 2.4 menyediakan rollout, PDB, backup, dan troubleshooting.
- Fase 4 menyediakan cluster readiness.
- Fase 6 membawa chart ke ArgoCD/GitOps.
- Fase 7 membawa chart observability dan telemetry evidence.

## Catatan SRE

Promotion bukan sekadar mengganti values. Setiap environment harus mempertahankan invariants yang aman, sementara capacity, endpoint, replica, storage, dan policy harus divalidasi sebagai perubahan terpisah.

## Status Runtime

Custom chart, OCI push, `helm test`, observability deployment, upgrade, rollback, dan promotion belum diverifikasi tanpa execution evidence.
