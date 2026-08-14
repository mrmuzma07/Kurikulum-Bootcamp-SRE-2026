# LAB-02 — Drift, Self-Heal, dan ApplicationSet

## Tujuan

Menganalisis drift pada target disposable, menguji policy self-heal secara terkontrol, dan memvalidasi ApplicationSet tanpa memperluas destination.

## Prasyarat dan Guardrail

Gunakan [teori ApplicationSet, secret, drift](../03-applicationset-secrets-drift-selfheal.md). Runtime hanya disposable. Manual drift edit production dilarang.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menguji prune pada resource penting, menghapus namespace luas, atau mencetak decrypted SOPS/Sealed Secret.

## Lane A — Static Simulation

### 1. Model ApplicationSet

Tulis generator cluster dengan selector:

```yaml
generators:
  - clusters:
      selector:
        matchLabels:
          sre.managed: "true"
          sre.environment: <approved-environment-label>
```

Dokumentasikan label source, destination allowlist, namespace ownership, name collision, project, revision pin, dan prosedur menghapus ApplicationSet.

### 2. Model drift

Buat tabel:

| Perubahan | Penyebab mungkin | Tindakan |
|---|---|---|
| replicas berbeda | manual edit/controller | inspect owner, revert atau policy |
| label ditambah webhook | admission mutation | evaluasi ignore field sempit |
| Secret ciphertext berubah | rotation/promotion | review GitOps commit dan key boundary |
| resource hilang dari Git | prune candidate | abort sampai owner/recovery jelas |

### 3. Secret boundary review

Bandingkan External Secrets, Sealed Secrets, dan SOPS + age berdasarkan siapa yang memegang key, kapan dekripsi terjadi, bagaimana rotation/backup/recovery, dan apa yang masuk evidence. Pastikan ciphertext tidak disamakan dengan key management.

## Lane B — Optional Disposable Runtime

1. Verifikasi context, ApplicationSet, cluster labels, project, dan namespace.
2. Pastikan hanya satu resource non-kritis yang menjadi objek uji.
3. Lakukan perubahan manual terkontrol pada resource disposable sesuai approval.
4. Amati `OutOfSync`, diff, self-heal/manual sync, dan health.
5. Jika self-heal tidak diaktifkan, lakukan sync manual setelah review.
6. Uji satu generator ApplicationSet pada label yang sudah disetujui; cek tidak ada cluster lain yang terpilih.
7. Simpan status/revision/time dan diff redacted.

Tanpa execution evidence, `ApplicationSet deployment` dan `drift/self-heal` tetap **belum diverifikasi**.

## Stop Conditions

- selector memilih cluster/namespace di luar lab;
- diff berisi CRD, PVC, Secret plaintext, atau namespace penting;
- self-heal berkonflik dengan controller/incident break-glass;
- prune proposal tidak memiliki owner/recovery;
- key/decryption output muncul pada log;
- Application name collision atau generator tidak deterministic.

## Evidence Template

```text
lab: LAB-02
generator_selector: <redacted-selector>
selected_targets: <approved-count-or-redacted-list>
application_revision: <commit-or-tag>
drift_resource: <non-sensitive-kind-name>
pre_status: <summary>
correction: <manual-sync-or-self-heal-summary>
post_health: <summary>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Selector ApplicationSet bounded dan deterministic.
- [ ] AppProject/namespace/resource ownership direview.
- [ ] Drift cause, correction, ignore policy, self-heal, dan prune dipisahkan.
- [ ] Secret key/decryption boundary dijelaskan tanpa plaintext.
- [ ] Runtime tidak menyentuh production dan evidence redacted.

## Troubleshooting

- Terlalu banyak Application: hentikan generator, audit selector/labels, dan batasi project.
- Self-heal loop: cari controller yang menulis field sama; gunakan ownership/ignore field yang sempit.
- Drift tidak terlihat: periksa resource tracking dan refresh, bukan langsung disable comparison.
- Decrypt gagal: cek key access/controller policy tanpa mencetak isi secret.

## Kaitan

Hubungkan hasil ke [Modul 6.3 promotion](../../modul-6.3-end-to-end-flow/README.md) dan [Fase 7 Observability](../../../fase-7-observability/README.md).
