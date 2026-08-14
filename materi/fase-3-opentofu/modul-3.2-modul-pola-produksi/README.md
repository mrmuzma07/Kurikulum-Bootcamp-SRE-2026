# Modul 3.2 — Modul & Pola Produksi OpenTofu

> **Tujuan akhir:** menyusun konfigurasi OpenTofu yang reusable, terisolasi per environment, aman terhadap perubahan address dan state, serta siap direview dan dipromosikan melalui pipeline tanpa menyimpan credential sebagai plain text.

## Capaian Modul (Wajib)

- [x] Membedakan root module, child module, caller, input contract, dan output contract.
- [x] Menyusun layout `modules/` dan `environments/` dengan ownership yang jelas.
- [x] Menjelaskan provider passing, provider alias, versioning, composition, dan batas abstraksi module.
- [x] Membandingkan workspace dengan direktori terpisah untuk isolasi environment dan state.
- [x] Memakai backend key yang berbeda untuk `dev`, `staging`, dan `prod`.
- [x] Memilih `for_each`, `count`, conditional, dan data source tanpa address churn yang tidak disengaja.
- [x] Menjelaskan secret boundary, `sensitive`, environment-driven credential, Vault/SOPS/secret manager, dan paparan state.
- [x] Menyusun alur CI `fmt → validate → plan`, retensi plan artifact, approval, dan promotion.
- [x] Menjelaskan handoff metadata OpenTofu ke Ansible tanpa memindahkan ownership konfigurasi OS ke OpenTofu.
- [x] Mengumpulkan evidence plan/module/state yang sudah diredáksi.

> Checklist materi tidak berarti seluruh runtime telah dieksekusi. Status runtime harus ditulis berdasarkan command dan evidence yang benar-benar tersedia.

## Rencana 3 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01 — Arsitektur Modul Reusable](01-arsitektur-modul-reusable.md) dan [02 — Environment, Workspace, State](02-environment-workspace-dan-state.md) | Rancang layout repository dan contract module; mulai [LAB-01](lab/LAB-01-modul-reusable-web-server.md) |
| 2 | [03 — `for_each`, `count`, Conditional, Data Source](03-foreach-count-conditional-data-source.md) | Implementasi instance map, conditional, dan review address; lanjut LAB-01 dan LAB-02 |
| 3 | [04 — Secret Handling dan CI Plan](04-secret-handling-dan-ci-plan.md) | [LAB-02 Promotion & Secret Safety](lab/LAB-02-environment-promotion-dan-secret-safety.md), latihan, dan kuis |

## Prasyarat

- [Modul 3.1 — Dasar OpenTofu](../modul-3.1-dasar-opentofu/README.md).
- Linux/Git, container, OrbStack, dan dasar Kubernetes dari Fase 0–2.
- Mac Apple Silicon/ARM64 dengan OrbStack atau Docker-compatible runtime untuk fast lane.
- Pemahaman bahwa state dapat memuat atribut sensitif walaupun HCL tidak menulis secret.
- Tidak perlu Vault, SOPS, remote backend, atau CI aktif untuk mengikuti jalur statis; fitur tersebut dijelaskan sebagai pola yang harus diverifikasi di organisasi.

## Dua Lane Praktik

### Local / Docker / OrbStack Fast Lane

- Child module Docker web server pada resource disposable.
- `for_each` untuk beberapa instance dengan port dan nama yang eksplisit.
- Local state atau backend lab yang telah disetujui.
- Plan, graph, state inspection, dan HTTP check lokal.
- Jika `tofu` atau Docker tidak tersedia, selesaikan walkthrough HCL dan tandai runtime **belum diverifikasi**.

### Production On-Prem Provisioning Lane

- Root module per environment memanggil module yang direview dan versioned.
- Remote S3-compatible state dengan key, permission, backup, dan locking terpisah.
- Plan artifact direview; apply/promotion membutuhkan approval yang sesuai.
- OpenTofu menyediakan metadata host/IP/role ke Ansible; Ansible mengerjakan bootstrap dan konfigurasi OS; instalasi k3s mengikuti prasyarat host.
- Provider Proxmox, vSphere, libvirt, CloudInit, atau HTTP/REST belum diklaim teruji di modul ini.

## Layout Repository yang Direkomendasikan

