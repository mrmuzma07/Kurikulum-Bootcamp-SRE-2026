# LAB-01 — Chart Skeleton, Render, dan Lint

## Tujuan

Membuat skeleton chart aplikasi non-secret dan membuktikan batas antara review statis, render, lint, dan runtime.

## Prasyarat

- Modul 5.1 teori dibaca.
- Directory kerja disposable.
- Helm optional untuk runtime lane.
- Tidak memerlukan credential atau cluster untuk static lane.

## Static Lane

1. Buat struktur `Chart.yaml`, `values.yaml`, `values.schema.json`, `templates/`, `_helpers.tpl`, `NOTES.txt`, dan test placeholder.
2. Isi values dengan `<approved-image-repository>`, `<immutable-image-digest>`, replica, Service port, resources, dan probes.
3. Review selector, labels, namespace, securityContext, dan image reference.
4. Tinjau template delimiters, `include`, `toYaml`, `nindent`, `required`, dan `default`.
5. Jalankan bila tersedia:

```bash
helm lint <chart-path>
helm template <release-name> <chart-path> \
  --namespace <approved-namespace> \
  --values <values-file>
```

6. Redact output sebelum disimpan sebagai evidence. Jangan menyimpan rendered Secret atau credential.
7. Catat hasil sebagai `lint/render: verified` hanya bila command benar-benar dijalankan; jika tidak, tulis `belum diverifikasi`.

## Runtime Lane Opsional

Hanya pada namespace dan cluster disposable yang telah diverifikasi:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl get namespace <approved-namespace>
```

Render dan review terlebih dahulu. Jangan install sebagai bagian lab ini kecuali scope mutation, approval, timeout, dan rollback sudah jelas.

## Failure Drill

- values wajib hilang;
- replica bertipe string;
- indentation `nindent` salah;
- image digest memakai `latest`;
- Service selector berbeda dari Pod labels.

Untuk setiap failure, simpan class error dan perbaikan, bukan output mentah yang mungkin sensitif.

## Acceptance Criteria

- [ ] Chart skeleton lengkap dan non-secret.
- [ ] Schema menolak input wajib/tipe salah.
- [ ] Rendered YAML diperiksa secara manual.
- [ ] Lint/render execution dibedakan dari desain static.
- [ ] Tidak ada credential, token, kubeconfig, atau Secret value.

## Catatan SRE

Lint yang hijau hanya mengurangi kesalahan sintaks/chart. Ia tidak membuktikan Pod dapat dijadwalkan atau aplikasi memenuhi SLO.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
