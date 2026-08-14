# Latihan Modul 6.2 — ArgoCD

## Petunjuk

Semua latihan dapat dikerjakan static. Runtime hanya pada disposable scope yang disetujui. Nilai minimum lulus 80/100. Pelanggaran secret/destructive guardrail berarti tidak lulus otomatis.

## Latihan 1 — Application dan project (25 poin)

Rancang AppProject dan Application untuk staging. Batasi repository, cluster, namespace, resource kind, source revision, dan sync mode. Jelaskan mengapa production memerlukan gate berbeda.

## Latihan 2 — ApplicationSet (20 poin)

Buat desain generator cluster berbasis label yang hanya memilih target dengan `sre.managed=true` dan environment yang disetujui. Jelaskan name collision, namespace ownership, dan recovery saat generator salah memilih.

## Latihan 3 — Drift dan self-heal (20 poin)

Analisis tiga drift: manual replica edit, webhook label mutation, dan resource yang dihapus dari Git. Tentukan revert, ignore field, self-heal, atau abort/prune review untuk tiap kasus.

## Latihan 4 — Secret boundary (20 poin)

Bandingkan External Secrets, Sealed Secrets, dan SOPS + age. Bahas key location, decryption point, rotation, backup, recovery, audit, dan evidence redaction tanpa menulis ciphertext/secret nyata.

## Latihan 5 — Health evidence (15 poin)

Buat chain Argo revision → rollout → smoke test → telemetry → SLO. Jelaskan mengapa `Synced`, `Healthy`, atau self-heal saja tidak cukup.

## Rubrik

- 90–100: boundary, deletion safety, dan recovery lengkap.
- 80–89: acceptance terpenuhi dengan koreksi kecil.
- 60–79: remediation diperlukan.
- <60: belum lulus.
- Credential plaintext, decrypted secret, manual production edit, atau broad deletion: **tidak lulus otomatis**.

## Pengumpulan

Kumpulkan YAML placeholder, tabel policy, drift decision matrix, dan evidence template redacted. Runtime yang tidak dijalankan harus diberi status **belum diverifikasi**.
