# Kuis dan Jawaban Modul 6.3

## Soal

1. Mengapa image digest diperlukan dalam promotion?
2. Apa urutan evidence dari source sampai SLO?
3. Apa perbedaan GitOps commit dan Argo revision?
4. Sebutkan tujuh kelas failure end-to-end.
5. Mengapa pipeline green tidak membuktikan health?
6. Apa syarat approval production?
7. Apa caveat Git revert terhadap database migration?
8. Apa perbedaan canary dan blue-green?
9. Mengapa canary membutuhkan stable/canary service atau traffic boundary?
10. Apa policy aman saat analysis metrics `NoData`?
11. Mengapa capacity harus direview pada blue-green?
12. Apa yang harus dilakukan bila rollout stuck?
13. Mengapa Argo Healthy bukan SLO?
14. Apa evidence minimum untuk menyatakan promotion runtime?
15. Mengapa CI sebaiknya tidak memiliki kubeconfig production?

## Jawaban

1. Digest immutable mengikat promotion pada artifact spesifik dan mencegah tag berubah diam-diam.
2. Commit, pipeline/job/test, digest/scan, GitOps commit, Argo revision, rollout/readiness, smoke/telemetry, dan SLO decision.
3. GitOps commit adalah desired-state change; Argo revision adalah revision yang dibaca/reconcile controller pada target.
4. CI, registry, manifest, Argo sync, Kubernetes rollout, application health, dan observability/SLO.
5. Green hanya membuktikan job/scope CI berhasil, bukan deployment, traffic, dependency, telemetry, atau SLO.
6. Artifact/compatibility, GitOps diff, staging evidence, telemetry window, rollback, owner, protected environment, dan approved change window.
7. Revert code tidak otomatis membalikkan data/schema migration atau external side effect.
8. Canary menaikkan traffic/capacity bertahap; blue-green menyiapkan environment kandidat lalu switch active service.
9. Agar traffic dan health kandidat dapat diisolasi, diukur, dipause, atau diabort.
10. Pause/stop dan investigate; jangan menganggap NoData sebagai sukses.
11. Dua environment/versi dapat membutuhkan resource hampir dua kali lipat dan memengaruhi scheduling/quota.
12. Pause/stop, klasifikasikan scheduling/image/readiness/dependency, ambil evidence redacted, dan jangan promote.
13. Healthy adalah assessment resource/controller, sedangkan SLO memerlukan metric, window, traffic, dan objective.
14. Commit/job/digest/GitOps/Argo revision, target, approval, rollout/readiness, telemetry/SLO outcome, waktu, dan evidence redacted.
15. Untuk membatasi blast radius; CI mempublikasikan artifact/reviewed desired state, Argo yang reconcile dengan boundary-nya.

## Penilaian

Soal 1–10 bernilai 2 poin, soal 11–15 bernilai 4 poin. Total 40 poin; lulus minimal 32 poin tanpa pelanggaran guardrail.

## Kaitan

Setelah lulus, peserta siap menghubungkan GitOps evidence dengan [Fase 7 Observability](../../../fase-7-observability/README.md) dan praktik reliability Fase 8.
