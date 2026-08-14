# Kuis dan Jawaban Modul 5.2

## Petunjuk

Pilih jawaban 1–20 dan jawab esai 21–30. **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Pilihan Ganda

1. `values.schema.json` membantu ... A. validasi input B. menyimpan token C. backup PVC D. membuat node
2. `values-prod.yaml` boleh memuat ... A. capacity non-secret B. password C. kubeconfig D. registry PAT
3. Selector Deployment yang berubah dapat ... A. memicu masalah immutable/availability B. mengenkripsi Pod C. memperbaiki DNS D. membuat backup
4. Image production sebaiknya ... A. immutable digest B. latest C. tanpa tag D. token tag
5. PDB dipakai untuk ... A. availability saat disruption B. chart package C. registry auth D. schema parse
6. Hook memiliki ... A. event, weight, deletion policy B. password otomatis C. SLO otomatis D. state provider
7. Migration hook harus ... A. compatibility/recovery review B. selalu dijalankan C. disembunyikan D. menghapus PV
8. `helm test` membuktikan ... A. test release tertentu B. SLO semua kondisi C. backup valid D. security total
9. NOTES sebaiknya ... A. safe instruction non-secret B. mencetak kubeconfig C. mencetak token D. mencetak password
10. CRD lifecycle ... A. perlu review terpisah B. selalu rollback sempurna C. sama dengan ConfigMap D. tidak relevan
11. `Chart.lock` membantu ... A. dependency reproducibility B. API access C. secret rotation D. PDB
12. OCI promotion sebaiknya memakai ... A. immutable artifact reference B. rebuild diam-diam C. latest D. raw token
13. Resource footprint observability mencakup ... A. compute/storage/retention B. password C. SSH key D. chart name saja
14. Hook failure harus ... A. diklasifikasi dan menghentikan promotion B. disembunyikan C. reset cluster D. lanjut massal
15. NetworkPolicy default deny dapat ... A. memutus dependency jika allowlist salah B. memperbaiki image C. membuat release immutable D. backup
16. `--atomic` tidak menjamin ... A. data migration rollback B. release action C. wait timeout D. revision
17. Evidence aman memuat ... A. version/target/status redacted B. raw Secret C. registry token D. kubeconfig
18. Production values harus ... A. melewati schema/render review B. bypass lint C. di-commit credential D. selalu auto-approve
19. Observability chart aktif berarti ... A. perlu cluster/tool/evidence B. lint hijau C. package ada D. README selesai
20. Promotion harus berhenti bila ... A. compatibility/health belum terverifikasi B. chart version jelas C. approval ada D. digest immutable

## Kunci

1 A, 2 A, 3 A, 4 A, 5 A, 6 A, 7 A, 8 A, 9 A, 10 A, 11 A, 12 A, 13 A, 14 A, 15 A, 16 A, 17 A, 18 A, 19 A, 20 A.

## Esai dan Panduan

21. Rancang values/schema per environment.
22. Jelaskan selector, image, probe, resource, dan PDB contract.
23. Rancang hook yang idempotent dan observable.
24. Bedakan `helm test` dengan SLO evidence.
25. Analisis CRD dan migration saat rollback.
26. Susun supply-chain evidence untuk OCI.
27. Rancang chart security review.
28. Hitung concern resource/storage observability.
29. Susun rollback stop condition.
30. Jelaskan evidence minimum untuk promotion production.

## Penilaian

Pilihan ganda 40, esai 60, lulus minimal 80. Pelanggaran guardrail secret/destructive = tidak lulus.
