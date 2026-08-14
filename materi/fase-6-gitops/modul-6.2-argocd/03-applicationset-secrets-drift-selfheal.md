# 03 — ApplicationSet, Secret Boundary, Drift, dan Self-Heal

## Tujuan

Mengelola banyak environment/cluster dengan ApplicationSet serta memahami batas aman secret, drift correction, self-heal, dan prune.

## ApplicationSet dan AppProject

`AppProject` membatasi repository, destination cluster/namespace, dan resource kind yang boleh dikelola. `ApplicationSet` membuat Application dari generator yang terkontrol; generator bukan izin tanpa batas.

Contoh non-secret dengan selector placeholder:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: <approved-applicationset>
  namespace: <argocd-namespace>
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            sre.environment: <approved-environment-label>
            sre.managed: "true"
  template:
    metadata:
      name: '<app-name>-{{name}}'
    spec:
      project: <approved-project>
      source:
        repoURL: <approved-gitops-repository>
        path: apps/<app-name>/overlays/{{metadata.labels.sre.environment}}
        targetRevision: <approved-commit-or-tag>
      destination:
        server: '{{server}}'
        namespace: <approved-namespace>
```

Sebelum mengaktifkan generator, review:

- label cluster dan allowlist destination;
- template name collision;
- namespace ownership;
- repository/path/revision;
- resource kinds dan project policy;
- apakah staging dan production benar-benar terpisah;
- bagaimana menghapus ApplicationSet tanpa menghapus aplikasi yang masih dimiliki.

Jangan memakai selector luas seperti semua cluster tanpa policy. App-of-apps dan self-management ArgoCD harus memiliki bootstrap/recovery path agar tidak membentuk circular dependency.

## Secret Management Boundary

Pilihan umum:

| Pola | Yang ada di Git | Key/decryption boundary |
|---|---|---|
| External Secrets | reference dan policy | external secret manager/controller |
| Sealed Secrets | ciphertext SealedSecret | controller key di cluster, backup/rotation policy |
| SOPS + age | ciphertext dan metadata | age private key di CI/controller/secret manager |

Ciphertext bukan bukti secret aman dengan sendirinya. Tetapkan ownership, rotation, backup, recovery, audit, dan siapa yang dapat mendekripsi.

Jangan commit:

- raw Kubernetes Secret;
- age private key atau Vault password;
- decrypted SOPS file;
- ArgoCD repository/cluster credential;
- PAT, deploy token, kubeconfig, atau private key.

Jangan menggunakan `--set` untuk secret karena nilai dapat masuk shell history atau CI log. Gunakan protected/masked variable, OIDC/workload identity, atau controller integration yang disetujui.

## Drift, Ignore, dan Self-Heal

Drift dapat berasal dari:

- perubahan manual;
- mutating/admission webhook;
- controller status/defaulting;
- dependency atau operator;
- perbedaan field ownership.

Alur aman:

```text
observe OutOfSync → klasifikasikan owner dan penyebab
→ cek diff redacted → pilih revert Git, sync manual, atau policy
→ verifikasi health → catat evidence
```

`selfHeal` dapat mengembalikan field ke desired state, tetapi tidak membalikkan migration database, external API call, PVC/data deletion, atau side effect hook. `ignoreDifferences` harus sempit dan didokumentasikan; jangan dipakai untuk menyembunyikan drift penting.

`prune` dapat menghapus resource yang hilang dari desired state. Aktifkan hanya setelah ownership, finalizer, namespace, CRD, PVC, dan deletion recovery direview.

## Runtime Drift Simulation

Jika tersedia target disposable:

1. Verifikasi context, namespace, Application, dan resource ownership.
2. Ubah satu field non-destruktif pada resource lab.
3. Amati `OutOfSync` dan diff tanpa mencetak Secret.
4. Uji manual sync atau self-heal sesuai policy.
5. Pastikan resource kembali sehat dan catat revision/time/job evidence.
6. Hapus lab melalui prosedur scoped, bukan broad deletion.

Manual edit production dilarang. Tanpa evidence aktual, drift/self-heal/ApplicationSet runtime **belum diverifikasi**.

## Acceptance Criteria

- [ ] ApplicationSet generator dan destination allowlist dapat dijelaskan.
- [ ] AppProject membatasi repository, cluster, namespace, dan resource kind.
- [ ] Secret ciphertext/key/decryption boundary, rotation, backup, dan recovery jelas.
- [ ] Drift dibedakan dari expected controller mutation.
- [ ] Self-heal dan prune memiliki deletion safety.
- [ ] Runtime tanpa evidence dilaporkan **belum diverifikasi**.

## Troubleshooting

- ApplicationSet tidak membuat Application: cek generator selector, permissions, template validation, dan controller logs redacted.
- Application salah cluster: hentikan sync, periksa label/allowlist dan destination policy.
- False drift: identifikasi mutating controller dan pertimbangkan ignore field yang sempit.
- Secret decrypt gagal: bedakan key access, format, policy, dan controller readiness; jangan mencetak plaintext.
- Prune mengusulkan resource penting: abort, review ownership/finalizer, dan jangan confirm.

## Kaitan

Praktikkan [LAB-01 Application sync](lab/LAB-01-argocd-application-sync.md) dan [LAB-02 drift/ApplicationSet](lab/LAB-02-drift-selfheal-applicationset.md). Hubungkan artifact CI dari [Modul 6.1](../modul-6.1-gitlab-ci-cd/README.md) ke promotion [Modul 6.3](../modul-6.3-end-to-end-flow/README.md).
