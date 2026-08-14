# Latihan Modul 6.1 — GitLab CI/CD

## Petunjuk

Kerjakan pada repository latihan disposable. Semua nilai credential, endpoint privat, token, kubeconfig, raw state, dan raw plan harus diganti placeholder. Nilai minimum lulus: 80/100 dan tidak ada pelanggaran secret/destructive guardrail.

## Latihan 1 — Pipeline graph (25 poin)

Buat graph lint → test → template validation → build multi-arch → scan/sign/provenance → publish digest → reviewed GitOps change. Jelaskan `needs`, `rules`, `workflow`, protected branch, manual gate, dan `resource_group`.

**Kriteria:** graph 10, rules 5, gate 5, evidence 5.

## Latihan 2 — Runner, artifact, cache (20 poin)

Bandingkan shared runner dengan self-hosted OrbStack ARM64. Tentukan artifact/cache untuk test summary, SBOM, digest, dependency cache, raw log, kubeconfig, dan rendered Secret.

**Kriteria:** architecture 6, isolation 5, classification 5, retention/redaction 4.

## Latihan 3 — OpenTofu dan Ansible (25 poin)

Tulis desain job MR plan dan protected apply, lalu job Ansible lint/syntax/check/diff dengan `--limit` dan serial. Sertakan stop conditions, state lock, approval, dan handoff metadata non-secret.

**Kriteria:** OpenTofu 8, Ansible 8, approval 5, safety 4.

## Latihan 4 — Incident review (15 poin)

Sebuah job publish terpicu dari fork, menggunakan cache yang dibuat untrusted MR, lalu artifact menyimpan log environment lengkap. Tulis containment, credential response (tanpa menyebut nilai credential), evidence yang disimpan, dan perbaikan pipeline.

**Kriteria:** containment 5, secret response 4, evidence 3, prevention 3.

## Latihan 5 — Promotion boundary (15 poin)

Jelaskan mengapa pipeline green dan image tersedia tidak membuktikan deployment, readiness, health, maupun SLO. Rancang handoff ke GitOps repository dengan image digest dan reviewer.

**Kriteria:** boundary 7, immutable reference 4, reviewer/evidence 4.

## Rubrik umum

- 90–100: production reasoning lengkap, boundary dan recovery jelas.
- 80–89: memenuhi acceptance dengan koreksi kecil.
- 60–79: perlu remediasi dan pengulangan lab.
- <60: belum lulus.
- Menyimpan credential nyata, mencetak token/kubeconfig, atau melakukan mutation tanpa scope/approval: **tidak lulus otomatis**.

## Evidence Pengumpulan

Kumpulkan diagram, snippet non-secret, tabel artifact/cache, checklist approval, dan evidence template yang telah di-redact. Runtime tidak wajib; jika tidak dijalankan, tulis **belum diverifikasi**.
