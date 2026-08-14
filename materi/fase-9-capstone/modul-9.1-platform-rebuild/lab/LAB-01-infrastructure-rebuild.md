# LAB-01 — Infrastructure Rebuild pada Disposable Scope

> **Target:** membuktikan contract OpenTofu → Ansible → k3s melalui target disposable yang dapat dihancurkan dan dibangun ulang tanpa menyimpan secret atau raw state.

## Mode Lab

Lab dapat diselesaikan sebagai **static simulation** bila tool/runtime belum tersedia. Static submission harus memuat plan contract, target allowlist, approval record sintetis, evidence schema, dan daftar gap dengan label `belum diverifikasi`. Jangan mengubah dokumen menjadi klaim runtime.

Runtime hanya boleh dilakukan pada disposable target yang benar-benar diidentifikasi, bukan production atau workstation yang menyimpan data penting.

## Prasyarat

- Modul 9.1 teori dan [Fase 4 — Ansible](../../../fase-4-ansible/README.md) selesai.
- Repository capstone memiliki revision yang direview.
- Target allowlist, owner, maintenance window, backup, dan recovery path tersedia.
- Resource OrbStack dan akses bastion telah diverifikasi read-only.
- Tidak ada credential yang ditulis pada repository, command line, log, atau evidence.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menjalankan `destroy` pada target ambigu. Jangan memakai `-auto-approve` sebagai default. Jangan mencetak raw plan/state, kubeconfig, token, private key, password, registry credential, atau backup archive. `--check` dan `--diff` bukan jaminan zero side effect.

## Evidence Contract

Buat file lokal di luar repository atau evidence redacted dengan field berikut:

```yaml
lab: LAB-01
revision: <git-revision>
target_allowlist: [<disposable-target>]
operator_role: <role>
window_utc: <start-end>
approval_ref: <approval-id>
preflight: <pass-fail-unknown>
plan_summary: <resource-counts-only>
action_summary: <redacted-status>
postcheck: <domain-summary>
cleanup: <pass-fail-unknown>
known_gaps: [<gap>]
```

## Langkah

### 1. Tetapkan Scope dan Approval

Tulis target, workspace, owner, tujuan rebuild, resource yang boleh berubah, resource yang dilarang, stop condition, rollback/recovery, dan communication boundary. Cocokkan target dengan allowlist sebelum membuka state atau menjalankan action.

Approval harus menyatakan bahwa target disposable, backup/recovery tersedia, maintenance window aktif, dan blast radius dipahami. Bila salah satu tidak ada, berhenti di static review.

### 2. Read-Only Preflight

Catat summary dari:

- revision dan working tree;
- platform/tool version yang tersedia;
- workspace/backend/lock status tanpa mengungkap path credential;
- VM/network/storage target;
- OrbStack resource budget;
- Ansible inventory reachability dan access recovery;
- desired topology, existing topology, dan drift;
- backup identifier/checksum summary;
- cleanup plan.

Context kosong, target mismatch, lock stale, atau bastion tidak reachable berarti **STOP**. Jangan melanjutkan dengan asumsi.

### 3. Review Plan OpenTofu

Gunakan workflow proyek yang telah disetujui untuk membuat plan. Simpan output plan sementara pada lokasi terlindungi; simpan hanya counts/summary yang telah direduksi ke evidence. Review create/update/destroy, dependency, IP allocation, storage, and external side effect bersama peer.

Pertanyaan review:

- Apakah semua destroy berada di disposable allowlist?
- Apakah dedicated MetalLB pool tidak overlap?
- Apakah resource headroom dan recovery path cukup?
- Apakah state/lock dan backup dapat dipulihkan?
- Apakah output yang diteruskan ke Ansible non-secret?

### 4. Apply Terbatas dan Handoff

Setelah approval, jalankan apply pada reviewed scope sesuai prosedur repository. Jangan menyalin token atau kubeconfig ke output. Serahkan metadata non-secret ke Ansible, lalu jalankan playbook dengan `--limit <approved-scope>` dan serial strategy yang disepakati.

Ansible `changed=0` bukan bukti host siap. Catat OS, network, disk, time sync, access, k3s prerequisite, dan failure summary per domain.

### 5. Verifikasi Before/After

Gunakan read-only checks untuk membandingkan:

- VM/node count dan role;
- network address/range dan DNS summary;
- storage class/capacity summary;
- k3s API/quorum/node readiness;
- MetalLB prerequisite;
- access recovery;
- expected drift versus unexpected drift.

`kubectl get nodes` hanya satu signal. Bila readiness domain unknown, handoff berstatus `conditional` atau `not ready`.

### 6. Cleanup atau Destroy/Rebuild

Jika exercise memilih destroy, ulangi target allowlist, backup, plan/diff, approval, dan maintenance window. Jalankan destroy hanya terhadap disposable scope; verifikasi resource cleanup tanpa raw output. Kemudian rebuild dari repository dan ulangi post-check.

Jika recovery gagal atau resource di luar scope tersentuh, hentikan action, preserve redacted evidence, dan eskalasi. Jangan mencoba cluster reset aktif atau perintah mass-delete.

### 7. Tutup Evidence

Isi evidence contract, bandingkan desired/as-built, catat elapsed time hanya bila diukur, dan tulis known gaps. Submit review ke peer/mentor. Runtime outcome tetap `belum diverifikasi` jika langkah runtime tidak benar-benar dieksekusi.

## Acceptance Criteria

- [ ] Target disposable, allowlist, owner, approval, window, backup, dan stop condition jelas.
- [ ] Preflight read-only dan plan/diff review selesai atau gap dicatat.
- [ ] Handoff OpenTofu → Ansible hanya memakai metadata non-secret.
- [ ] Apply/destroy tidak memakai `-auto-approve` default dan dibatasi scope.
- [ ] Before/after readiness, cleanup, recovery, dan known gap tercatat.
- [ ] Tidak ada secret, raw state/plan, kubeconfig, atau raw archive dalam repository/evidence.
- [ ] Runtime result diberi label **terverifikasi** hanya bila execution evidence lengkap; jika tidak, **belum diverifikasi**.

## Troubleshooting

| Gejala | Tindakan |
|---|---|
| Plan menyentuh VM yang tidak ada di allowlist | Stop; jangan apply; perbaiki workspace/scope dan minta review ulang |
| Lock backend tidak sehat | Stop; ikuti recovery lock resmi; jangan menghapus lock secara paksa |
| Ansible `unreachable` | Verifikasi bastion, DNS, routing, dan access recovery secara read-only |
| Resource OrbStack tidak cukup | Stop sebelum apply; revisi kapasitas atau scope dengan approval |
| Rebuild menghasilkan drift | Bandingkan revision, provider input non-secret, inventory, dan as-built; jangan manual edit production |

## Lanjut

Lanjutkan handoff dan cluster readiness ke [LAB-02](LAB-02-k3s-bootstrap-handoff.md), lalu delivery ke [Modul 9.2](../../modul-9.2-delivery-observability/README.md).
