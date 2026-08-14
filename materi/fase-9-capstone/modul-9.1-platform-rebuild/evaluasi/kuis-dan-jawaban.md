# Kuis dan Jawaban Modul 9.1

## Soal

1. Apa ownership OpenTofu pada Capstone?
2. Mengapa handoff OpenTofu → Ansible harus menjadi metadata contract?
3. Sebutkan minimal empat item preflight sebelum destroy.
4. Mengapa `-auto-approve` bukan default yang aman?
5. Apa yang harus dilakukan jika plan menyentuh target di luar allowlist?
6. Mengapa quorum dan PDB perlu direview sebelum bootstrap/maintenance?
7. Apa perbedaan desired topology dan as-built topology?
8. Mengapa `sensitive = true` bukan pengganti encryption state?
9. Sebutkan domain readiness selain node `Ready`.
10. Kapan hasil rebuild boleh disebut runtime terverifikasi?

## Jawaban

1. OpenTofu memiliki VM, network, storage metadata, state/lock, dan lifecycle disposable; bukan package OS, workload, atau secret value.
2. Contract menjaga boundary, auditability, repeatability, dan memastikan hanya metadata non-secret diteruskan ke Ansible.
3. Target allowlist, workspace/lock, owner/approval, backup/recovery, access recovery, resource/network/storage, maintenance window, dan cleanup plan.
4. Karena apply/destroy yang tidak direview dapat mengubah atau menghapus resource di luar maksud operator; approval harus eksplisit.
5. Stop, jangan apply/destroy, cocokkan workspace/target, regenerate plan, dan minta review/approval ulang.
6. Quorum menjaga control-plane availability; PDB menjaga availability workload saat perubahan dan drain.
7. Desired adalah kontrak repository; as-built adalah hasil aktual yang diverifikasi dari environment.
8. `sensitive` hanya menyamarkan presentasi; state tetap memerlukan backend protection, encryption, access control, backup, dan key custody.
9. API/quorum, CNI/DNS, scheduling/capacity, storage, MetalLB/Ingress, access recovery, time/disk, dan dependency readiness.
10. Hanya setelah target disposable, approval, before/after health, handoff, cleanup/recovery, dan evidence redacted lengkap; tanpa itu status **belum diverifikasi**.

## Penilaian

Soal 1–10 bernilai 4 poin, total 40. Lulus minimal 32/40 tanpa pelanggaran guardrail.

## Kaitan

Lanjut ke [Modul 9.2 — Delivery, Observability, Recovery](../../modul-9.2-delivery-observability/README.md).
