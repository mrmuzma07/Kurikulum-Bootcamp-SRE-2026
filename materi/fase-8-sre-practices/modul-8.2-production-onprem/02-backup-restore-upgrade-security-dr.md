# 02 — Backup, Restore, Upgrade, Security, dan Disaster Recovery

## Tujuan

Merancang recovery yang dapat diuji, bukan sekadar menyimpan artifact atau melihat status `Completed`.

## 1. Backup Boundary

Pisahkan tiga kategori:

1. **Kubernetes objects:** namespace, workload, service, config reference, RBAC, policy, dan metadata yang disetujui.
2. **Persistent/application data:** PV, database, queue, object data, encryption metadata, dan consistency point.
3. **Cluster state:** etcd snapshot untuk state k3s/control plane sesuai prosedur resmi.

Velero dapat membantu object/PV workflow, tetapi provider, snapshot capability, application quiesce, credentials, encryption, retention, and restore hooks harus direview. etcd snapshot tersedia bukan bukti cluster berhasil dipulihkan. Backup `Completed` bukan bukti aplikasi atau database konsisten.

Backup contract minimal:

```text
scope: <approved-namespace-or-cluster-scope>
owner: <approved-owner>
location: <approved-backup-location-reference>
retention: <approved-retention>
encryption/key-custody: <approved-key-boundary>
consistency: <application-database-consistency-method>
restore-order: <approved-order>
rpo: <approved-duration>
rto: <approved-duration>
validation: <approved-post-restore-check>
last-restore-test: <timestamp-or-not-run>
```

Jangan menulis credential, encryption key, kubeconfig, raw archive, atau raw Velero/etcd output ke repository. Simpan identifier/checksum summary dan access-controlled evidence reference.

## 2. Restore dan RPO/RTO

Restore order umumnya: infrastructure/network metadata → Ansible host/k3s readiness → control-plane/etcd state bila diperlukan → storage/PV/data → Kubernetes objects/Helm/GitOps → secrets melalui approved mechanism → workload → DNS/MetalLB → observability → SLO validation. Urutan aktual harus mengikuti application/database dependency.

- **RPO:** jumlah data maksimum yang boleh hilang, bukan umur artifact semata.
- **RTO:** waktu dari declaration sampai service memenuhi recovery acceptance, bukan waktu command selesai.
- **Backup validity:** dibuktikan melalui restore test, checksum/identifier summary, consistency check, application check, telemetry, dan documented outcome.

Migration, replication lag, encryption key loss, DNS propagation, and external dependency recovery dapat membuat restore object sukses tetapi service tetap gagal.

## 3. Patching dan k3s Upgrade

OS patch via Ansible harus memiliki inventory allowlist, `--limit`, serial/batch strategy, preflight, maintenance window, quorum/PDB/capacity review, access recovery, rollback/recovery, and post-check. `--check` dan `--diff` bukan zero side effect; jangan menjadikan manual SSH sebagai desired-state evidence.

Controlled k3s upgrade:

```text
review release/compatibility → backup/restore readiness → capacity/quorum/PDB review
→ cordon/drain one approved node → upgrade serially → uncordon
→ control-plane/agent/etcd/workload/telemetry post-check
→ stop or recover on first unsafe outcome
```

Jangan melakukan cluster reset atau restore snapshot pada cluster aktif tanpa prosedur resmi dan approval. K3s version change dapat memengaruhi API, CNI, storage, CRD, workload, dan addon.

## 4. Security dan Supply Chain

Review RBAC least privilege, NetworkPolicy default boundary, SSH/bastion, secret rotation boundary, image digest/provenance, Trivy severity, remediation owner, due date, compensating control, dan exception expiry. Trivy exit code hanya menunjukkan hasil scan pada scope/image tersebut, bukan seluruh supply-chain security.

## 5. Disaster Recovery Path

```text
OpenTofu metadata
→ Ansible host/bootstrap/readiness
→ k3s control plane/agents
→ Helm/GitOps desired state
→ restore objects/PV/data according to consistency order
→ DNS/MetalLB/access
→ observability/alerts
→ SLO/error-budget validation
→ postmortem and action verification
```

DR plan harus memiliki dependency owner, access recovery, key custody, backup retention, communication, decision points, stop conditions, and evidence fields.

## Static dan Runtime Lane

Static lane: review backup matrix, restore order, RPO/RTO math, Ansible/k3s plan, RBAC/NetworkPolicy, Trivy policy, exception, dan DR tabletop.

Disposable runtime: scoped Velero/etcd backup/restore, patch, upgrade, atau scan pada target disposable dengan approval, timeout, serial controls, rollback/recovery, cleanup, dan evidence redacted. Tanpa restore application/SLO evidence status tetap **belum diverifikasi**.

## Acceptance Criteria

- [ ] Objects, PV/application data, dan etcd state dipisahkan.
- [ ] Consistency, encryption/key custody, retention, restore order, RPO/RTO, dan validation lengkap.
- [ ] Patching dan k3s upgrade scoped, serial, memiliki quorum/PDB/capacity review dan recovery.
- [ ] RBAC, NetworkPolicy, secret rotation, Trivy remediation/exception jelas.
- [ ] DR path sampai observability/SLO validation tersedia.
