# Modul 3.1 — Dasar OpenTofu

> **Tujuan akhir:** memahami cara berpikir Infrastructure as Code (IaC), menulis konfigurasi HCL dasar, menjalankan workflow OpenTofu dengan review plan, serta mengelola state local/remote secara aman pada environment lab.

## Capaian Modul (Wajib)

- [x] Menjelaskan perbedaan imperative dan declarative serta hubungan desired state, current state, dan state file.
- [x] Menjelaskan OpenTofu sebagai proyek open source dan membandingkannya dengan Terraform dari sisi lisensi dan komunitas.
- [x] Membaca dan menulis blok HCL `terraform`, `provider`, `resource`, `variable`, `locals`, dan `output`.
- [x] Menjalankan `tofu fmt`, `tofu validate`, `tofu plan`, dan `tofu apply` pada resource lab yang disposable.
- [x] Menjelaskan kapan memakai local state dan remote state S3-compatible/MinIO.
- [x] Memahami locking, concurrency control, backup state, import, state inspection, dan drift remediation.
- [x] Menyusun evidence plan/state tanpa mencetak credential atau secret.

> Checklist di atas menunjukkan materi dan lab sudah tersedia. Eksekusi resource tetap harus dilakukan pada project lab yang scope-nya jelas.

## Rencana 2 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01 — Konsep IaC & OpenTofu](01-konsep-iac-opentofu.md), [02 — HCL, Resource, Variable, Output, Provider](02-hcl-resource-variable-output-provider.md) | Mulai [LAB-01 Docker Web Server](lab/LAB-01-tofu-docker-web-server.md), latihan desain dan HCL |
| 2 | [03 — Workflow](03-workflow-init-plan-apply-destroy.md), [04 — State, Remote, Import, Drift](04-state-remote-import-drift.md) | [LAB-02 State MinIO, Import & Drift](lab/LAB-02-state-minio-import-drift.md), latihan, kuis |

## Prasyarat

- Fase 0–2: Linux/Git, container, OrbStack, dan dasar Kubernetes.
- Mac Apple Silicon/ARM64 dengan OrbStack atau Docker-compatible runtime.
- OpenTofu dapat dipasang dengan `brew install opentofu` atau mengikuti paket resmi yang telah direview.
- Paham bahwa state dapat berisi nilai sensitif walaupun konfigurasi HCL tidak memuat secret.

## Setup Mac ARM64

```bash
brew install opentofu
tofu version

docker version

docker info --format '{{.OSType}}/{{.Architecture}}'
```

Jika binary/provider atau Docker runtime tidak tersedia, baca materi dan lakukan validasi statis. Jangan mengganti verifikasi yang gagal dengan klaim bahwa resource sudah dibuat.

## Deliverables Modul

1. Project HCL Docker web server yang memakai variable dan output.
2. Output `tofu plan` yang direview serta evidence `tofu state list/show` tanpa secret.
3. Catatan perbedaan local state dan remote S3-compatible state.
4. Bukti walkthrough import resource manual dan drift detection/remediation pada lab disposable, atau catatan bahwa runtime belum tersedia.
5. Jawaban latihan dan kuis dengan nilai minimal **80%**.

## Cara Belajar

1. Baca teori dengan membedakan **evidence**, **desired state**, dan **assumption**.
2. Jalankan `tofu fmt`/`validate` sebelum meminta perubahan.
3. Simpan plan binary atau output yang sudah diredáksi hanya di direktori lab lokal yang di-ignore Git.
4. Review setiap address resource dan action (`create`, `update`, `delete`, `replace`) sebelum apply.
5. Catat provider version dan image tag agar hasil dapat direproduksi.

## Guardrail State dan Credential

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menaruh access key/secret key MinIO dalam backend block, README, shell history, atau `.tfvars` yang masuk Git.
- Gunakan environment variable, credential helper, SOPS/Vault/secret manager sebagai pola konseptual.
- Tambahkan `.terraform/`, `*.tfstate`, `*.tfstate.*`, `*.tfvars`, dan `*.tfplan` ke `.gitignore`; tetap pahami bahwa ignore bukan pengganti secret manager.
- Jangan menjalankan `tofu apply`, `tofu destroy`, `tofu import`, `tofu state mv`, atau `tofu state rm` di luar project lab yang telah diverifikasi.
- `-auto-approve` bukan default. Bila demonstrasi disposable memerlukannya, scope, warning, dan cleanup harus tertulis.

## Kaitan dengan Modul Berikutnya

- [Overview Fase 3 — Infrastructure as Code](../README.md) menjelaskan dua lane praktik dan urutan handoff.
- **Modul 3.2 — Modul & Pola Produksi:** memperluas contoh menjadi `modules/`, `environments/`, `for_each`, conditional, data source, dan secret integration. Modul 3.1 hanya mengenalkan struktur reusable sederhana.
- **Modul 3.3 — Konteks On-Prem:** menerapkan provider Proxmox/vSphere/libvirt/HTTP atau mock dan pola OpenTofu → Ansible → k3s.
- **Fase 4 — Ansible:** menerima output host/IP dari OpenTofu untuk bootstrap dan konfigurasi OS; output itu tidak boleh dipakai sebagai alasan menyimpan credential.

## Catatan Status

Modul 3.1 selesai sebagai materi dasar dan lab disposable. Modul 3.2 serta Modul 3.3 tetap menjadi pekerjaan berikutnya; fitur reusable production dan provider on-prem belum diklaim teruji dalam modul ini.

## Catatan SRE

IaC mengurangi perubahan manual, tetapi tidak menghapus kebutuhan change review. State adalah bagian dari control plane automation: hilangnya state, lock yang salah, atau apply pada context yang salah dapat memperbesar blast radius lebih cepat daripada runbook manual.
