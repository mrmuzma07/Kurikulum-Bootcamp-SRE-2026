# 04 — Secret Handling dan CI Plan

> **Tujuan:** merancang boundary secret dan pipeline plan yang aman tanpa menganggap masking output, `.gitignore`, atau approval sebagai pengganti secret manager.

## Tujuan Belajar

Peserta dapat:

- membedakan secret input, state exposure, output, log, dan artifact;
- memilih environment variable, Vault, SOPS, atau secret manager sesuai boundary;
- menyusun CI `fmt`, `validate`, `plan` dengan approval;
- menerapkan policy untuk mencegah apply environment yang salah;
- menyerahkan metadata minimum ke Ansible tanpa mengekspos credential.

## 1. Secret Boundary

Secret boundary menjawab empat pertanyaan:

1. Di mana secret dibuat dan disimpan?
2. Identity mana yang boleh membacanya?
3. Apakah provider/resource menulisnya ke state?
4. Bagaimana secret dirotasi, direvoke, dan diaudit?

OpenTofu dapat menerima input sensitif, tetapi provider mungkin menyimpan atribut tersebut di state. Karena itu, jangan menganggap `sensitive = true` atau environment variable membuat state bebas secret.

```hcl
variable "registry_password" {
  description = "Credential ephemeral dari secret mechanism CI; tidak boleh dicommit."
  type        = string
  sensitive   = true
}

output "registry_password" {
  value     = var.registry_password
  sensitive = true
}
```

Contoh ini hanya menjelaskan metadata sensitivity. Jangan mengisi nilai nyata pada repository, shell history, issue, log, atau evidence.

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## 2. Pilihan Mekanisme

### Environment variable / credential helper

Cocok untuk lab atau provider yang mendukung credential chain. Nilai diberikan oleh secret mechanism lokal/CI dan tidak ditulis ke HCL.

```bash
export TF_VAR_registry_password="$(secret-helper read ci/registry-password)"
tofu plan -out=lab.tfplan
unset TF_VAR_registry_password
```

Command di atas adalah pola konseptual. Pastikan helper tidak mencetak value, shell history tidak merekam secret, process policy sesuai, dan plan/state backend memiliki permission minimum. Jangan mengganti `secret-helper` dengan secret literal.

### Vault atau secret manager

Vault/secret manager menyimpan secret dan memberikan lease/identity-based access. OpenTofu dapat memakai provider/data source atau wrapper CI, tetapi desain harus menjawab TTL, renewal, revoke, audit, state exposure, dan failure mode. Jangan menulis hasil secret ke output atau artifact.

### SOPS

SOPS dapat mengenkripsi file konfigurasi tertentu dengan age/KMS/PGP, tetapi repository tetap harus memiliki policy key access, rotation, review, dan dekripsi hanya pada runner yang dipercaya. File terenkripsi bukan alasan untuk mencetak hasil dekripsi atau menyimpan state di Git. Pastikan provider tidak mempersist value sensitif ke plan/state tanpa perlindungan.

### Secret injection ke Ansible

Metadata OpenTofu seperti hostname, IP, role, dan environment boleh diteruskan ke Ansible. Password SSH, token bootstrap, kubeconfig, dan private key tetap diambil Ansible/runner dari secret mechanism yang terpisah.

## 3. State dan Artifact Exposure

Sensitive value dapat muncul pada:

- state local/remote;
- plan binary dan `tofu show`;
- provider debug log;
- CI job output;
- crash log;
- shell history/process inspection;
- backup dan artifact retention.

Guardrail:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfplan
crash.log
crash.*.log
```

`.gitignore` mencegah sebagian commit baru, bukan menghapus data yang sudah pernah ter-commit, tercopy, masuk backup, atau tampil di log. State backend perlu encryption at rest/in transit, permission, versioning, locking, backup, retention, dan audit sesuai policy.

Jika plan artifact perlu disimpan untuk approval:

- simpan hanya pada CI storage berpermission minimum;
- tetapkan retention pendek dan audit access;
- jangan mengunggah raw plan ke issue/public artifact;
- redaction tidak boleh dianggap dapat menghapus secret dari binary;
- apply hanya plan yang dibuat dari commit, backend, provider, input, dan environment target yang sama.

## 4. CI Pipeline Read-Only dan Mutation

Tahap pull/merge request sebaiknya read-only:

```text
checkout commit
  ↓
tofu fmt -check
  ↓
tofu init (backend context terkontrol)
  ↓
tofu validate
  ↓
tofu plan -out=environment.tfplan
  ↓
tofu show -no-color environment.tfplan (redacted policy)
  ↓
review + policy checks
```

Apply adalah tahap terpisah dengan identity dan approval berbeda:

```text
approved commit + approved plan
  ↓
verify environment/backend/plan metadata
  ↓
tofu apply environment.tfplan
  ↓
provider validation + service health check
  ↓
record evidence dan change result
```

Jangan menyamakan `plan exit code 2` dengan failure service; pahami semantics versi OpenTofu/CI. Jangan memakai `-auto-approve` sebagai default. Production apply harus memiliki approval, concurrency control, lock, change window, dan stop condition.

Contoh guardrail shell non-secret:

```bash
set -eu

