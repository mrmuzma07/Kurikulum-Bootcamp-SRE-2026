# Kuis dan Jawaban Modul 6.2

## Soal

1. Apa fungsi AppProject?
2. Apa fungsi ApplicationSet?
3. Mengapa generator harus memiliki selector allowlist?
4. Apa perbedaan `Synced` dan `Healthy`?
5. Mengapa `Healthy` bukan bukti SLO?
6. Sebutkan sumber drift.
7. Kapan `ignoreDifferences` berbahaya?
8. Mengapa prune memerlukan review ownership?
9. Apa yang tidak dapat dibalikkan self-heal?
10. Sebutkan tiga secret pattern yang aman secara boundary.
11. Mengapa ciphertext SOPS bukan seluruh solusi keamanan?
12. Mengapa `kubectl edit` hanya boleh pada disposable target?
13. Apa yang dilakukan saat ApplicationSet memilih target salah?
14. Apa bukti minimum untuk menyatakan self-heal terverifikasi?
15. Mengapa app repo dan GitOps repo dipisahkan?

## Jawaban

1. Membatasi repository, destination, namespace, resource kind, dan policy yang dapat dikelola Application.
2. Menghasilkan Application dari generator yang terkontrol untuk banyak cluster/environment.
3. Agar generator tidak memilih cluster/namespace di luar ownership dan blast radius yang disetujui.
4. `Synced` membandingkan desired/live state; `Healthy` menilai health resource berdasarkan ArgoCD health assessment.
5. SLO membutuhkan telemetry, traffic, error rate, latency, dan window evaluasi yang tidak tercakup status controller.
6. Manual edit, admission/webhook, controller defaulting, operator, dependency, dan field ownership.
7. Saat dipakai untuk menyembunyikan drift penting atau menutupi konflik controller.
8. Karena resource yang hilang dari desired state dapat dianggap harus dihapus, termasuk data-bearing resource.
9. Migration, external side effect, database schema/data, PVC deletion, dan hook side effect.
10. External Secrets, Sealed Secrets, SOPS + age dengan key di luar Git (sesuai policy).
11. Key management, rotation, backup, recovery, access audit, dan decryption boundary tetap diperlukan.
12. Agar eksperimen drift tidak mengubah production atau memicu side effect yang tidak dapat dipulihkan.
13. Hentikan sync, audit selector/labels/project, batasi destination, dan pulihkan melalui reviewed change.
14. Target disposable, pre/post state, resource/revision, correction event, health outcome, waktu, dan evidence redacted.
15. Agar CI aplikasi tidak memerlukan kubeconfig production dan promotion menjadi reviewed desired-state change.

## Penilaian

Soal 1–10 bernilai 2 poin, soal 11–15 bernilai 4 poin. Total 40 poin; lulus minimal 32 poin tanpa pelanggaran guardrail.

## Kaitan

Kuis ini menjadi dasar [Modul 6.3 end-to-end flow](../../modul-6.3-end-to-end-flow/README.md).