```text
infra/
├── modules/
│   └── web-server/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── providers.tf
    │   ├── backend.tf
    │   └── variables.tf
    ├── staging/
    └── prod/
```

`modules/` berisi contract reusable dan tidak memilih environment. `environments/` adalah root module yang memilih backend, provider configuration, ukuran deployment, policy, dan approval boundary. Directory `prod` tidak boleh hanya dibedakan oleh workspace name jika proses organisasi membutuhkan review dan permission yang benar-benar terpisah.

## Deliverables Modul

1. Child module `web-server` dengan variable type/validation, output, label ownership, dan provider passing yang eksplisit.
2. Root caller dengan `for_each` instance map dan state address yang stabil.
3. Layout environment dengan backend/state key berbeda; bukti berupa konfigurasi dan review, bukan credential.
4. Catatan perbandingan workspace versus directory, termasuk blast radius dan recovery.
5. Simulasi promotion berbasis plan artifact yang tidak mengandung secret.
6. Desain secret boundary dan CI plan yang menjelaskan Vault/SOPS/secret manager tanpa mengklaim integrasi aktif.
7. Jawaban latihan serta kuis minimal **80%**.

## Guardrail Modul, Environment, State, dan Credential

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Module tidak boleh membaca credential dari repository, default variable, output biasa, atau log CI.
- `sensitive = true` hanya menyamarkan tampilan; state backend tetap harus dilindungi, dibatasi, dibackup, dan diaudit.
- Setiap environment memiliki directory/backend key dan permission yang dapat diverifikasi; jangan menjalankan command dari `prod` saat bermaksud menguji `dev`.
- `for_each` memakai key bisnis yang stabil; rename key berarti address change yang harus direncanakan.
- `tofu apply`, `tofu destroy`, `tofu import`, `tofu state mv`, dan `tofu state rm` dibatasi pada scope yang telah diverifikasi. `-auto-approve` bukan default.
- Simpan `.terraform/`, state, var file lokal, dan plan binary di luar Git melalui `.gitignore`; ignore bukan pengganti secret manager.
- Jangan meneruskan secret dari OpenTofu ke Ansible melalui output atau artifact biasa. Gunakan secret mechanism terpisah dan hanya kirim metadata minimum.
- Jangan memakai `kubectl delete -A`; jangan menjalankan `k3s server --cluster-reset` atau restore snapshot pada cluster aktif tanpa prosedur resmi; jangan menguji chaos pada cluster production.

## Kaitan dengan Modul Berikutnya

- [Modul 3.1 — Dasar OpenTofu](../modul-3.1-dasar-opentofu/README.md) menjadi dasar workflow, state, import, dan drift.
- Modul 3.3 — Konteks On-Prem masih menyusul; modul tersebut akan membahas provider on-prem dan boundary provisioning yang lebih nyata.
- Fase 4 — Ansible masih menyusul; fase tersebut akan menerima metadata provisioning untuk inventory/bootstrap. Jangan mengklaim file atau runtime Fase 4 tersedia bila belum ada.

## Acceptance Criteria Modul

- [ ] Module dapat dibaca sebagai contract mandiri: input, output, provider, ownership, dan failure mode terdokumentasi.
- [ ] Caller environment tidak berbagi backend key secara tidak sengaja.
- [ ] Contoh `for_each`, `count`, conditional, dan data source menjelaskan address serta ownership.
- [ ] Secret tidak muncul dalam HCL, example value, output, evidence, atau artifact.
- [ ] Plan CI memiliki gate review/approval dan tidak auto-apply ke production.
- [ ] Evidence menyebutkan command yang benar-benar dijalankan; runtime yang tidak tersedia ditandai jelas.

## Catatan Status

Modul 3.2 tersedia sebagai materi reusable dan pola produksi. Lab Docker/OrbStack, remote backend, Vault/SOPS, pipeline CI, dan promotion production hanya boleh dinyatakan berhasil jika execution evidence tersedia. Modul 3.3 tetap menyusul.

## Catatan SRE

Reusable module mengurangi duplikasi, tetapi juga memperluas blast radius ketika contract berubah. Perlakukan perubahan variable, output, provider, module version, backend key, dan `for_each` key sebagai perubahan API: review diff, prediksi address impact, siapkan rollback, dan promosikan bertahap.
