# Fase 5 — Helm

> **Tujuan fase:** mengemas workload Kubernetes menjadi chart yang dapat dirender, divalidasi, diuji, dipublikasikan, di-deploy, di-upgrade, dan di-rollback secara aman.

## Durasi dan Modul

Minggu 9 — dua modul dengan static review dan runtime disposable lane.

| Modul | Fokus | Status |
|---|---|---|
| 5.1 | Helm Fundamental | ✅ Tersedia |
| 5.2 | Chart untuk Production | ✅ Tersedia |

## Capaian Fase

- [ ] Menjelaskan chart, release, values, template, repository, dan OCI registry.
- [ ] Membuat struktur chart dengan `Chart.yaml`, `values.yaml`, templates, helper, schema, test, dan `NOTES.txt`.
- [ ] Menulis Go-template dengan defaults, validation, `include`, `toYaml`, `nindent`, conditionals, dan loops.
- [ ] Merender dan memeriksa chart menggunakan lint, template, schema, dan rendered-manifest review.
- [ ] Menjalankan lifecycle install, upgrade, history, rollback, dan uninstall pada scope disposable yang diverifikasi.
- [ ] Menyusun custom chart dengan values per environment, image immutable, probes, resources, security context, PDB, dan Service/Ingress.
- [ ] Menjelaskan hooks, `helm test`, `NOTES.txt`, dependency locking, SemVer, OCI publishing, dan promotion.
- [ ] Merancang upgrade/rollback yang memperhatikan CRD, migration, hooks, PDB, quorum, data safety, dan stop condition.
- [ ] Menyiapkan chart observability tanpa mengklaim stack aktif tanpa evidence.

> Lint, render, package, dan desain chart tidak membuktikan API reachability, scheduling, readiness, application correctness, atau production safety.

## Rencana Belajar

| Hari | Materi | Praktik |
|---|---|---|
| 1 | [Modul 5.1](modul-5.1-helm-fundamental/README.md), [struktur chart](modul-5.1-helm-fundamental/01-struktur-chart-values-template.md) | Membaca chart contract dan membuat skeleton |
| 2 | [Lifecycle release](modul-5.1-helm-fundamental/02-render-install-upgrade-rollback.md), [repository OCI](modul-5.1-helm-fundamental/03-repository-chart-oci-gitlab.md) | Render, lint, package, dan lifecycle review |
| 3 | [Modul 5.2](modul-5.2-chart-production/README.md), [custom values](modul-5.2-chart-production/01-custom-chart-environment-values.md) | Menyusun chart aplikasi dengan values staging/prod |
| 4 | [Hooks/tests/NOTES](modul-5.2-chart-production/02-hooks-tests-notes-upgrade.md), [reliability](modul-5.2-chart-production/03-production-chart-security-reliability.md) | Test, upgrade, rollback, dan security review |
| 5 | [LAB-01](modul-5.1-helm-fundamental/lab/LAB-01-chart-skeleton-render-lint.md), [LAB-02](modul-5.1-helm-fundamental/lab/LAB-02-release-lifecycle-values.md) | Static lane atau release disposable |
| 6 | [LAB-03](modul-5.2-chart-production/lab/LAB-01-custom-app-chart.md), [LAB-04](modul-5.2-chart-production/lab/LAB-02-observability-oci-test-rollback.md) | Custom chart, OCI, test, evidence, evaluasi |

## Dua Lane Praktik

### Static lane

```text
chart skeleton → lint/schema review → helm template atau render review
→ rendered YAML inspection → package/version review → release runbook
```

Gunakan lane ini bila `helm`, registry, atau cluster tidak tersedia. Hasil static lane tidak boleh ditulis sebagai bukti install, test, readiness, push, atau rollback.

### Disposable runtime lane

```text
verify helm/kubectl/context/namespace
→ render + lint + diff review → approval
→ install/upgrade --wait --timeout → status/history/health evidence
→ helm test → controlled rollback or uninstall
```

Runtime hanya untuk cluster dan namespace disposable yang scope-nya jelas. Jangan mengoperasikan production atau context aktif yang belum diverifikasi.

## Prasyarat

