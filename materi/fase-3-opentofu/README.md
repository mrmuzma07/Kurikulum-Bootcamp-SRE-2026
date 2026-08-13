# Fase 3 — Infrastructure as Code: OpenTofu

> **Tujuan fase:** menguasai konsep dan praktik *Infrastructure as Code* (IaC) dengan OpenTofu, mengelola state secara aman, menyusun modul yang *reusable*, serta memahami jalur provisioning infrastruktur on-prem menuju automation dengan Ansible dan Kubernetes.

## Durasi

Minggu 6–7

## Modul di Fase Ini

| # | Modul | Durasi | Status |
|---|---|---|---|
| 3.1 | Dasar OpenTofu | 2 hari | ✅ Tersedia |
| 3.2 | Modul & Pola Produksi | 3 hari | ⏳ Menyusul |
| 3.3 | Konteks On-Prem & Provisioning | 3 hari | ⏳ Menyusul |

## Capaian Fase (Wajib)

- [x] Bisa menjelaskan alur kerja IaC (*desired state*, deklaratif vs imperatif, idempotency, dan *reconciliation*).
- [x] Bisa membedakan OpenTofu dan Terraform dari sisi lisensi, komunitas, serta evolusi arsitektur.
- [x] Bisa menulis sintaks HCL (*provider*, *resource*, *variable*, *output*, *locals*, *data source*).
- [x] Bisa menjalankan workflow standar: `init → fmt → validate → plan → apply → destroy`.
- [x] Bisa mengelola *state* lokal dan remote (S3-compatible/MinIO di OrbStack) lengkap dengan *locking*.
- [x] Bisa melakukan inspeksi, pemindahan, penghapusan *state*, *import* resource manual, serta *drift detection/remediation*.
- [ ] Bisa menyusun modul *reusable* dengan struktur `modules/` dan `environments/` (Modul 3.2).
- [ ] Bisa mengelola *secrets* secara aman tanpa *commit plain text* (Modul 3.2).
- [ ] Bisa menghubungkan OpenTofu *provisioning* ke Ansible *configuration management* dan k3s *installation* (Modul 3.3).

> Status capaian di atas menandakan kesiapan materi Modul 3.1; eksekusi *apply* dan *state modification* tetap dilakukan pada environment lab disposable.

## Dua Lane Praktik

```text
Local / Docker / OrbStack Fast Lane
  → provider Docker/local, fast feedback loop, MinIO di OrbStack, drift simulation, import exercise

Production On-Prem Provisioning Lane
  → HCL modules, S3/MinIO remote backend with locking, handoff metadata to Ansible, multi-environment isolation
```

## Prasyarat

- Fase 0 (Linux & Git), Fase 1 (Container & OrbStack), dan Fase 2 (Kubernetes) selesai.
- Binary `tofu` terpasang di Mac (`brew install opentofu`).
- Runtime OrbStack aktif untuk pengujian container dan backend S3 MinIO.

## Guardrail Operasional IaC

- Jangan menyimpan *secret*, *access key*, *secret key*, atau *credentials* secara *plain text* dalam file `.tf`, `.tfvars`, atau Git.
- Selalu tinjau hasil `tofu plan` sebelum mengeksekusi `tofu apply`.
- Jangan menggunakan `-auto-approve` untuk perubahan produksi.
- Selalu buat *backup state* sebelum mengeksekusi `tofu state rm`, `tofu state mv`, atau *backend migration*.
- Pembatalan atau penghapusan infrastruktur (`tofu destroy`) harus memiliki *scope* terverifikasi.

## Kaitan dengan Fase Berikutnya

- **Fase 4 — Ansible:** menerima output metadata (IP/host) hasil *provisioning* OpenTofu untuk bootstrap OS dan k3s.
- **Fase 5 — Helm & Fase 6 — GitOps:** mengelola aplikasi di atas cluster yang infrastruktur dasarnya telah siap dan konsisten.
