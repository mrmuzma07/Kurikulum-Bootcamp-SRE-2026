# Kuis dan Jawaban Modul 9.2

## Soal

1. Mengapa image production dipromosikan memakai digest?
2. Sebutkan urutan minimal evidence delivery.
3. Apa boundary CI dan ArgoCD pada production path?
4. Sebutkan metadata wajib minimal untuk alert meaningful.
5. Mengapa `Firing` tidak membuktikan notification diterima?
6. Apa yang harus dimiliki SLO contract?
7. Mengapa label metrics harus bounded?
8. Bedakan Kubernetes object backup, PV/application backup, dan etcd snapshot.
9. Mengapa Velero `Completed` bukan bukti recovery?
10. Apa syarat runtime restore evidence?

## Jawaban

1. Digest immutable mengikat deployment pada artifact spesifik dan mencegah mutable tag berubah diam-diam.
2. Source commit, CI checks, digest/provenance, GitOps commit, Argo revision, rollout, smoke/traffic, telemetry, dan SLO decision.
3. CI memvalidasi dan mempublikasikan artifact/desired state; ArgoCD mereconcile approved GitOps revision. CI tidak memakai kubeconfig production atau manual apply.
4. Owner, severity, query reference, evaluation window, missing-data policy, runbook, route, notification boundary, dan evidence receipt.
5. Rule evaluation dan Alertmanager route dapat gagal atau tertahan; receipt boundary harus diverifikasi terpisah.
6. Service, objective, window, numerator, denominator, eligible traffic, exclusions, missing-data policy, owner, dan review cadence.
7. Unbounded user/request/raw-input labels menyebabkan cardinality dan cost meningkat serta dapat membocorkan data.
8. Objects memulihkan metadata; PV/application memulihkan data dengan consistency; etcd memulihkan control-plane state. Ketiganya memiliki recovery caveat berbeda.
9. Completed hanya menunjukkan backup job/artifact selesai, bukan isolated restore, dependency, endpoint, telemetry, atau SLO recovery.
10. Target disposable terisolasi, approval, restore order, object/data/endpoint/telemetry validation, measured RPO/RTO, cleanup, dan redacted evidence.

## Penilaian

Soal 1–10 bernilai 4 poin, total 40. Lulus minimal 32/40 tanpa pelanggaran guardrail.

## Kaitan

Lanjut ke [Modul 9.3 — Game Day dan Graduation](../../modul-9.3-game-day-graduation/README.md).
