# Kuis dan Jawaban Modul 4.2

## Petunjuk

Pilih satu jawaban 1–20 dan jawab esai 21–30. **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Pilihan Ganda

1. Role defaults adalah ... A. input default B. secret store C. state provider D. kubeconfig
2. Collection perlu dipin untuk ... A. reproducibility B. membuka firewall C. bypass lint D. mengganti SSH
3. Template Jinja2 harus ... A. divalidasi B. berisi token C. dicetak debug D. tidak punya defaults
4. `no_log` digunakan untuk ... A. task sensitif B. semua task C. menyembunyikan failure D. mengganti audit
5. `--diff` berisiko karena ... A. dapat menampilkan isi konfigurasi B. selalu gagal C. menghapus host D. mengubah quorum
6. `serial: 1` berarti ... A. satu batch host B. satu task total C. encryption D. backup
7. `block/rescue` ... A. menangani failure terkontrol B. menjamin rollback C. selalu lanjut D. menyimpan secret
8. CI syntax-check membuktikan ... A. parse/playbook syntax B. host health C. k3s quorum D. production readiness
9. Vault memberi ... A. encrypted-at-rest workflow B. akses otomatis C. rotation otomatis D. health check
10. Artifact CI harus ... A. redacted/minimum B. raw verbose C. memuat decrypted secret D. public
11. Role contract mencakup ... A. input/side effect/rollback B. password C. kubeconfig D. API reset
12. `--limit` mengontrol ... A. target scope B. variable precedence C. YAML parser D. etcd
13. Handler berjalan ... A. saat notify dan perubahan B. selalu C. sebelum task D. hanya di CI
14. `molecule` adalah ... A. test harness konseptual B. secret manager C. hypervisor D. ingress
15. Flag firewall default false ... A. menghindari side effect awal B. hardening selesai C. membuka port D. bukti safety
16. Lint hijau berarti ... A. lint lulus, bukan integration B. host sehat C. k3s siap D. backup valid
17. Protected secret injection harus ... A. melalui mekanisme disetujui B. commit C. debug D. plain text
18. Variable override berisiko perlu ... A. review B. diabaikan C. dihapus state D. apply otomatis
19. Partial failure harus ... A. dihentikan dan diklasifikasi B. retry semua C. disembunyikan D. reset cluster
20. Role eksternal perlu ... A. provenance/version review B. dipercaya otomatis C. token di README D. tanpa pin

## Kunci

1 A, 2 A, 3 A, 4 A, 5 A, 6 A, 7 A, 8 A, 9 A, 10 A, 11 A, 12 A, 13 A, 14 A, 15 A, 16 A, 17 A, 18 A, 19 A, 20 A.

## Esai dan Panduan

21. Rancang role contract.
22. Jelaskan variable precedence yang dapat diaudit.
23. Mengapa `no_log` bukan pengganti observability?
24. Rancang template non-secret dengan validation.
25. Susun CI gate dan approval.
26. Bedakan Vault encryption dengan secret manager/rotation.
27. Analisis diff yang mengandung secret.
28. Rancang recovery partial failure.
29. Mengapa `serial` tidak menjamin availability?
30. Tentukan stop condition sebelum role production.

## Penilaian

Pilihan ganda 40, esai 60, lulus minimal 80. Pelanggaran guardrail = tidak lulus.
