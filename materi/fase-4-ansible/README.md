# Fase 4 — Konfigurasi Server dengan Ansible

> **Tujuan fase:** mengubah metadata host hasil provisioning menjadi konfigurasi server yang konsisten, dapat diulang, dapat diaudit, dan siap menjadi fondasi k3s on-prem.

## Durasi dan Modul

Minggu 8 — tiga modul, masing-masing dua hari latihan dan review.

| Modul | Fokus | Status |
|---|---|---|
| 4.1 | Fundamental Ansible | ✅ Tersedia |
| 4.2 | Pola Produksi Ansible | ✅ Tersedia |
| 4.3 | Ansible untuk k3s/On-Prem | ✅ Tersedia |

## Capaian Fase (Wajib)

- [ ] Menjelaskan control node, managed node, SSH, inventory, groups, variables, dan `ansible.cfg`.
- [ ] Menulis playbook YAML dengan task, module, handler, conditionals, loops, tags, dan error handling.
- [ ] Membuat playbook idempotent dan membuktikannya melalui check mode, rerun, serta review `changed`.
- [ ] Menyusun role dan collections dengan input/output contract yang jelas.
- [ ] Menggunakan Jinja2, variable precedence, template validation, dan handler secara aman.
- [ ] Menjelaskan boundary Ansible Vault, secret reference, redaction, dan artifact hygiene.
- [ ] Menggunakan `--check`, `--diff`, `--limit`, `serial`, serta approval untuk membatasi blast radius.
- [ ] Mengubah metadata non-secret OpenTofu menjadi inventory Ansible tanpa membawa credential.
- [ ] Menyusun host readiness gate, hardening dasar, rolling patching, dan runbook k3s.
- [ ] Menghasilkan evidence chain tanpa menyimpan secret, token, PAT, kubeconfig, atau raw output sensitif.

> Checklist materi tidak membuktikan execution. SSH, Ansible, Vault, CI, VM, hardening, dan k3s hanya boleh diklaim berhasil dengan evidence target, command, waktu, scope, dan output yang telah diredáksi.

## Rencana Belajar

| Hari | Materi | Praktik |
|---|---|---|
| 1 | [Modul 4.1](modul-4.1-fundamental-ansible/README.md) | Inventory, SSH preflight, ad-hoc, playbook pertama |
| 2 | Modul 4.1 | Idempotency, handlers, check mode, evaluasi |
| 3 | [Modul 4.2](modul-4.2-pola-produksi-ansible/README.md) | Role `common`, Jinja2, variable contract |
| 4 | Modul 4.2 | Vault boundary, CI validation, limit/diff, evaluasi |
| 5 | [Modul 4.3](modul-4.3-ansible-k3s-onprem/README.md) | OpenTofu handoff, readiness, k3s design |
| 6 | Modul 4.3 | Rolling patching, health/evidence, evaluasi |

## Dua Lane Praktik

### Static lane

```text
placeholder inventory → YAML/INI review → lint/syntax review → predicted changes
→ readiness matrix → evidence design
```

Gunakan lane ini bila `ansible-playbook`, SSH target, VM, atau k3s tidak tersedia. Static review bukan execution dan tidak boleh ditulis sebagai hasil playbook.

### Disposable runtime lane

```text
Mac ARM64 control node → VM lab yang disetujui → SSH preflight
→ --check/--diff + --limit → approval → playbook terbatas
→ rerun/idempotency → host readiness → k3s runbook
```

Jangan menjalankan mutation pada host yang tidak diketahui scope-nya. Runtime terakhir yang diketahui dari fase sebelumnya adalah `ansible-playbook=unavailable`; status harus diperbarui hanya setelah preflight nyata.

## Prasyarat

- Fase 0–3, khususnya Linux/SSH, networking, Git, k3s, operasi Kubernetes, dan Modul 3.3.
- Pemahaman YAML dasar, shell, systemd, package manager, dan CIDR.
- Mac Apple Silicon/ARM64; OrbStack VM boleh dipakai bila tersedia.
- Akses ke VM disposable dan user sudo hanya diperlukan untuk runtime lane.
- Tidak diperlukan credential nyata untuk static lane.

## Deliverables

