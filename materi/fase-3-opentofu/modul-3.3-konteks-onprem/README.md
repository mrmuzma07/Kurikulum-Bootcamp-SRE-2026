# Modul 3.3 — Konteks On-Prem & Provisioning

> **Tujuan akhir:** memahami cara memilih provider dan boundary on-prem, menyusun simulasi provisioning yang aman, serta merancang handoff OpenTofu → Ansible → k3s tanpa mencampur ownership, credential, dan evidence runtime.

## Capaian Modul (Wajib)

- [ ] Menjelaskan peran provider Proxmox, vSphere, libvirt, CloudInit, dan HTTP/REST dalam konteks on-prem.
- [ ] Membedakan provider, resource, data source, guest initialization, dan configuration management.
- [ ] Memilih provider berdasarkan API, identity, version compatibility, state behavior, dan failure mode.
- [ ] Membuat simulasi lokal dengan Docker, libvirt, atau mock tanpa menganggap container sebagai VM production.
- [ ] Menyusun input/output contract untuk metadata host yang non-secret.
- [ ] Menjelaskan network, IP, DNS, storage, time sync, firewall, dan readiness gate sebelum bootstrap.
- [ ] Merancang boundary OpenTofu provisioning → Ansible bootstrap/configuration → k3s installation.
- [ ] Menyusun evidence chain dari commit/provider lock hingga plan, approval, handoff, dan health check.
- [ ] Menjelaskan drift, replacement, recovery, dan stop condition pada environment on-prem.
- [ ] Menjaga secret, state, kubeconfig, token, dan credential tetap di luar repository dan artifact umum.

> Checklist materi tidak berarti provider on-prem atau cluster telah dijalankan. Setiap klaim runtime harus didukung command, environment, timestamp, dan evidence yang dapat diaudit.

