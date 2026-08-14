# Latihan Modul 9.2 — Delivery, Observability, dan Recovery

Gunakan placeholder dan summary redacted. Jangan menulis token, webhook, password, kubeconfig, raw alert payload, atau backup archive.

## Latihan

1. Gambar evidence chain dari source commit sampai SLO decision.
2. Buat contoh record yang mengikat image digest, provenance, GitOps commit, dan Argo revision.
3. Jelaskan perbedaan staging k3d dan production k3s dalam promotion policy.
4. Rancang lima alert meaningful lengkap dengan owner, severity, query, window, missing-data policy, runbook, route, dan receipt.
5. Buat SLO contract dengan numerator, denominator, eligible traffic, exclusion, dan error budget.
6. Jelaskan mengapa Argo `Healthy` tidak sama dengan SLO sehat.
7. Buat restore order untuk Kubernetes objects, PV/application, dan etcd.
8. Tulis decision tree saat alert `Firing` tetapi notification tidak diterima.
9. Jelaskan caveat rollback ketika release memiliki database migration.
10. Buat static/runtime evidence matrix untuk Velero `Completed`, isolated restore, endpoint, telemetry, dan RPO/RTO.

## Tugas Refleksi

- Signal mana yang paling berisiko false confidence?
- Bagaimana redaction, bounded labels, sampling, dan retention diterapkan?
- Evidence apa yang hilang jika CI runner atau registry tidak tersedia?

## Penilaian

Setiap soal 4 poin, total 40. Minimal 32/40 dan tanpa guardrail violation. Mengklaim promotion/restore/paging tanpa execution evidence berarti gagal otomatis.

## Lanjut

Cek [kuis dan jawaban](kuis-dan-jawaban.md), lalu lanjut ke [Modul 9.3](../../modul-9.3-game-day-graduation/README.md).
