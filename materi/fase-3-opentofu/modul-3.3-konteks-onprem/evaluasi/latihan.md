# Latihan Modul 3.3 — Konteks On-Prem

> Kerjakan tanpa credential nyata dan tanpa mutation production. Gunakan placeholder untuk endpoint, host, dan version.

## Latihan 1 — Pilihan Provider

Pilih provider/boundary yang paling tepat dan jelaskan alasannya:

1. Tim perlu membuat VM dari template pada cluster Proxmox dengan network dan storage tertentu.
2. Tim memiliki vCenter dan ingin mengelola VM/template/portgroup secara deklaratif.
3. Lab Linux memiliki QEMU/KVM dan libvirt daemon untuk menguji guest VM.
4. Platform internal hanya menyediakan endpoint REST untuk request server.
5. Host membutuhkan hostname dan package minimum saat first boot.

Untuk setiap jawaban, tulis API, identity, version, state ownership, dan failure mode yang harus diverifikasi.

## Latihan 2 — Simulasi versus Production

Buat tabel dua kolom:

- dibuktikan oleh Docker/mock;
- tidak dibuktikan oleh Docker/mock.

Isi minimal: module contract, `for_each`, labels, port, state address, VM boot, systemd, SSH, CloudInit, storage, hypervisor, k3s readiness, dan L2 network.

## Latihan 3 — Address Stability

Diberikan map `public`, `admin`, dan `worker-1`:

1. tulis predicted resource addresses;
2. jelaskan dampak mengganti key `worker-1` menjadi `worker-a`;
3. tulis review yang harus dilakukan sebelum apply;
4. jelaskan mengapa list index tidak selalu cocok untuk identity host.

## Latihan 4 — Handoff Contract

Buat output metadata non-secret untuk dua host control-plane/worker. Sertakan hostname, address, role, environment, module version, dan provisioning reference. Jelaskan mengapa SSH private key, password, k3s token, kubeconfig, dan provider credential tidak boleh menjadi output.

## Latihan 5 — Readiness Gate

Buat checklist sebelum Ansible berjalan yang mencakup:

- identity;
- management/node network;
- IP/DNS/route;
- time synchronization;
- SSH;
- OS/kernel/resource;
- storage;
- firewall;
- security baseline.

Tambahkan stop condition untuk setiap gate.

## Latihan 6 — Failure Analysis

Jelaskan tindakan aman untuk tiga skenario:

1. provider timeout tetapi VM mungkin sudah dibuat;
2. Ansible gagal pada satu worker;
3. OpenTofu plan ingin replace control-plane yang telah masuk cluster.

Jawaban wajib menyebut scope, state, evidence, owner, dan approval.

## Latihan 7 — Evidence Chain

Susun chain berikut secara benar:

```text
plan, k3s health, commit SHA, provider lock, approval, Ansible result,
host metadata, readiness, backend/state key, apply result
```

Jelaskan mengapa raw state, raw plan, token, dan kubeconfig bukan artifact repository biasa.

## Latihan 8 — Production Promotion

Buat promotion checklist dari disposable simulation ke staging lalu production. Sertakan alasan mengapa plan harus dibuat pada target state/context dan tidak boleh disalin antar-environment.

## Latihan 9 — CloudInit Boundary

Tulis user-data CloudInit non-secret untuk hostname marker dan package minimum. Jelaskan risiko bila user-data berisi password atau token dan siapa yang mengambil alih setelah first boot.

## Latihan 10 — SRE Decision

Tulis kapan automation harus berhenti walaupun `tofu plan` valid. Gunakan minimal lima kondisi: identity salah, backend key tidak jelas, IP collision, replacement tanpa migration plan, readiness tidak diverifikasi, dan evidence mengandung secret.

## Rubrik

- Ketepatan konsep provider/boundary: 25%
- Batas simulasi dan ARM64: 15%
- Handoff metadata/readiness: 25%
- Failure/recovery/evidence: 25%
- Secret dan destructive guardrail: 10%

Nilai minimal: **80%**. Pelanggaran secret/destructive guardrail menggugurkan latihan walaupun jawaban teknis lainnya benar.

## Kaitan

Gunakan hasil latihan untuk memeriksa [Kuis dan Jawaban](kuis-dan-jawaban.md) dan [LAB-02](../lab/LAB-02-handoff-ke-ansible-dan-k3s.md).
