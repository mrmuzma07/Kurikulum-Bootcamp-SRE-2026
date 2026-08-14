# 02 — Hooks, Tests, NOTES, dan Upgrade

## Tujuan

Menggunakan hook dan test sebagai mekanisme lifecycle yang terlihat, terbatas, dan dapat dipulihkan—bukan sebagai tempat menyembunyikan operasi berbahaya.

## Hooks

Hook memakai annotation, misalnya:

```yaml
metadata:
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "10"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

Jenis hook umum: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, dan `pre-delete`. Review:

- apakah operasi idempotent;
- apakah hook memiliki timeout dan cleanup;
- apakah failure menghentikan release;
- apakah hook mengubah data atau external system;
- apakah ordering/weight dapat diprediksi;
- apakah log hook mengandung data sensitif.

Migration dan backup hook bukan pengganti migration runbook. Jika migration mengubah schema, compatibility dengan old/new application harus diuji terpisah.

## `helm test`

Test chart dapat berupa Pod/Job dengan annotation:

```yaml
metadata:
  annotations:
    "helm.sh/hook": test
```

Test sebaiknya memeriksa jalur yang aman dan terbatas, misalnya koneksi Service atau endpoint health. Test lulus tidak otomatis membuktikan load behavior, durability, security, atau SLO.

Contoh workflow:

```bash
helm test <release-name> \
  --namespace <approved-namespace> \
  --logs
```

`--logs` dapat menampilkan data sensitif. Simpan hanya ringkasan yang diredáksi.

## `NOTES.txt`

Notes membantu operator memahami endpoint internal, command read-only, atau next step. Jangan mencetak token, kubeconfig, password, atau Secret. Gunakan placeholder dan pesan context-safe.

```gotemplate
Release {{ .Release.Name }} terpasang pada namespace {{ .Release.Namespace }}.
Validasi readiness dengan context yang telah diverifikasi.
Endpoint publik: <approved-endpoint-reference>
```

## Upgrade Safety

Sebelum upgrade:

1. verifikasi chart version dan image digest;
2. render staging/target values;
3. review diff dan immutable field;
4. cek PDB, replica, capacity, storage, dependency, dan maintenance window;
5. tentukan backup/restore serta rollback decision;
6. cek compatibility migration/CRD/hook;
7. minta approval sesuai environment.

Gunakan `--wait`, `--timeout`, dan bila sesuai `--atomic`, tetapi dokumentasikan trade-off. `--atomic` tidak membatalkan external side effect atau perubahan data.

## CRD dan Dependency

CRD memiliki lifecycle berbeda dari resource biasa. Chart upgrade/uninstall tidak selalu menghapus atau mengubah CRD seperti yang diharapkan. Review ownership dan compatibility sebelum release.

Dependency harus dipin dan lock file direview. Jangan mengandalkan repository mutable atau dependency tanpa provenance.

## Failure Handling

Jika hook/test gagal:

1. hentikan promotion;
2. catat chart/release revision dan failure class tanpa secret;
3. cek status, events, Pod/Job logs yang diredáksi;
4. tentukan apakah release perlu retry, rollback, atau manual recovery;
5. jangan menghapus bukti atau menjalankan uninstall sebagai shortcut.

## Acceptance Criteria

- [ ] Hook memiliki purpose, weight, deletion policy, timeout, dan failure behavior.
- [ ] `helm test` memiliki target, evidence, dan batasan.
- [ ] NOTES tidak mencetak credential.
- [ ] Upgrade runbook memiliki diff, migration/CRD review, PDB, backup, approval, dan rollback.

## Catatan SRE

Hook adalah code yang berjalan pada titik paling sensitif dalam lifecycle. Semakin besar side effect, semakin kuat kebutuhan akan idempotency, observability, dan recovery.

## Kaitan

Gunakan operasi dari [Modul 2.4](../../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md) dan lanjutkan ke [security/reliability chart](03-production-chart-security-reliability.md).
