# 02 — Role Instalasi dan Upgrade k3s

## 1. Topology

```text
cp1 ─┐
cp2 ─┼─ control-plane/quorum
cp3 ─┘
worker1, worker2 ── agents
```

Jumlah server, datastore, quorum, static IP, API endpoint, CNI, ingress, dan servicelb harus mengikuti runbook Modul 2.2. Jangan bootstrap server kedua sebelum server pertama dan datastore health lulus.

## 2. Role Contract

Role k3s harus mendokumentasikan:

- supported OS/architecture dan versi k3s yang dipin;
- server/agent mode dan urutan join;
- sumber secret reference tanpa menulis token;
- API endpoint, TLS/SAN, CNI/ingress/servicelb decision;
- idempotent install marker dan upgrade behavior;
- health check, rollback, uninstall, dan replacement procedure.

Role komunitas seperti `xanmanning.k3s` dapat dipelajari, tetapi version, source, permissions, dan behavior harus ditinjau sebelum dipakai. Nama role bukan evidence.

## 3. Sequencing

```text
readiness semua host
→ bootstrap server pertama
→ health API/etcd dan backup decision
→ join server berikutnya sesuai quorum
→ join agents
→ node condition/workload smoke test
→ evidence dan promotion
```

Token k3s hanya melalui secret mechanism. Jangan mencetak command install lengkap bila itu akan memuat token atau kubeconfig.

## 4. Upgrade

Pin versi target, baca release notes, pastikan backup dan quorum, lakukan satu node per satu, tunggu health, lalu lanjut. `serial: 1` mengontrol concurrency tetapi tidak menggantikan PDB, drain plan, atau application owner approval.

## 5. Health Check

Gunakan context safety:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'
```

Redact kubeconfig/token dari evidence. Perintah hanya boleh dijalankan pada context yang diverifikasi.

## Stop Conditions

- version atau architecture tidak disetujui;
- API path/DNS/time/firewall belum siap;
- quorum tidak cukup atau node belum healthy;
- secret reference unavailable;
- replacement tanpa cordon/drain/rejoin plan;
- backup/evidence tidak tersedia.

## Acceptance Checklist

- [ ] Topology dan quorum terdokumentasi.
- [ ] Token boundary terpisah dari inventory.
- [ ] Version pin, sequencing, health, upgrade, dan rollback ada.
- [ ] Tidak ada klaim install/upgrade tanpa execution evidence.

## Catatan SRE

K3s installation adalah perubahan control plane, bukan sekadar package install. API readiness, datastore durability, workload behavior, dan recovery harus dinilai bersama.

## Kaitan

Lanjutkan ke [Hardening, Patching, dan Rolling](03-hardening-patching-rolling.md).
