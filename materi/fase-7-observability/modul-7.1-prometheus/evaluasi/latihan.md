# Latihan Prometheus

## Tugas

1. 1. Dasar metric dan exporter: tulis ringkasan konsep, ownership, failure mode, dan evidence field.
2. 2. PromQL, rules, dan Alertmanager: buat contoh non-secret atau query/config static dengan placeholder.
3. LAB-01 scrape: lengkapi Lane A static dan acceptance criteria.
4. LAB-02 rules: susun Lane B optional disposable dengan context verification, approval, stop condition, rollback, cleanup, dan evidence redacted.
5. Jelaskan mengapa status controller/config/render tidak otomatis membuktikan data, notification, atau SLO.

## Guardrail

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Jangan menyertakan PII, credential, raw telemetry payload, raw state, rendered Secret, atau notification secret.

## Rubrik

| Aspek | Bobot |
|---|---:|
| Konsep dan boundary | 25 |
| Design/config/query non-secret | 25 |
| Failure/capacity/security reasoning | 20 |
| Runtime evidence dan status honesty | 20 |
| Guardrail dan clarity | 10 |

Nilai minimal **80/100** dan tidak ada pelanggaran guardrail. Jika runtime tidak tersedia, static lane tetap dinilai; runtime dicatat **belum diverifikasi**, bukan dibuat-buat.

## Kaitan

Gunakan README modul dan materi teori sebagai referensi. Hubungkan hasil ke modul berikutnya dan Fase 6 GitOps/Fase 8 SRE bila relevan.
