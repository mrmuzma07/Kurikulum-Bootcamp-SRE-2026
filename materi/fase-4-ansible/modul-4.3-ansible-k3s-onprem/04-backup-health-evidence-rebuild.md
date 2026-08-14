# 04 — Backup, Health, Evidence, dan Rebuild

## 1. Backup Boundary

Backup etcd/k3s snapshot, konfigurasi host, aplikasi, PV, database, dan secret memiliki owner serta retention berbeda. Snapshot etcd bukan pengganti backup aplikasi/PV/database. Restore harus diuji pada cluster disposable dan mengikuti prosedur resmi; jangan restore pada cluster aktif sebagai eksperimen.

## 2. Health dan Evidence

Evidence minimum yang tidak memuat secret:

```text
commit SHA → inventory/module version → target environment
→ readiness result → playbook summary redacted → node condition
→ API/workload smoke test → backup reference → approval/change record
```

Jangan menyimpan raw `-vvv`, kubeconfig, token, decrypted Vault content, atau raw diff. Simpan ringkasan, checksum/reference yang tidak sensitif, timestamp, dan scope.

## 3. Rebuild Runbook

```text
verify owner/scope
→ provision metadata
→ readiness gate
→ bootstrap common
→ hardening approved
→ k3s server/agent sequencing
→ health and backup verification
→ application handoff
```

Rebuild bukan berarti menghapus state lalu mencoba ulang. Bedakan host replacement, cluster restore, application restore, dan infrastructure recreation.

## 4. Drift dan Recovery

Ansible dapat mendeteksi sebagian drift konfigurasi; OpenTofu memiliki lifecycle infrastructure. Tentukan owner sebelum remediation. Jika playbook gagal setelah partial change, jangan blind rerun pada semua host; classify state, limit target, dan buat recovery plan.

## Acceptance Checklist

- [ ] Backup boundary dan restore scope jelas.
- [ ] Evidence chain dapat diaudit tanpa raw secret.
- [ ] Rebuild memiliki gate dan ownership.
- [ ] Drift/remediation tidak mencampur state OpenTofu dan host config.

## Catatan SRE

Recovery yang tidak pernah diuji hanyalah asumsi. Latihan restore dan rebuild harus disposable, memiliki RPO/RTO, evidence, dan keputusan berhenti yang eksplisit.

## Kaitan

Gunakan [LAB-02](lab/LAB-02-rolling-patching-readiness.md), lalu kerjakan evaluasi Modul 4.3.
