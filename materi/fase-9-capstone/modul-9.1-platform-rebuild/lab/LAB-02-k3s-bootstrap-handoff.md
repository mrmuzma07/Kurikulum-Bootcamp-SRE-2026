# LAB-02 — k3s Bootstrap dan Handoff Readiness

> **Target:** memvalidasi handoff host dari Ansible ke k3s production simulation, termasuk quorum, worker, storage, MetalLB prerequisite, dan access recovery.

## Mode Lab

Static lane menghasilkan inventory schema, role matrix, readiness checklist, dan simulated evidence. Disposable runtime lane hanya pada cluster yang disetujui; jangan memakai production dan jangan menguji chaos di sini.

## Prasyarat

- [LAB-01](LAB-01-infrastructure-rebuild.md) atau static contract yang disetujui.
- Host metadata dan inventory non-secret dari OpenTofu.
- Ansible control node, maintenance window, serial strategy, rollback/recovery, dan bastion.
- Network, DNS, time sync, storage, and dedicated MetalLB pool contract.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

K3s join token hanya digunakan melalui approved secret mechanism. Jangan mencetak token atau kubeconfig. Jangan menjalankan `k3s server --cluster-reset` atau restore snapshot pada cluster aktif. `--check`/`--diff` bukan jaminan tanpa side effect.

## Evidence Contract

```yaml
lab: LAB-02
revision: <git-revision>
target: <disposable-cluster>
bootstrap_scope: <approved-limit>
quorum_expected: <n>
node_summary: <counts-and-roles>
api_readiness: <pass-fail-unknown>
storage: <pass-fail-unknown>
metallb_prerequisite: <pass-fail-unknown>
access_recovery: <pass-fail-unknown>
known_gaps: [<gap>]
```

## Langkah

### 1. Review Inventory dan Role

Pastikan setiap host memiliki role `server`, `agent`, `infra`, atau supporting role yang jelas. Validasi address, labels, taints, network range, storage attachment, and failure domain tanpa menyimpan private data.

Review bahwa control-plane count mendukung quorum, worker capacity mendukung PDB, dan endpoint API memiliki access recovery. Jangan meneruskan inventory bila owner atau recovery path tidak diketahui.

### 2. Bootstrap Host Secara Serial

Jalankan approved Ansible roles pada `--limit <approved-host-group>` dengan serial strategy. Mulai dari prerequisite OS/access, lalu control plane, kemudian worker. Simpan hanya summary status dan changed/failed counts; jangan menaruh verbose output yang mengandung secret.

Untuk failure, stop pada batch terkait, ambil redacted evidence, dan ikuti rollback/recovery. Jangan melompat langsung ke semua host.

### 3. Validasi Control Plane dan Worker

Read-only checks harus mencakup:

- node role dan readiness;
- API endpoint dan certificate status summary;
- etcd/quorum health summary;
- CNI/CoreDNS dan system workload;
- taint/label/scheduling;
- PDB/capacity/headroom;
- time synchronization dan disk.

Quorum healthy tidak berarti aplikasi sehat. Node `Ready` tidak membuktikan storage, ingress, telemetry, atau SLO.

### 4. Validasi Storage, MetalLB, dan Access

Periksa StorageClass/PV behavior pada disposable namespace, reclaim policy, capacity, and backup boundary. Periksa bahwa servicelb conflict tidak ada, MetalLB pool dedicated, ARP/L2 dependency terdokumentasi, dan Ingress route memiliki traffic test plan.

Validasi access recovery melalui runbook tanpa menyalin kubeconfig. Catat identifier context secara aman, bukan credential value.

### 5. Handoff Approval

Isi tabel berikut:

| Domain | Status | Evidence reference | Owner | Next action |
|---|---|---|---|---|
| Host readiness | pass/fail/unknown | <ref> | <role> | <action> |
| Quorum/API | pass/fail/unknown | <ref> | <role> | <action> |
| Scheduling/capacity | pass/fail/unknown | <ref> | <role> | <action> |
| Storage | pass/fail/unknown | <ref> | <role> | <action> |
| MetalLB/Ingress | pass/fail/unknown | <ref> | <role> | <action> |
| Access recovery | pass/fail/unknown | <ref> | <role> | <action> |

Handoff hanya `pass` bila dependency kritis memiliki evidence. Jika ada unknown, status minimal `conditional`; delivery Modul 9.2 harus menunggu approval yang sesuai.

### 6. Cleanup

Hapus namespace/test workload yang dibuat lab sesuai allowlist. Pastikan cleanup tidak menghapus shared resource. Catat post-cleanup summary dan outstanding resource. Jangan menggunakan mass-delete.

## Acceptance Criteria

- [ ] Inventory role, quorum, failure domain, network, storage, dan access recovery direview.
- [ ] Bootstrap dibatasi dengan `--limit`, serial strategy, approval, dan recovery.
- [ ] Control-plane/API/quorum, worker, CNI/DNS, storage, MetalLB prerequisite, dan capacity dinilai terpisah.
- [ ] Handoff table memiliki status, evidence, owner, dan next action.
- [ ] Tidak ada token/kubeconfig/credential/raw output dalam repository atau evidence.
- [ ] Handoff `pass` hanya setelah runtime evidence lengkap; selain itu tetap **belum diverifikasi** atau `conditional`.

## Troubleshooting

- **API reachable tetapi quorum unknown:** periksa evidence control-plane melalui approved read-only path; jangan menganggap node Ready cukup.
- **Pod Pending:** bedakan capacity, taint, PVC, image, dan dependency; jangan langsung menambah replica.
- **MetalLB pool pending:** review address range, speaker/controller, VLAN/ARP, dan conflict ownership.
- **Access recovery tidak tersedia:** stop; jangan lanjutkan upgrade, drain, destroy, atau restore.

## Lanjut

Gunakan handoff yang disetujui untuk [Modul 9.2](../../modul-9.2-delivery-observability/README.md).
