# Latihan Modul 9.3 — Game Day dan Graduation

Gunakan scenario sintetis atau disposable target yang telah disetujui. Jangan menyimpan raw log, PII, credential, kubeconfig, atau raw alert payload.

## Latihan

1. Buat incident command card dengan primary/secondary on-call, IC, communications lead, dan operations lead.
2. Buat matrix tujuh Game Day scenario dari kurikulum dengan signal, runbook, mitigation, stop condition, dan post-check.
3. Tulis timeline UTC untuk bad release dan ukur definisi rollback <5 menit.
4. Buat decision tree worker down yang memulai dari read-only checks.
5. Jelaskan mengapa latency 10x harus dianalisis melalui trace/dependency, bukan hanya restart workload.
6. Susun postmortem blameless dengan contributing factors, missing signal, dan action verification.
7. Buat graduation evidence matrix untuk architecture, rebuild, delivery, observability, recovery, dan operations.
8. Tentukan kapan keputusan `ready`, `conditional`, dan `not ready` dipilih.
9. Tulis stop condition untuk disk full, certificate expired, dan MetalLB/ARP conflict.
10. Rancang action-item register dengan owner, due date UTC, risk, expiry, dan verification.

## Tugas Refleksi

- Apa perbedaan tabletop evidence dan runtime evidence?
- Mengapa postmortem tidak boleh menjadi tempat menyimpan raw incident data?
- Gap apa yang otomatis mencegah `ready`?

## Penilaian

Setiap soal 4 poin, total 40. Minimal 32/40 dan tanpa pelanggaran guardrail. Failure injection production, credential exposure, atau `ready` tanpa evidence wajib berarti gagal otomatis.

## Lanjut

Bandingkan dengan [kuis dan jawaban](kuis-dan-jawaban.md) dan lakukan [LAB-02 destroy/rebuild graduation](../lab/LAB-02-destroy-rebuild-graduation.md).
