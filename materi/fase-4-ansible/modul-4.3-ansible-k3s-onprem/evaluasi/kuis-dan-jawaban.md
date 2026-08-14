# Kuis dan Jawaban Modul 4.3

## Petunjuk

Pilih jawaban 1–20 dan jawab esai 21–30. **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Pilihan Ganda

1. OpenTofu terutama memiliki ... A. VM/network/storage lifecycle B. package harian C. PDB D. kubeconfig
2. Ansible terutama memiliki ... A. host configuration B. etcd quorum C. hypervisor state D. Helm chart
3. Readiness `not-verified` berarti ... A. stop B. lanjut C. reset D. ignore
4. Metadata handoff boleh memuat ... A. role/address/version B. token C. private key D. password
5. k3s server bootstrap harus dilakukan setelah ... A. readiness B. plan belum review C. DNS rusak D. time skew
6. `serial: 1` membantu ... A. rolling blast radius B. encryption C. quorum otomatis D. backup
7. Token k3s sebaiknya ... A. secret mechanism terpisah B. inventory plain text C. output debug D. Git
8. `kubectl get nodes` membuktikan ... A. node view pada context B. backup valid C. app SLO D. SSH
9. Backup etcd bukan ... A. backup aplikasi/PV/database B. datastore backup C. evidence reference D. runbook input
10. Replace control-plane memerlukan ... A. migration/quorum plan B. apply langsung C. reset D. delete state
11. Firewall change dapat ... A. memutus access path B. meningkatkan quorum C. encrypt secret D. membuat DNS
12. PDB relevan untuk ... A. availability saat drain B. provider lock C. inventory parse D. token
13. Version pin membantu ... A. reproducibility B. membuka API C. bypass approval D. storage
14. Health check setelah join mencakup ... A. node/API/workload B. hanya task changed C. hanya package D. hanya SSH
15. Automatic updates harus ... A. policy/review B. selalu aktif C. tanpa rollback D. diabaikan
16. Jika node pertama patch gagal ... A. stop batch B. lanjut semua C. reset cluster D. hapus inventory
17. Stable IP penting untuk ... A. node/API path B. Jinja syntax C. Vault password D. role folder
18. Handoff credential ... A. tidak boleh via metadata biasa B. wajib output C. dicetak debug D. masuk README
19. Restore snapshot aktif tanpa runbook ... A. dilarang B. default C. health check D. patch
20. Rebuild evidence menghubungkan ... A. commit sampai health B. password sampai Git C. raw log sampai public D. token sampai inventory

## Kunci

1 A, 2 A, 3 A, 4 A, 5 A, 6 A, 7 A, 8 A, 9 A, 10 A, 11 A, 12 A, 13 A, 14 A, 15 A, 16 A, 17 A, 18 A, 19 A, 20 A.

## Esai dan Panduan

21. Rancang contract OpenTofu → Ansible.
22. Susun readiness gate dengan evidence.
23. Jelaskan quorum dan replacement control-plane.
24. Rancang k3s role version pin/idempotency tanpa token.
25. Buat rolling patch runbook.
26. Analisis firewall/SSH failure dan access recovery.
27. Bedakan backup etcd dan backup aplikasi.
28. Buat evidence chain redacted.
29. Jelaskan mengapa `changed=0` bukan cluster health.
30. Tentukan lima kondisi automation harus berhenti.

## Penilaian

Pilihan ganda 40, esai 60, lulus minimal 80. Secret/destructive violation = tidak lulus.