1. Repository Ansible dengan inventory terpisah per environment, `ansible.cfg`, playbook, role `common`, dan role k3s yang dipin/ditinjau.
2. Checklist host readiness sebelum mutation.
3. Bukti desain idempotency: check mode, first run, rerun, dan alasan setiap `changed`.
4. Runbook hardening dan rolling patching dengan `serial`, stop condition, serta rollback.
5. Handoff metadata OpenTofu → inventory tanpa credential.
6. Evidence chain dari commit/module version sampai readiness dan cluster health.
7. Nilai kuis minimal **80%** pada setiap modul.

## Boundary Ownership

| Boundary | Owner | Catatan |
|---|---|---|
| OpenTofu/provider | VM, network, storage, metadata | Tidak mengonfigurasi OS harian atau menyimpan secret Ansible |
| Ansible | SSH, OS bootstrap, package, service, hardening, readiness | Tidak mengubah state provider secara tersembunyi |
| k3s/runbook | topology server/agent, join, quorum, upgrade, backup, health | Token/kubeconfig melalui secret mechanism terpisah |
| Helm/GitOps | aplikasi di atas cluster | Bukan bagian provisioning host |

## Guardrail Fase

- **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
- Jangan menulis password, private key, k3s token, kubeconfig, vault password, access key, decrypted Vault content, raw inventory secret, atau credential nyata ke Git, README, log, shell history, `group_vars`, artifact, atau evidence.
- Placeholder harus jelas: `<management-address>`, `<bootstrap-user>`, `<approved-k3s-version>`, `<secret-reference>`.
- Ansible Vault melindungi nilai terenkripsi saat disimpan; password Vault dan decrypted output tetap harus berada di secret mechanism yang disetujui.
- `--check` dan `--diff` tidak menjamin tanpa side effect. Review module behavior, target, `--limit`, dan approval terlebih dahulu.
- Redact task output dan gunakan `no_log: true` hanya untuk data sensitif yang memang perlu disembunyikan; jangan memakai redaction untuk menutupi kegagalan.
- Jangan menjalankan `kubectl delete -A`, `k3s server --cluster-reset`, restore snapshot pada cluster aktif, atau chaos pada production.
- `kubectl drain`, patch, firewall, SSH, upgrade, dan k3s install membutuhkan maintenance window, PDB/replica/quorum review, dan rollback decision.
- Runtime, CI, Vault, hardening, dan k3s harus diberi label **belum diverifikasi** bila evidence belum tersedia.

## Acceptance Criteria Fase

- [ ] Ketiga modul, enam lab, dan enam evaluasi tersedia serta saling terhubung.
- [ ] Contoh inventory/playbook/template tidak mengandung credential literal.
- [ ] Setiap mutation memiliki static lane, scope, check mode, limit, approval, dan stop condition.
- [ ] Handoff metadata mengandung hanya stable key, address, role, environment, version, dan reference non-secret.
- [ ] Idempotency dibedakan dari health dan dibuktikan dengan desain rerun.
- [ ] Rolling patching dan k3s readiness mencakup quorum, access path, network, time, storage, backup, dan rollback.
- [ ] Semua runtime yang tidak dieksekusi dilaporkan sebagai belum diverifikasi.

## Kaitan

- [Modul 3.3 — Konteks On-Prem & Provisioning](../fase-3-opentofu/modul-3.3-konteks-onprem/README.md) menyediakan metadata handoff dan boundary provider.
- [Modul 2.2 — k3s Production](../fase-2-kubernetes/modul-2.2-k3s-production/README.md) menyediakan topologi, server/agent, dan quorum.
- [Modul 2.4 — Operasi & Troubleshooting](../fase-2-kubernetes/modul-2.4-operasi-troubleshooting/README.md) menyediakan backup, drain, upgrade, context safety, dan evidence.
- Fase 5 — Helm masih menyusul; chart mengelola aplikasi setelah host dan cluster siap.

## Catatan SRE

Ansible mengurangi variasi konfigurasi, bukan menghapus failure domain. Setiap perubahan harus memiliki target yang benar, blast radius terukur, evidence, dan cara berhenti. Playbook yang sukses tanpa readiness dan health check belum membuktikan service production-ready.
