# Kuis dan Jawaban Modul 6.1

## Soal

1. Mengapa `needs` penting selain `stages`?
2. Apa perbedaan artifact dan cache?
3. Mengapa `latest` tidak cocok sebagai promotion reference?
4. Apa boundary aman untuk registry credential?
5. Mengapa runner ARM64 tidak otomatis menghasilkan image AMD64?
6. Apa yang harus dibuktikan sebelum job publish dijalankan?
7. Mengapa raw OpenTofu plan/state tidak boleh menjadi artifact terbuka?
8. Apa fungsi `--limit` pada Ansible?
9. Mengapa `--check --diff` bukan jaminan tanpa side effect?
10. Apa arti pipeline green dan apa yang tidak dibuktikannya?
11. Kapan `resource_group` diperlukan?
12. Apa tindakan bila plan berbeda setelah approval?
13. Mengapa dotenv dibatasi pada metadata non-secret?
14. Sebutkan tiga stop condition untuk pipeline IaC.
15. Apa evidence minimal untuk menghubungkan commit dengan image?

## Kunci Jawaban

1. `needs` membentuk dependency graph eksplisit, memungkinkan job yang tidak saling bergantung berjalan paralel dan mengontrol artifact transfer.
2. Artifact adalah output job yang ditelusuri dengan retention/permission; cache hanya akselerasi, dapat stale atau tidak terpercaya, dan bukan source of truth.
3. `latest` mutable sehingga isi dapat berubah tanpa mengubah referensi; commit SHA/digest lebih reproducible dan dapat diaudit.
4. Protected/masked variable atau short-lived workload identity pada environment/runner yang disetujui; tidak di YAML, log, artifact, atau shell argument.
5. Host architecture, builder, target platform, dan manifest harus dipilih eksplisit; build lokal ARM64 dapat menghasilkan hanya ARM64.
6. Source protected, rules benar, approval tersedia, target/environment benar, artifact immutable, dan evidence job/commit dapat ditelusuri.
7. Plan/state dapat memuat nilai sensitif dan detail infrastruktur; simpan pada backend/access boundary yang disetujui, bukan artifact terbuka.
8. Membatasi host target sehingga blast radius Ansible dapat direview dan tidak meluas ke host lain.
9. Module tertentu tetap melakukan koneksi atau side effect; mode check hanya simulasi parsial.
10. Job pada scope tertentu berhasil; tidak membuktikan deployment, readiness, application health, atau SLO.
11. Saat target environment/state/registry harus serialized agar retry atau pipeline paralel tidak menghasilkan dua perubahan bersaing.
12. Stop, bandingkan commit/state/provider/plan reference, lalu minta approval ulang; jangan apply blind.
13. Agar metadata dapat diteruskan tanpa membawa credential atau raw secret ke downstream job/artifact.
14. Contoh: target tidak sesuai, lock gagal, plan berubah, inventory di luar limit, credential muncul di log, atau recovery tidak tersedia.
15. Commit SHA, pipeline ID/job ID, test summary, image digest, platform, dan status redacted.

## Penilaian

Setiap jawaban benar bernilai 2 poin; soal 15 bernilai 6 poin. Total 34 poin, lulus minimal 27 poin dan tidak ada pelanggaran guardrail.

## Kaitan

Gunakan hasil kuis bersama [Modul 6.2](../../modul-6.2-argocd/README.md) untuk memahami bagaimana artifact menjadi desired state melalui GitOps.
