# LAB-02 — Rolling Patching dan Readiness

## Tujuan

Menyusun runbook patching satu node per satu dengan availability, quorum, dan rollback sebagai gate.

## Langkah Static

1. Pilih target `k3s_nodes` dan tetapkan `serial: 1`.
2. Buat preflight: context, node condition, PDB, replica, quorum, API path, backup, maintenance window.
3. Rancang cordon/drain sebagai langkah yang memerlukan owner approval; jangan menulis destructive shortcut.
4. Patch satu node, post-check Ready, workload smoke test, lalu lanjut node berikutnya.
5. Tulis rollback dan stop condition.
6. Buat evidence summary redacted.

## Runtime Optional

Runtime hanya pada disposable cluster yang scope-nya jelas. Sebelum setiap perubahan:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl get nodes -o wide
```

Jangan menjalankan `kubectl delete -A`, cluster reset, atau restore snapshot aktif. Jika node pertama gagal, hentikan batch.

## Acceptance Criteria

- [ ] Serial satu node dan PDB/replica/quorum dibahas.
- [ ] Access recovery dan rollback tersedia.
- [ ] Health gate berada di antara node.
- [ ] Backup boundary dijelaskan.
- [ ] Runtime yang tidak dijalankan berstatus belum diverifikasi.

## Catatan SRE

Rolling maintenance adalah eksperimen availability yang dikontrol. Evidence post-check lebih penting daripada menyelesaikan loop Ansible secepat mungkin.
