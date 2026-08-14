# 02 — Render dan Lifecycle Release

## Tujuan

Menyusun alur lifecycle Helm dengan guardrail context, namespace, diff, readiness, history, dan rollback.

## Render Sebelum Mutation

Urutan aman untuk review:

```text
values contract → helm lint → helm template
→ rendered YAML review → kubectl client-side review bila context aman
→ approval → install/upgrade terbatas
```

Contoh render multi-environment:

```bash
helm template <release-name> ./internal-app \
  --namespace <approved-namespace> \
  --values values.yaml \
  --values values-staging.yaml > <redacted-render-output>
```

Output yang memuat Secret, endpoint internal, atau metadata sensitif tidak boleh di-commit atau dijadikan artifact biasa.

## Install

```bash
kubectl config current-context
kubectl config get-contexts
kubectl get namespace <approved-namespace>

helm install <release-name> ./internal-app \
  --namespace <approved-namespace> \
  --create-namespace \
  --values values-staging.yaml \
  --wait \
  --timeout <approved-timeout>
```

`--wait` menunggu kondisi resource tertentu, bukan jaminan SLO. `--create-namespace` harus tetap berada pada scope yang disetujui. Gunakan context flag atau environment wrapper yang dapat diaudit.

## Upgrade

```bash
helm upgrade <release-name> ./internal-app \
  --namespace <approved-namespace> \
  --values values-staging.yaml \
  --wait \
  --timeout <approved-timeout> \
  --atomic
```

`--atomic` dapat melakukan rollback ketika upgrade gagal, tetapi tidak menyelesaikan migration yang sudah mengubah data, external side effect, atau CRD compatibility. Review rendered diff dan backup decision sebelum upgrade.

Hindari `--force` sebagai shortcut. Immutable field, selector, CRD, migration, dan hook membutuhkan runbook tersendiri.

## History dan Status

```bash
helm status <release-name> --namespace <approved-namespace>
helm history <release-name> --namespace <approved-namespace>
helm get values <release-name> --namespace <approved-namespace> --all
```

`helm get` dapat menampilkan konfigurasi sensitif. Simpan hanya ringkasan yang telah diredáksi, bukan output mentah.

## Rollback dan Uninstall

```bash
helm rollback <release-name> <approved-revision> \
  --namespace <approved-namespace> \
  --wait \
  --timeout <approved-timeout>

helm uninstall <release-name> --namespace <approved-namespace>
```

Rollback hanya mengembalikan release resources yang dikelola Helm; tidak otomatis mengembalikan database schema, PV data, external API, atau perubahan yang dilakukan hook. Uninstall pada cluster disposable membutuhkan verifikasi release dan namespace agar tidak salah target.

## Values Precedence

Secara konseptual, nilai dapat berasal dari chart defaults, beberapa `-f` files, dan `--set`; sumber yang lebih akhir dapat menimpa nilai sebelumnya. Tetapkan urutan yang eksplisit dan review hasil akhirnya. Jangan menaruh secret pada `--set` atau file values biasa.

## Health Gate

Setelah release, validasi secara terpisah:

```bash
kubectl -n <approved-namespace> get deploy,pod,svc
kubectl -n <approved-namespace> rollout status deploy/<approved-deployment> \
  --timeout=<approved-timeout>
kubectl -n <approved-namespace> get events --sort-by=.lastTimestamp
```

Health gate harus mencakup readiness, image pull, Service/Ingress, dependency, smoke test, dan telemetry yang relevan. `helm status: deployed` bukan application SLO.

## Stop Conditions

Hentikan workflow jika:

- context/namespace/release tidak cocok;
- rendered diff menyentuh resource di luar scope;
- PDB/replica/quorum tidak memenuhi maintenance policy;
- migration/CRD/hook tidak memiliki compatibility plan;
- image digest atau chart version tidak dapat ditelusuri;
- health gate `not-verified` atau failure evidence belum diklasifikasi.

## Troubleshooting

- Install timeout: kumpulkan status dan events redacted; bedakan scheduling, image pull, probe, storage, dan dependency.
- Upgrade menghasilkan revision baru tetapi app gagal: cek rollout, logs redacted, events, dan compatibility; jangan langsung menghapus release.
- Rollback tidak cukup: periksa state data, CRD, hook, dan immutable field.
- Namespace salah: stop; verifikasi context dan release sebelum tindakan lanjutan.

## Acceptance Criteria

- [ ] Runbook memisahkan render dari mutation.
- [ ] Install/upgrade/rollback memiliki context, namespace, timeout, approval, dan stop condition.
- [ ] Health gate tidak disamakan dengan status Helm.
- [ ] Output release dan events diredáksi.

## Catatan SRE

Rollback adalah perubahan produksi tersendiri. Persiapkan sebelum upgrade, bukan setelah incident dimulai.

## Kaitan

Gunakan guardrail dari [Modul 2.4](../../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md) dan lanjutkan ke [chart repository/OCI](03-repository-chart-oci-gitlab.md).
