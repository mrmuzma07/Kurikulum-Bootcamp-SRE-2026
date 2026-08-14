# Kuis dan Jawaban Modul 8.2

## Soal

1. Mengapa static IP pool, VLAN, DNS, ARP, dan MTU harus masuk dependency map?
2. Apa perbedaan failure domain Local PV, NFS, dan SAN?
3. Mengapa MetalLB L2/BGP bukan cloud load balancer?
4. Apa boundary bastion dan manual SSH pada operasi on-prem?
5. Mengapa Velero `Completed` bukan bukti aplikasi pulih?
6. Mengapa etcd snapshot bukan backup database/PV?
7. Apa perbedaan RPO dan RTO?
8. Sebutkan kontrol utama OS patch dan k3s upgrade.
9. Apa yang harus ada pada Trivy remediation/exception policy?
10. Sebutkan urutan umum disaster recovery on-prem.

## Jawaban

1. Kegagalan addressing, routing, name resolution, neighbor discovery, atau frame size dapat memutus service meski Kubernetes terlihat sehat.
2. Local PV terikat node/disk; NFS bergantung network/server; SAN bergantung fabric/zoning/multipath dan controller.
3. MetalLB tetap membutuhkan jaringan, ARP/BGP peer, route policy, switch, capacity, dan health ownership sendiri.
4. Bastion membatasi/audit akses; manual SSH hanya break-glass dengan approval dan follow-up Ansible, bukan desired-state workflow.
5. Itu hanya menunjukkan workflow backup selesai pada scope tersebut, bukan consistency, restore, application health, atau SLO.
6. etcd menyimpan cluster state, bukan isi database/PV atau application-consistency point.
7. RPO adalah data loss maksimum yang dapat diterima; RTO adalah waktu sampai service memenuhi recovery acceptance.
8. Scope allowlist, `--limit`, serial, maintenance window, backup, quorum/PDB/capacity review, rollback/recovery, dan post-check.
9. Severity, digest/provenance, scanner scope/version, owner, due date, remediation, compensating control, dan exception expiry.
10. OpenTofu metadata, Ansible host/k3s, cluster state, Helm/GitOps, objects/PV/data restore, DNS/MetalLB, observability, dan SLO validation.

## Penilaian

Soal 1–10 bernilai 2 poin. Total 20; lulus minimal 16 tanpa pelanggaran guardrail.

## Kaitan

Hubungkan output ke [Modul 8.1](../../modul-8.1-praktik-sre/README.md), [Modul 8.3](../../modul-8.3-runbook-dokumentasi/README.md), [Fase 4](../../../fase-4-ansible/README.md), [Fase 6](../../../fase-6-gitops/README.md), dan [Fase 7](../../../fase-7-observability/README.md).