## Rencana 3 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01 — Provider On-Prem dan Boundary](01-provider-onprem-dan-boundary.md), [02 — Simulasi Lokal](02-simulasi-lokal-docker-libvirt-mock.md) | [LAB-01 — Simulasi Provisioning Lokal](lab/LAB-01-simulasi-provisioning-lokal.md) |
| 2 | [03 — OpenTofu, Ansible, dan k3s](03-opentofu-ansible-k3s-handoff.md) | [LAB-02 — Handoff ke Ansible dan k3s](lab/LAB-02-handoff-ke-ansible-dan-k3s.md) |
| 3 | [04 — Production Readiness dan Evidence](04-production-readiness-dan-evidence.md) | [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 3.1 — Dasar OpenTofu dan Modul 3.2 — Modul & Pola Produksi.
- Fase 0–2: Linux/Git, container, OrbStack, networking, k3s, MetalLB, dan troubleshooting Kubernetes.
- MacBook Apple Silicon/ARM64 dengan OrbStack untuk fast lane, bila tersedia.
- Pemahaman bahwa provider, backend, Ansible, dan k3s memiliki lifecycle serta failure mode berbeda.
- Tidak perlu akses Proxmox/vSphere/libvirt, Ansible, atau cluster k3s untuk mengikuti jalur statis.

## Dua Lane Praktik

### Lane A — Simulasi Lokal

```text
OpenTofu + provider Docker/mock
  → object disposable yang mewakili host/service
  → metadata non-secret
  → review predicted plan dan contract
```

Docker membantu menguji wiring module, labels, ports, dan output. Docker tidak membuktikan boot VM, systemd, SSH, kernel, CloudInit, storage block, quorum, atau network L2 on-prem. Libvirt atau mock hanya boleh disebut runtime bila benar-benar tersedia dan dieksekusi.

### Lane B — Production On-Prem Design

```text
OpenTofu
  → provision VM/server/network attachment/storage
  → output metadata minimum
  → Ansible bootstrap, hardening, dan konfigurasi OS
  → host readiness gate
  → Ansible memasang dan mengonfigurasi k3s
  → health check, evidence, dan handoff berikutnya
```

Provider Proxmox, vSphere, libvirt, CloudInit, HTTP/REST, Ansible, dan k3s pada lane ini dibahas sebagai desain kecuali execution evidence tersedia.

## Boundary Ownership

| Boundary | Tanggung jawab utama | Bukan tanggung jawab utama |
|---|---|---|
| OpenTofu/provider | VM/server, attachment, network reference, storage reference, metadata | patch OS harian, isi file daemon, rotasi credential |
| CloudInit/guest init | first boot, user/package minimum, hostname/network seed | pengganti konfigurasi berulang dan policy jangka panjang |
| Ansible | bootstrap, hardening, package, service, konfigurasi host, k3s runbook | membuat state provider atau menggantikan backend OpenTofu |
| k3s | control plane/agent dan lifecycle cluster sesuai runbook | provisioning hypervisor atau pengelolaan secret repository |
| Helm/GitOps | aplikasi di atas cluster | provisioning VM dan bootstrap OS |

## Struktur Materi

```text
modul-3.3-konteks-onprem/
├── README.md
├── 01-provider-onprem-dan-boundary.md
├── 02-simulasi-lokal-docker-libvirt-mock.md
├── 03-opentofu-ansible-k3s-handoff.md
├── 04-production-readiness-dan-evidence.md
├── lab/
│   ├── LAB-01-simulasi-provisioning-lokal.md
│   └── LAB-02-handoff-ke-ansible-dan-k3s.md
└── evaluasi/
    ├── latihan.md
    └── kuis-dan-jawaban.md
```

## Deliverables Modul

1. Matriks pilihan provider dan boundary ownership.
2. Simulasi module host/service disposable dengan input/output contract non-secret.
3. Diagram dan tabel network/IP/DNS/storage serta readiness gate.
4. Metadata handoff yang dapat menjadi inventory Ansible tanpa credential.
5. Runbook sequencing OpenTofu → Ansible → k3s dengan stop condition dan rollback decision.
6. Evidence checklist yang menghubungkan commit, provider lock, backend key, plan, approval, handoff, dan health check.
7. Jawaban latihan dan kuis dengan nilai minimal **80%**.

## Guardrail Modul

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menulis access key, secret key, password, private key, token, kubeconfig, atau nilai credential nyata pada HCL, README, shell history, `.tfvars`, output, plan artifact, evidence, atau log.
- `sensitive = true` hanya menyamarkan presentasi; state dan plan tetap harus diperlakukan sebagai data sensitif.
- `.gitignore` bukan pengganti secret manager.
- Jangan memakai `-auto-approve` sebagai default.
- Apply, destroy, import, `state mv`, dan `state rm` hanya pada scope disposable yang diverifikasi, dengan plan, backup, lock, review, dan approval.
- Jangan menjalankan provider on-prem nyata atau menyentuh production hanya untuk mengikuti walkthrough.
- Jangan menjalankan `kubectl delete -A`, `k3s server --cluster-reset`, atau restore snapshot pada cluster aktif tanpa prosedur resmi. Jangan menguji chaos pada cluster production.
- Jangan mengklaim provider, remote backend, CloudInit, Ansible, k3s handoff, CI, secret rotation, atau promotion berhasil tanpa execution evidence.

## Acceptance Criteria Modul

- [ ] Provider on-prem dibandingkan berdasarkan API, identity, version, state, ownership, dan failure mode.
- [ ] Simulasi lokal menyatakan dengan jelas apa yang dibuktikan dan apa yang tidak dibuktikan.
- [ ] Metadata output hanya berisi hostname/address/role/environment/version atau atribut non-secret yang disetujui.
- [ ] Readiness gate mencakup network, IP, DNS, time sync, SSH, storage, firewall, dan resource capacity.
- [ ] Boundary provisioning, configuration, dan k3s ditulis dengan owner dan stop condition.
- [ ] Evidence chain dan recovery plan dapat direview tanpa credential atau raw artifact.
- [ ] Semua runtime yang tidak dijalankan ditandai **belum diverifikasi**.

## Status Runtime

Materi ini menyediakan desain dan walkthrough. Provider Proxmox/vSphere/libvirt/CloudInit/HTTP-REST, Ansible, k3s handoff, dan production on-prem belum diklaim berhasil tanpa evidence pada environment yang sesuai. Jika binary atau endpoint tidak tersedia, gunakan Jalur B pada setiap lab.

## Kaitan dengan Modul Berikutnya

- Modul 3.2 menyediakan module, environment, state, dan secret boundary yang dipakai di sini.
- Fase 4 — Ansible masih menyusul; handoff nyata dan playbook idempotent akan dilatih di sana.
- Modul 2.2 dan 2.4 menyediakan topologi, k3s readiness, backup, quorum, dan operational guardrail yang menjadi prasyarat handoff.
- Fase 5/6 akan mengelola aplikasi di atas cluster setelah infrastructure dan host siap.

## Catatan SRE

On-prem bukan cloud dengan endpoint yang berbeda. Tim harus memiliki ownership atas network, IP, DNS, hypervisor, storage, firmware/OS, identity, backup, dan failure recovery. Automation yang baik memperkecil perubahan manual, tetapi tetap membutuhkan preflight, plan review, stop condition, dan evidence yang menghubungkan setiap boundary.
