# Fase 3 — Infrastructure as Code: OpenTofu

> **Tujuan fase:** menguasai konsep dan praktik *Infrastructure as Code* (IaC) dengan OpenTofu, mengelola state secara aman, menyusun modul yang *reusable*, serta memahami jalur provisioning infrastruktur on-prem menuju automation dengan Ansible dan Kubernetes.

## Durasi

Minggu 6–7

## Modul di Fase Ini

| # | Modul | Durasi | Status |
|---|---|---|---|
| 3.1 | Dasar OpenTofu | 2 hari | ✅ Tersedia |
| 3.2 | Modul & Pola Produksi | 3 hari | ✅ Tersedia |
| 3.3 | Konteks On-Prem & Provisioning | 3 hari | ✅ Tersedia |

## Capaian Fase (Wajib)

- [x] Bisa menjelaskan alur kerja IaC (*desired state*, deklaratif vs imperatif, idempotency, dan *reconciliation*).
- [x] Bisa membedakan OpenTofu dan Terraform dari sisi lisensi, komunitas, serta evolusi arsitektur.
- [x] Bisa menulis sintaks HCL (*provider*, *resource*, *variable*, *output*, *locals*, *data source*).
- [x] Bisa menjalankan workflow standar: `init → fmt → validate → plan → apply → destroy`.
- [x] Bisa mengelola *state* lokal dan remote (S3-compatible/MinIO di OrbStack) lengkap dengan *locking*.
- [x] Bisa melakukan inspeksi, pemindahan, penghapusan *state*, *import* resource manual, serta *drift detection/remediation*.
- [x] Bisa menyusun modul *reusable* dengan struktur `modules/` dan `environments/` (Modul 3.2).
- [x] Bisa mengelola *secrets* secara aman tanpa *commit plain text* (Modul 3.2).
- [x] Bisa menjelaskan konteks provider on-prem, simulasi lokal, dan boundary OpenTofu → Ansible → k3s (Modul 3.3).

> Status capaian di atas menandakan kesiapan materi Modul 3.1–3.3. Eksekusi *apply*, *state modification*, provisioning provider, Ansible, dan k3s tetap harus dibuktikan pada environment lab yang sesuai.

## Dua Lane Praktik

```text
Local / Docker / OrbStack Fast Lane
  → provider Docker/local, fast feedback loop, MinIO di OrbStack, drift simulation, import exercise

Production On-Prem Provisioning Lane
  → HCL modules, remote backend dengan locking, handoff metadata to Ansible, multi-environment isolation
```

> Runtime Docker/OrbStack, MinIO, remote backend, locking, provider on-prem, Ansible, dan k3s tidak diklaim berhasil tanpa execution evidence.

## Capaian Modul 3.1 — Dasar OpenTofu

- Konsep IaC, desired state, deklaratif versus imperatif, idempotency, reconciliation, dan drift.
- HCL: provider, resource, variable, output, locals, dan data source.
- Workflow `init`, `fmt`, `validate`, `plan`, `apply`, dan `destroy`.
- State lokal/remote, locking, import, state inspection, drift, dan recovery.

## Capaian Modul 3.2 — Modul & Pola Produksi

- Struktur `modules/` dan `environments/` dengan root/child module serta input-output contract.
- Isolasi directory, workspace, backend key, provider alias, dan approval per environment.
- Penggunaan `for_each`, `count`, conditional, dan data source dengan address yang dapat direview.
- Secret boundary, state/artifact exposure, Vault/SOPS/secret manager secara konseptual, dan CI plan gate.
- Promotion berbasis commit/module version dan plan baru pada target environment.

## Capaian Modul 3.3 — Konteks On-Prem & Provisioning

- Provider Proxmox, vSphere, libvirt, CloudInit, dan HTTP/REST API internal sebagai boundary integrasi.
- Simulasi Docker/libvirt/mock pada lane lokal dan batas bukti dibanding VM production.
- Readiness network, IP, DNS, storage, time sync, firewall, image, state, dan identity.
- Contract metadata non-secret dari OpenTofu ke Ansible.
- Sequencing provisioning host → bootstrap/configuration → k3s installation dan operational handoff.
- Evidence chain, drift/recovery, approval, promotion, dan stop condition.

> Modul 3.3 tersedia sebagai materi dan walkthrough. Provider on-prem, runtime remote, Ansible, k3s handoff, CI, Vault/SOPS, dan promotion production harus dibuktikan dengan execution evidence sebelum diklaim berhasil.

## Prasyarat

- Fase 0 (Linux & Git), Fase 1 (Container & OrbStack), dan Fase 2 (Kubernetes) selesai.
- Binary `tofu` terpasang di Mac (`brew install opentofu`) bila mengikuti runtime lane.
- Runtime OrbStack/Docker aktif bila mengikuti simulasi container.
- Akses Proxmox/vSphere/libvirt, Ansible, atau cluster k3s tidak diperlukan untuk mengikuti static lane.

## Guardrail Operasional IaC

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menyimpan access key, secret key, password, private key, token, kubeconfig, credential nyata, raw state, atau raw plan di HCL, README, evidence, log, shell history, `.tfvars`, Git, atau artifact.
- `sensitive = true` hanya menyamarkan presentasi; state dan plan tetap harus diperlakukan sebagai data sensitif.
- `.gitignore` bukan pengganti secret manager.
- Selalu tinjau hasil `tofu plan` sebelum mengeksekusi `tofu apply`.
- Jangan menggunakan `-auto-approve` sebagai default.
- Apply, destroy, import, `state mv`, dan `state rm` hanya pada scope disposable yang diverifikasi, dengan plan, backup, lock, review, dan approval.
- Jangan menjalankan provider on-prem nyata atau menyentuh production hanya untuk mengikuti walkthrough.
- Jangan menjalankan `kubectl delete -A`, `k3s server --cluster-reset`, atau restore snapshot pada cluster aktif tanpa prosedur resmi. Jangan menguji chaos pada cluster production.

## Kaitan dengan Fase Berikutnya

- [Modul 3.2 — Modul & Pola Produksi](modul-3.2-modul-pola-produksi/README.md) menyediakan reusable module, environment isolation, address safety, secret boundary, dan CI plan pattern.
- [Modul 3.3 — Konteks On-Prem & Provisioning](modul-3.3-konteks-onprem/README.md) membahas provider Proxmox/vSphere/libvirt/CloudInit/HTTP-REST, simulasi lokal, readiness, serta boundary OpenTofu → Ansible → k3s. Provider dan runtime production tetap tidak diklaim teruji tanpa execution evidence.
- [Fase 4 — Ansible](../fase-4-ansible/README.md) menerima metadata non-secret hasil provisioning untuk bootstrap OS dan k3s. Runtime Ansible, SSH, dan k3s tetap belum diverifikasi tanpa execution evidence.
- **Fase 5 — Helm & Fase 6 — GitOps:** masih menyusul; mengelola aplikasi di atas cluster yang infrastrukturnya telah siap dan konsisten.