expected_environment="dev"
actual_environment="${TOFU_ENVIRONMENT:?TOFU_ENVIRONMENT wajib diset oleh job}"

if [ "$actual_environment" != "$expected_environment" ]; then
  printf 'environment mismatch; stop tanpa apply\n' >&2
  exit 1
fi

tofu fmt -check
tofu validate
tofu plan -out="${expected_environment}.tfplan"
```

Dalam implementasi nyata, pastikan quoting dan input job diuji; contoh tersebut tidak memasukkan credential dan tidak menjalankan apply.

## 5. Policy Checks

Policy minimal memeriksa:

- directory dan environment allowlist;
- backend bucket/key tidak shared lintas environment;
- provider version dipin dan binary checksum diverifikasi sesuai policy;
- image tag/digest tidak mutable `latest`;
- resource address dan action `delete/replace` memerlukan review;
- production apply memerlukan approval;
- plan artifact dibuat dari commit yang sedang direview;
- tidak ada secret literal, raw state, atau private key di diff;
- target/resource scope tidak menggunakan wildcard berbahaya;
- output hanya metadata yang disetujui.

Policy dapat berupa linter, Open Policy Agent/Conftest, CI script, atau review checklist. Tool policy tidak menghapus kebutuhan memahami provider behavior dan runtime health.

## 6. Promotion dan Rollback

Promotion harus menyimpan relasi:

```text
commit SHA → module version → provider lock → environment input
          → backend key → plan artifact → approval → apply result
```

Plan artifact tidak boleh dipindahkan antar-environment. Buat plan baru pada target environment. Rollback module berarti memilih commit/module version sebelumnya, membuat plan target baru, membaca replacement/downtime, lalu approval. Restore state bukan rollback resource otomatis dan dapat memperbesar drift.

## 7. Handoff OpenTofu ke Ansible

Output yang layak diteruskan:

```hcl
output "ansible_inventory_metadata" {
  description = "Metadata non-secret untuk inventory tahap berikutnya."
  value = {
    for key, instance in module.node : key => {
      hostname   = instance.hostname
      address    = instance.ip_address
      role       = instance.role
      environment = var.environment
    }
  }
}
```

Fase 4 dapat mengubah metadata tersebut menjadi inventory sementara atau input pipeline. OpenTofu tidak seharusnya memasang package OS, mengelola file konfigurasi daemon, atau menyimpan SSH credential. Ansible mengerjakan bootstrap, hardening, package, service, dan prasyarat k3s; Helm/GitOps kemudian menangani aplikasi.

## Troubleshooting

### Secret terlihat pada log CI

Hentikan job dan cegah artifact publication. Rotasi/revoke secret melalui owner, hapus exposure sesuai incident procedure, periksa history/log/cache, lalu perbaiki masking dan secret injection. Jangan hanya menghapus baris dari README.

### `sensitive` membuat output tidak terlihat

Itu perilaku presentasi, bukan error. Gunakan metadata non-secret untuk evidence dan inspeksi state dengan redaction. Jangan menonaktifkan sensitivity hanya agar pipeline mudah dibaca.

### Plan artifact tidak dapat di-apply

Periksa commit, provider lock, input, backend key, workspace, dan versi OpenTofu. Buat plan baru jika context berubah. Jangan memaksa apply plan dari environment lain.

### CI menjalankan plan pada prod

Hentikan job. Verifikasi path, environment variable, backend, identity, branch rules, dan allowlist. Tambahkan guardrail yang gagal tertutup (fail closed) sebelum init/plan pada environment sensitif.

### Vault/SOPS tidak tersedia

Gunakan jalur statis: dokumentasikan boundary, identity, lifecycle, rotation, audit, state exposure, dan failure mode. Tandai integrasi runtime **belum diverifikasi**; jangan mengganti dengan secret literal.

## Acceptance Checklist

- [ ] Secret boundary menjelaskan storage, identity, state exposure, rotation, dan audit.
- [ ] Contoh hanya memakai placeholder/helper, tanpa nilai credential nyata.
- [ ] `sensitive` dijelaskan sebagai masking, bukan encryption.
- [ ] CI memiliki `fmt`, `validate`, `plan`, policy, artifact retention, dan approval.
- [ ] Apply/promotion tidak auto-approve dan memakai context target yang sama.
- [ ] Handoff hanya mengeluarkan metadata non-secret ke Ansible.
- [ ] Vault/SOPS/secret rotation tidak diklaim runtime bila belum dieksekusi.

## Catatan SRE

Secret leakage adalah incident meskipun command gagal dan resource tidak berubah. Audit log, plan artifact, state backup, dan CI cache sama pentingnya dengan HCL. Desain pipeline agar gagal sebelum secret dicetak dan agar identity apply lebih sempit daripada identity plan.

## Kaitan dengan Modul Berikutnya

- [LAB-02 — Environment Promotion dan Secret Safety](lab/LAB-02-environment-promotion-dan-secret-safety.md) mempraktikkan review context secara statis/disposable.
- [Modul 3.1 — State, Remote, Import, Drift](../modul-3.1-dasar-opentofu/04-state-remote-import-drift.md) membahas state exposure dan recovery.
- Modul 3.3 — Konteks On-Prem masih menyusul; modul tersebut akan meneruskan metadata ke boundary provisioning/configuration.
