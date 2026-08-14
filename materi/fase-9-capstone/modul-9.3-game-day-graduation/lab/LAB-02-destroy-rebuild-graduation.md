# LAB-02 — Destroy, Rebuild, dan Graduation Review

> **Target:** membuktikan platform disposable dapat direbuild dari repository, lalu menilai readiness dengan evidence packet dan known gaps.

## Mode Lab

Static lane mensimulasikan destroy/rebuild contract, evidence matrix, RPO/RTO, postmortem, dan rubric. Runtime lane hanya target disposable yang telah diverifikasi dan disetujui. Jangan memakai production atau target yang menyimpan data penting.

## Prasyarat

- LAB-01 selesai atau tabletop equivalent.
- OpenTofu/Ansible/k3s handoff approved.
- GitOps/observability/backup procedures tersedia.
- Target allowlist, plan/diff review, backup, recovery path, maintenance window, access recovery, and cleanup tersedia.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Destroy/rebuild hanya disposable. Jangan memakai `-auto-approve` sebagai default, `kubectl delete -A`, cluster reset aktif, raw state/plan, kubeconfig, token, credential, backup archive, atau decrypted secret. Jangan menyebut platform ready hanya karena build selesai.

## Evidence Contract

```yaml
lab: LAB-02
capstone_revision: <revision>
target_allowlist: [<disposable-target>]
preflight: <summary>
approval_ref: <approval-id>
before_state: <redacted-summary>
destroy: <summary>
rebuild: <revision-and-summary>
bootstrap_handoff: <summary>
delivery_observability: <summary>
backup_restore: <summary>
recovery_elapsed: <measured-or-unknown>
rpo_rto: <measured-or-unknown>
known_gaps: [<gap>]
graduation: <ready-conditional-not-ready>
```

## Langkah

### 1. Freeze dan Evidence Baseline

Freeze perubahan yang tidak terkait. Capture revision, desired topology, as-built summary, app digest, GitOps/Argo revision, telemetry/SLO, alert route, backup IDs, and access recovery reference. Semua output harus summary/redacted.

### 2. Scope Review

Cocokkan target dengan allowlist dan workspace. Review destroy plan/diff, dependencies, storage, network, state lock, backup restore, RTO objective, stop condition, and cleanup. Peer/mentor menyetujui sebelum action.

### 3. Destroy Disposable Target

Jalankan prosedur repository tanpa auto approval default. Monitor action dan stop bila resource di luar scope muncul. Setelah selesai, verify cleanup counts and no shared resource impact. Jangan menghapus resource secara manual untuk menutupi drift.

### 4. Rebuild dari Repository

Build VM/network/storage metadata, bootstrap Ansible scoped/serial, lalu handoff k3s. Ikuti LAB-02 Modul 9.1. Jangan menyisipkan secret lewat variable, shell history, `--set`, atau file commit.

### 5. Rehydrate Delivery dan Observability

Gunakan GitOps revision yang direview untuk deploy app. Verifikasi digest, Argo revision, rollout, MetalLB/Ingress traffic, metrics/logs/traces, dashboards, five alert contracts/notification, and SLO. Status lokal tidak boleh menggantikan evidence end-to-end.

### 6. Restore dan Measure

Pada target restore yang disetujui, validasi object/PV/application consistency, etcd/control-plane procedure, endpoint, telemetry, and post-restore SLO. Ukur recovery elapsed time dan RPO/RTO hanya dengan clock/evidence yang jelas. Cleanup restore target.

### 7. Graduation Review

Panel mengisi domain matrix:

| Domain | Pass/Fail/Unknown | Evidence | Gap owner | Decision |
|---|---|---|---|---|
| Architecture | <status> | <ref> | <role> | <note> |
| Rebuild | <status> | <ref> | <role> | <note> |
| Delivery | <status> | <ref> | <role> | <note> |
| Observability | <status> | <ref> | <role> | <note> |
| Recovery | <status> | <ref> | <role> | <note> |
| Game Day/operations | <status> | <ref> | <role> | <note> |

`ready` membutuhkan seluruh evidence wajib dan risk yang dapat diterima. Jika evidence wajib hilang atau runtime belum dijalankan, pilih `conditional` atau `not ready`.

### 8. Action Verification

Setiap known gap memiliki owner, due date UTC, risk/expiry, dan verification test. Graduation packet tidak selesai ketika action item baru dibuat; review lanjutan harus membuktikan action tersebut.

## Acceptance Criteria

- [ ] Destroy hanya pada disposable target dengan allowlist, plan/diff, approval, backup, recovery, dan cleanup.
- [ ] Rebuild dimulai dari repository dan melewati OpenTofu → Ansible → k3s handoff.
- [ ] Delivery, MetalLB/Ingress, telemetry, alerts, backup/restore, SLO, RPO/RTO, dan Game Day evidence dipisahkan.
- [ ] Evidence packet lengkap, redacted, memiliki revision/owner/UTC.
- [ ] Graduation memakai `ready`/`conditional`/`not ready` secara jujur.
- [ ] Known gaps memiliki owner, due date, dan verification.

## Troubleshooting

| Gejala | Tindakan |
|---|---|
| Destroy plan berubah saat approval | Stop; generate/review ulang; approval lama tidak berlaku otomatis |
| Rebuild host selesai tetapi app tidak pulih | Pisahkan readiness, GitOps, storage, dependency, telemetry, dan SLO gate |
| Backup ada tetapi restore unknown | Tandai recovery belum terbukti; jalankan isolated restore sesuai approval |
| Evidence hilang karena raw output dibuang | Gunakan summary yang aman dan rerun read-only; jangan menyimpan raw credential |
| Tekanan untuk memilih `ready` | Kembalikan ke evidence matrix; missing mandatory evidence berarti conditional/not ready |

## Lanjut

Fase9 selesai secara dokumentasi setelah [evaluasi Modul 9.3](../evaluasi/latihan.md) dan kuis lulus. Runtime graduation tetap bergantung pada evidence aktual.
