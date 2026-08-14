# LAB-01 — Handoff ke k3s Multinode

## Tujuan

Mempraktikkan contract metadata → inventory → readiness → sequencing k3s tanpa membawa secret.

## Static Lane

1. Ambil contoh metadata non-secret dari Modul 3.3.
2. Render inventory `k3s_servers`/`k3s_agents` dengan stable key.
3. Buat readiness matrix untuk tiga server dan worker.
4. Review topology, API address, static IP, DNS, time, firewall, storage, version, CNI/ingress/servicelb.
5. Tulis sequencing tanpa token: server bootstrap, health, server join, agent join, smoke test.
6. Tentukan stop condition setiap tahap.

## Runtime Lane

Hanya jika VM/cluster disposable, Ansible/SSH/kubectl/k3s tersedia, dan approval ada:

```text
inventory graph → SSH ping → readiness read-only → check mode
→ review/approval → server bootstrap → health → join bertahap
```

Jangan menyalin token/kubeconfig ke file. Gunakan secret mechanism terpisah. Jangan menjalankan pada context aktif yang tidak diverifikasi.

## Failure Drill

- server pertama tidak Ready;
- agent memiliki DNS/route salah;
- planned replacement terhadap control-plane.

Stop dan kumpulkan evidence; jangan reset cluster sebagai shortcut.

## Acceptance Criteria

- [ ] Metadata/inventory non-secret.
- [ ] Topology dan quorum direview.
- [ ] Readiness gate mendahului mutation.
- [ ] Token/kubeconfig tidak masuk artifact.
- [ ] Install yang tidak dieksekusi ditandai belum diverifikasi.