- Fase 1 untuk image multi-arch dan registry.
- Fase 2, khususnya Modul 2.1 untuk objek Kubernetes dan Modul 2.4 untuk rollout, PDB, context safety, dan troubleshooting.
- Fase 3 untuk provisioning boundary.
- Fase 4 untuk host/k3s readiness.
- Helm dan `kubectl` hanya diperlukan untuk runtime lane; static lane dapat menggunakan review file.
- Tidak diperlukan credential nyata untuk membaca materi atau menyusun chart skeleton.

## Boundary Ownership

| Layer | Tanggung jawab |
|---|---|
| OpenTofu | VM, network, storage, dan metadata non-secret |
| Ansible | OS bootstrap, hardening, readiness, dan konfigurasi host/k3s |
| Kubernetes/k3s | scheduling, service discovery, storage, cluster health |
| Helm | packaging, templating, release history, deployment application |
| GitOps/Fase 6 | pull-based promotion dan reconciliation dari Git |
| Observability/Fase 7 | metrics, logs, traces, dan SLO evidence |

Helm bukan pengganti Kubernetes controller atau GitOps reconciliation. Release sukses hanya menunjukkan operasi Helm selesai; readiness dan SLO tetap perlu dibuktikan terpisah.

## Deliverables

1. Chart aplikasi custom dengan `Chart.yaml`, defaults, schema, templates, helper, test, dan `NOTES.txt`.
2. Values terpisah untuk staging dan production tanpa credential plain text.
3. Bukti desain atau execution `lint`, render, package, dan versioning.
4. Runbook install/upgrade/rollback dengan context, namespace, diff, timeout, approval, dan stop condition.
5. Review hook, test, dependency lock, OCI artifact, image immutability, probes, resources, PDB, dan security context.
6. Evidence chain redacted dari commit/chart version sampai rendered manifest dan health/test outcome.
7. Nilai kuis minimal **80%** pada setiap modul.

## Guardrail Fase

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menulis password, registry credential, PAT, private key, kubeconfig, Kubernetes Secret value, Helm token, raw release secret, raw rendered Secret, atau credential nyata ke repository, values file, README, log, shell history, CI artifact, atau evidence.
- `values.yaml` hanya berisi defaults non-secret. `values-prod.yaml` bukan secret manager.
- Jangan memakai `--set` untuk secret karena command line dapat masuk history dan log; gunakan secret mechanism yang disetujui dan reference non-secret.
- `helm lint`, `helm template`, dan `helm package` bukan bukti runtime health.
- Verifikasi context, namespace, release name, image reference, dan target sebelum install, upgrade, rollback, atau uninstall.
- `--wait`, `--atomic`, dan `--timeout` membantu lifecycle control tetapi tidak menjamin data safety atau application correctness.
- Redact `helm get all`, rendered manifests, failed hook logs, release metadata, dan CI artifacts; redaction tidak boleh menutupi failure.
- Jangan menjalankan `kubectl delete -A`, cluster reset, restore snapshot aktif, atau destructive Helm shortcut.
- OCI push, install, upgrade, rollback, `helm test`, observability deployment, dan promotion berstatus **belum diverifikasi** tanpa execution evidence.

## Kaitan

- [Fase 1 — Container & OrbStack](../fase-1-container-orbstack/README.md) menyediakan image multi-arch dan registry.
- [Modul 2.1 — Konsep Kubernetes](../fase-2-kubernetes/modul-2.1-konsep-k3d/README.md) menyediakan objek yang dirender chart.
- [Modul 2.4 — Operasi](../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md) menyediakan rollout, PDB, context safety, dan troubleshooting.
- [Fase 4 — Ansible](../fase-4-ansible/README.md) menyediakan cluster readiness sebelum Helm.
- [Fase 6 — GitOps](../fase-6-gitops/README.md) menggunakan Helm dalam alur GitLab CI dan ArgoCD; runtime GitOps tetap **belum diverifikasi** tanpa execution evidence.
- [Fase 7 — Observability](../fase-7-observability/README.md) memberi telemetry dan SLO evidence untuk workload yang dikelola chart; runtime stack tetap **belum diverifikasi** tanpa execution evidence.

## Catatan SRE

Chart yang rapi mengurangi variasi deployment, bukan menghapus failure domain. Setiap release harus dapat ditelusuri ke versi chart, values, image immutable, context, namespace, approval, dan evidence health yang sesuai.

## Status Runtime

Status materi: **tersedia**. Status runtime Helm, registry OCI, cluster deployment, `helm test`, observability, upgrade, dan rollback: **belum diverifikasi** sampai preflight dan execution evidence tersedia.
