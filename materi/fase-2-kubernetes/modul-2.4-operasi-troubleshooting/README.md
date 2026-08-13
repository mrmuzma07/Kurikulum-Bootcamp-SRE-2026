# Modul 2.4 — Operasi & Troubleshooting Kubernetes

> **Tujuan akhir:** mengoperasikan cluster Kubernetes dengan disiplin SRE: menemukan root cause dari kegagalan workload, mengelola resource, menyimpan dan memulihkan state etcd k3s, serta melakukan rolling maintenance dengan dampak terukur.

## Capaian Modul (Wajib)

- [x] Bisa menjalankan flow debug `get → describe/events → logs → logs --previous → exec`.
- [x] Bisa membedakan dan memperbaiki `CrashLoopBackOff`, `ImagePullBackOff`, `Pending`, dan `OOMKilled`.
- [x] Bisa menghubungkan requests/limits dengan scheduling, QoS, cgroup, dan OOM behavior.
- [x] Bisa menggunakan `kubectl top`/metrics-server dan memahami batas HPA.
- [x] Bisa membuat snapshot etcd k3s, memverifikasi artifact, dan menjelaskan restore pada cluster disposable.
- [x] Bisa merencanakan `cordon → drain → maintenance → uncordon` dengan PDB dan quorum safety.
- [x] Bisa melakukan rolling upgrade k3s satu node pada satu waktu sesuai runbook.
- [x] Bisa mengumpulkan evidence, menjaga context safety, dan menulis incident timeline.

> Status capaian di atas berarti materi dan lab tersedia; eksekusi cluster tetap harus dilakukan pada environment lab yang disetujui.

## Rencana 2 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-debug-workload](01-debug-workload.md), [02-resource-hpa](02-resource-hpa.md) | [LAB-01 Chaos & Debug](lab/LAB-01-chaos-debug-workload.md) |
| 2 | [03-etcd-backup-restore-k3s](03-etcd-backup-restore-k3s.md), [04-upgrade-k3s-rolling](04-upgrade-k3s-rolling.md) | [LAB-02 Backup/Restore & Upgrade](lab/LAB-02-etcd-backup-restore-rolling-upgrade.md) + [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 2.1–2.3 selesai: paham objek Kubernetes, kubectl, k3d, k3s VM, dan MetalLB.
- Context `k3d` dan `k3s` tersedia; peserta dapat membedakan keduanya.
- Untuk LAB-01: k3d multi-node dengan resource cukup.
- Untuk LAB-02: k3s VM OrbStack, idealnya 3 server + 2 worker, dan akses SSH/sudo.
- OrbStack resource dinaikkan sementara sesuai topologi; turunkan lagi setelah lab.
- `kubectl`, `k3d`, `jq` opsional, `curl`, dan tool observasi node tersedia.

## Dua Lane Operasi

```text
k3d fast lane
  → chaos workload, object debugging, scale test, cepat create/delete

k3s VM production lane
  → systemd, embedded etcd, snapshot, quorum, drain, maintenance, upgrade
```

Jangan menjalankan snapshot restore, drain control-plane, atau uninstall pada context yang tidak disposable. Sebelum setiap perubahan:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl get nodes -o wide
```

## Deliverables Modul

1. **Incident report** dari LAB-01: gejala, scope, command, evidence, root cause, mitigasi, dan pencegahan.
2. **Resource/HPA manifest** dengan requests/limits dan catatan perilaku saat beban berubah.
3. **Bukti snapshot etcd**: nama file, ukuran, timestamp, checksum, lokasi backup, dan hasil verifikasi.
4. **Runbook rolling maintenance/upgrade** dengan preflight, drain/PDB, quorum, stop condition, dan rollback.
5. **Diagram jalur troubleshooting** dan catatan perbedaan k3d vs k3s.
6. Nilai kuis minimal **80%**.

## Cara Belajar

1. Mulai dari symptom, bukan asumsi root cause.
2. Selesaikan diagnosis dengan read-only command sebelum mengubah resource.
3. Ubah satu variabel pada satu waktu dan catat before/after.
4. Gunakan namespace lab dan manifest versioned; jangan menyimpan Secret plain text.
5. Simulasikan failure hanya pada cluster yang disetujui instruktur.
6. Setelah setiap eksperimen, restore keadaan sehat dan validasi Service/Ingress/MetalLB.

## Guardrail Operasional

- Jangan memakai `kubectl delete -A`.
- `kubectl drain` dapat mengganggu aplikasi; cek PDB, replica, dan maintenance window.
- Jangan menjalankan `k3s server --cluster-reset` atau restore snapshot pada cluster aktif tanpa prosedur resmi dan approval.
- Backup etcd bukan pengganti backup aplikasi/PV/database.
- HPA tidak dapat bekerja tanpa metrics yang valid dan requests CPU yang sesuai.
- `latest` tidak dipakai untuk incident reproduction; gunakan tag image yang immutable/semver.
- Redact token k3s, kubeconfig, PAT, dan Secret dari laporan.

## Kaitan dengan Modul Berikutnya

- **Fase 3 — OpenTofu:** resource cluster dan network mulai dikelola deklaratif; runbook manual di sini menjadi kandidat automation.
- **Fase 4 — Ansible:** bootstrap k3s, upgrade, backup schedule, dan preflight dapat dijadikan playbook idempotent.
- **Fase 5 — Helm:** manifest workload/resource/HPA dikemas menjadi chart dengan values per environment.
- **Fase 6 — GitOps:** incident fix dan konfigurasi cluster dipromosikan melalui Git, bukan perubahan manual tanpa audit.
- Fondasi Fase 2 selesai: objek, k3d, k3s, MetalLB, troubleshooting, dan etcd operations.

## Catatan SRE

Operational excellence bukan berarti cluster tidak pernah gagal. Targetnya adalah failure yang terdeteksi, dibatasi blast radius-nya, dapat dipulihkan, dan menghasilkan pembelajaran yang masuk ke runbook/automation.
