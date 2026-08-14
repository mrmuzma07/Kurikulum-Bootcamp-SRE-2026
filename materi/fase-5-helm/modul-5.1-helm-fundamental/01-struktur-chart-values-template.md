# 01 — Struktur Chart, Values, dan Go-template

## Tujuan

Memahami chart sebagai paket deklaratif yang menghasilkan manifest Kubernetes dari input values yang dapat direview.

## Chart dan Release

- **Chart** adalah source package: metadata, defaults, template, dependency, schema, test, dan dokumentasi.
- **Release** adalah instance chart yang dipasang ke namespace tertentu dengan nama, revision, values, dan lifecycle sendiri.
- **Values** adalah input konfigurasi, bukan tempat credential.
- **Repository/OCI registry** menyimpan package chart; cluster menyimpan hasil release dan objek Kubernetes.

Render chart tidak menghubungi API Kubernetes. Karena itu, hasil render belum membuktikan scheduling, admission, image pull, readiness, atau Service reachable.

## Struktur Minimum

```text
internal-app/
├── Chart.yaml
├── values.yaml
├── values.schema.json
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── pdb.yaml
│   ├── NOTES.txt
│   └── tests/test-connection.yaml
├── charts/
└── crds/
```

Contoh metadata non-secret:

```yaml
apiVersion: v2
name: internal-app
description: Chart aplikasi internal untuk lab
version: 0.1.0
appVersion: "<approved-app-version>"
type: application
```

`version` mengikuti SemVer chart. `appVersion` adalah informasi aplikasi dan tidak menggantikan image digest.

## Values Contract

```yaml
image:
  repository: <approved-image-repository>
  digest: <immutable-image-digest>
replicaCount: 2
service:
  type: ClusterIP
  port: 8080
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
probes:
  path: /ready
```

Defaults harus aman untuk static review. Hindari default `latest`, privilege tinggi, atau resource tanpa batas. Values production harus melewati schema dan review capacity.

## Template dan Helper

```gotemplate
{{- define "internal-app.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

Penggunaan:

```gotemplate
metadata:
  name: {{ include "internal-app.fullname" . }}
  labels:
    {{- include "internal-app.labels" . | nindent 4 }}
```

Pola penting:

- `required` menghentikan render ketika input wajib tidak tersedia.
- `default` memberi nilai fallback non-secret; jangan menggunakannya untuk menutupi konfigurasi wajib.
- `quote` mencegah tipe string dibaca sebagai angka/boolean secara tidak sengaja.
- `toYaml` + `nindent` menjaga nested map tetap valid.
- `with` mengubah context; gunakan `$` bila memerlukan root context.
- `range` menghasilkan list/map; validasi tipe values terlebih dahulu.
- Whitespace trimming (`{{-` dan `-}}`) harus direview bersama output YAML.

## Schema

`values.schema.json` membantu menolak tipe atau field yang salah sebelum install. Schema bukan secret manager dan bukan bukti runtime.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "replicaCount"],
  "properties": {
    "replicaCount": {"type": "integer", "minimum": 1},
    "image": {
      "type": "object",
      "required": ["repository", "digest"],
      "properties": {
        "repository": {"type": "string", "minLength": 1},
        "digest": {"type": "string", "pattern": "^<immutable-image-digest>$|^sha256:[a-f0-9]{64}$"}
      }
    }
  }
}
```

Gunakan pola placeholder untuk materi; jangan menulis digest atau registry credential nyata.

## Validasi Static

```bash
helm lint ./internal-app
helm template <release-name> ./internal-app \
  --namespace <approved-namespace> \
  --values values-staging.yaml
helm package ./internal-app --destination <approved-package-dir>
```

Jika Helm tidak tersedia, lakukan review struktur, delimiters, schema, dan expected rendered output. Jangan menyatakan lint atau package berhasil tanpa output nyata.

## Troubleshooting

- `can't evaluate field`: context `.`, `$`, dan path values tidak sesuai.
- `nil pointer`: nested value belum memiliki default atau schema required.
- YAML parse error: periksa quote, list indentation, `toYaml`, dan `nindent`.
- Name terlalu panjang: gunakan `trunc 63` dan `trimSuffix`.
- Schema gagal: periksa tipe integer/string, required fields, dan values file yang menang.

## Acceptance Criteria

- [ ] Struktur chart dapat digambar dan setiap file memiliki owner.
- [ ] Values contract tidak membawa secret dan memiliki schema.
- [ ] Helper/naming/labels/selectors konsisten.
- [ ] Render review membedakan proof statis dari runtime proof.
- [ ] Image reference bersifat immutable atau placeholder yang jelas.

## Catatan SRE

Template adalah program kecil. Perlakukan perubahan helper, schema, dan default sebagai perubahan code yang membutuhkan review, test, dan versioning.

## Kaitan

Lanjutkan ke [render dan lifecycle release](02-render-install-upgrade-rollback.md) dan gunakan objek dari [Modul 2.1](../../fase-2-kubernetes/modul-2.1-konsep-k3d/README.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
