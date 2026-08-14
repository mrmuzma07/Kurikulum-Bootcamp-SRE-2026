# LAB-01 — ArgoCD Application dan Sync

## Tujuan

Membuat desain Application yang bounded, memvalidasi source/destination, lalu memahami perbedaan sync, health, rollout, dan SLO.

## Prasyarat dan Guardrail

- [Modul 6.2](../README.md) dan [teori Application](../02-argocd-install-application-sync.md).
- Cluster k3s/ArgoCD disposable bila memilih runtime.
- Context, namespace, project, access recovery, dan rollback harus diverifikasi.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menaruh repository credential, ArgoCD admin password, kubeconfig, rendered Secret, atau private key pada YAML/evidence. Jangan sync ke production atau mengaktifkan prune tanpa approval.

## Lane A — Static Simulation

### 1. Review source dan destination

Gunakan contoh:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <application-name>
  namespace: <argocd-namespace>
spec:
  project: <approved-project>
  source:
    repoURL: <approved-gitops-repository>
    path: apps/internal-app/overlays/staging
    targetRevision: <approved-commit-or-tag>
  destination:
    name: <approved-cluster-name>
    namespace: <approved-namespace>
  syncPolicy:
    retry:
      limit: <approved-retry-limit>
```

Isi checklist:

- repo/path/revision benar;
- project allowlist cocok;
- cluster dan namespace bounded;
- chart values tidak memuat secret plaintext;
- image reference immutable;
- sync manual/automated memiliki alasan;
- prune/self-heal decision memiliki owner;
- health check dan rollout timeout ditentukan.

### 2. Simulasikan state transition

Buat tabel:

```text
Application created → Missing/Unknown → OutOfSync
→ sync requested → Synced + Progressing
→ rollout complete → Healthy
```

Untuk setiap state, tentukan signal, tindakan operator, dan evidence yang disimpan. Jelaskan mengapa `Synced` bukan berarti endpoint sehat dan `Healthy` bukan SLO.

### 3. Rancang health gate

Hubungkan:

```text
Argo revision → Deployment rollout → Pod readiness/events
→ Service/API smoke test → metrics/logs/traces → SLO decision
```

Semua log dan output harus redacted. Jika Fase 7 belum tersedia, catat telemetry gate sebagai dependency, bukan klaim lulus.

## Lane B — Optional Disposable Runtime

1. Jalankan `kubectl config get-contexts` dan verifikasi context/namespace secara read-only.
2. Pastikan ArgoCD installation dan project memiliki evidence atau tandai belum tersedia.
3. Apply Application hanya setelah review/approval pada target disposable.
4. Amati sync status, revision, resource tree, rollout, dan health.
5. Jalankan smoke test non-destructive pada endpoint lab.
6. Simpan Application name, source revision, target, timestamps, status, dan output redacted.
7. Revert GitOps commit atau delete resource melalui scope prosedur lab; jangan gunakan `kubectl delete -A`.

Jika salah satu execution evidence tidak tersedia, tulis `ArgoCD sync: belum diverifikasi`.

## Stop Conditions

- context kosong atau target tidak cocok;
- project destination terlalu luas;
- sync proposal berisi resource di luar ownership;
- prune mengusulkan PVC/CRD/namespace penting;
- credential muncul pada log;
- rollout stuck atau health unknown;
- rollback/access recovery belum ada.

## Evidence Template

```text
lab: LAB-01
application: <application-name>
source_revision: <commit-or-tag>
target: <cluster-namespace>
sync_status: <redacted-status>
health_status: <redacted-status>
rollout: <summary>
smoke_test: <summary>
slo: <not-measured-or-reference>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] Application source/project/destination tervalidasi.
- [ ] State transition dan failure classification lengkap.
- [ ] Sync, rollout, health, smoke test, telemetry, dan SLO dibedakan.
- [ ] Runtime hanya disposable dan memiliki evidence redacted.
- [ ] Tidak ada secret atau destructive broad command.

## Troubleshooting

- `InvalidSpec`: cek YAML, path, revision, project, dan repo allowlist.
- `OutOfSync`: review diff dan owner; jangan langsung mengaktifkan self-heal.
- `Progressing`: periksa scheduling, image pull, readiness, dependency, dan events.
- `Degraded`: bedakan aplikasi, workload, service, ingress, dan telemetry failure.

## Kaitan

Lanjutkan ke [LAB-02](LAB-02-drift-selfheal-applicationset.md) dan materi [ApplicationSet/secret/drift](../03-applicationset-secrets-drift-selfheal.md).
