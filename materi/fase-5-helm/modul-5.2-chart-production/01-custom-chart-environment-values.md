# 01 — Custom Chart dan Values per Environment

## Tujuan

Membuat contract chart aplikasi internal yang konsisten antara staging dan production tanpa mengubah invariant keamanan menjadi values bebas.

## Resource Contract

Chart aplikasi biasanya menghasilkan:

- Deployment atau StatefulSet sesuai lifecycle data;
- Service dengan selector stabil;
- Ingress bila routing tersedia;
- ConfigMap untuk konfigurasi non-secret;
- ServiceAccount/securityContext;
- probes, resources, PDB, dan optional NetworkPolicy;
- test Job dan `NOTES.txt`.

Jangan membuat StatefulSet hanya karena aplikasi memakai database eksternal. Tetapkan ownership data, storage, backup, dan restore di luar asumsi chart.

## Values Layering

```text
values.yaml                 # defaults aman dan non-secret
values-staging.yaml         # kapasitas dan endpoint staging
values-prod.yaml            # kapasitas production, tetap tanpa credential
values.schema.json          # tipe, required field, enum, bounds
```

Contoh perbedaan yang dapat direview:

```yaml
# values-staging.yaml
replicaCount: 1
resources:
  requests: {cpu: 100m, memory: 128Mi}
  limits: {cpu: 500m, memory: 512Mi}

# values-prod.yaml
replicaCount: 3
resources:
  requests: {cpu: 500m, memory: 512Mi}
  limits: {cpu: 2, memory: 2Gi}
```

Nilai di atas hanya contoh capacity. Endpoint dan secret harus menggunakan reference yang disetujui, bukan value credential.

## Labels dan Selector

Selector Deployment dan Service harus stabil sepanjang upgrade. Label sebaiknya mencakup:

```yaml
app.kubernetes.io/name: <chart-name>
app.kubernetes.io/instance: "{{ .Release.Name }}"
app.kubernetes.io/version: "<approved-app-version>"
app.kubernetes.io/managed-by: Helm
```

Jangan mengganti selector karena perubahan label kosmetik; selector immutable dapat memaksa replacement dan mengganggu availability.

## Image dan Security

Gunakan repository dan digest immutable:

```gotemplate
image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

Tambahkan secara eksplisit dan review:

- `runAsNonRoot: true` bila image mendukung;
- `allowPrivilegeEscalation: false`;
- `readOnlyRootFilesystem` sesuai aplikasi;
- drop capabilities yang tidak diperlukan;
- ServiceAccount minimal;
- resource requests/limits;
- liveness/readiness/startup probe;
- PDB sesuai replica dan availability policy.

## Ingress dan NetworkPolicy

Ingress memerlukan DNS, TLS, controller, dan path policy. NetworkPolicy memerlukan pemahaman CNI dan dependency; default deny tanpa allowlist dapat memutus health check. Jadikan keputusan ini explicit dan uji pada scope disposable.

## Schema Validation

Schema harus membatasi tipe, range, dan required values. Jangan memakai default berbahaya agar schema “selalu lulus”. Render setiap environment dan bandingkan:

```bash
helm template <release-name> ./internal-app -f values-staging.yaml
helm template <release-name> ./internal-app -f values-prod.yaml
```

Simpan hanya output redacted jika output dijadikan evidence.

## Promotion

Promotion yang dapat diaudit mengikat:

```text
commit SHA → chart version → image digest → values revision
→ rendered diff → approval → target environment → health evidence
```

Jangan mengubah artifact atau image secara diam-diam ketika mempromosikan.

## Troubleshooting

- Staging lulus, prod gagal scheduling: bandingkan requests, node capacity, taint, affinity, dan PDB.
- Service tidak menemukan Pod: periksa selector dan label hasil render.
- Ingress 404/503: bedakan route, controller, Service port, readiness, dan DNS.
- SecurityContext gagal: cocokkan user, filesystem, port, volume, dan image behavior.

## Acceptance Criteria

- [ ] Values staging/prod terpisah dan non-secret.
- [ ] Schema menangkap required/type/range failure.
- [ ] Selector stabil dan image immutable.
- [ ] Security context, probes, resources, PDB, dan ingress boundary direview.
- [ ] Promotion chain dapat ditelusuri.

## Catatan SRE

Environment values bukan sekadar angka berbeda. Setiap override harus menjelaskan failure mode yang dihindari dan capacity yang tersedia.

## Kaitan

Lanjutkan ke [hooks, tests, dan NOTES](02-hooks-tests-notes-upgrade.md) serta gunakan image contract dari Fase 1.
