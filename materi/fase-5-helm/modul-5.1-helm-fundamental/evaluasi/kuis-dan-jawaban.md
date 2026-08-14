# Kuis dan Jawaban Modul 5.1

## Petunjuk

Pilih jawaban 1–20 dan jawab esai 21–30. **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Pilihan Ganda

1. Chart adalah ... A. paket template dan metadata B. node Kubernetes C. password D. snapshot etcd
2. Release adalah ... A. instance chart pada namespace B. OCI token C. image layer D. SSH host
3. `values.yaml` sebaiknya berisi ... A. default non-secret B. private key C. kubeconfig D. PAT
4. `helm template` membuktikan ... A. render B. SLO C. scheduling production D. backup
5. `nindent` membantu ... A. indentation YAML B. encryption C. rollback data D. DNS
6. `required` digunakan untuk ... A. menghentikan input wajib yang hilang B. mencetak Secret C. skip lint D. force upgrade
7. `--wait` berarti ... A. menunggu kondisi resource tertentu B. menjamin SLO C. backup database D. pin image
8. `--atomic` ... A. dapat rollback release saat upgrade gagal B. memulihkan external side effect C. mengganti CRD otomatis D. menghapus semua namespace
9. Chart version mengikuti ... A. SemVer chart B. password registry C. Pod IP D. node name
10. `helm history` menampilkan ... A. revision release B. kernel log C. etcd snapshot D. Git commit otomatis
11. `--set` untuk secret berisiko karena ... A. masuk history/log B. selalu terenkripsi C. memperbaiki readiness D. membuat PDB
12. OCI registry menyimpan ... A. chart artifact B. kubelet state C. Pod logs saja D. DNS record
13. Selector Service harus ... A. cocok dengan labels Pod B. berubah tiap upgrade C. berisi token D. dihapus
14. `helm lint` hijau berarti ... A. lint lulus, bukan health B. production aman C. backup valid D. SLO tercapai
15. Rollback release tidak otomatis memulihkan ... A. data migration B. chart revision C. rendered template D. release name
16. `helm uninstall` harus memverifikasi ... A. release dan namespace B. semua cluster C. PAT D. raw state
17. Image production sebaiknya memakai ... A. digest immutable B. latest C. password tag D. random tag
18. `helm package` membuktikan ... A. archive chart dibuat B. registry push sukses C. Pod Ready D. Ingress reachable
19. Context safety perlu diperiksa sebelum ... A. mutation release B. membaca README C. menulis schema D. membuat helper
20. Evidence release minimum memuat ... A. version/target/status redacted B. token C. kubeconfig D. raw Secret

## Kunci

1 A, 2 A, 3 A, 4 A, 5 A, 6 A, 7 A, 8 A, 9 A, 10 A, 11 A, 12 A, 13 A, 14 A, 15 A, 16 A, 17 A, 18 A, 19 A, 20 A.

## Esai dan Panduan

21. Rancang struktur chart aplikasi internal.
22. Jelaskan values precedence dan schema validation.
23. Bedakan lint, render, package, install, dan health evidence.
24. Tulis runbook upgrade aman.
25. Analisis risiko `--set`, rendered output, dan `helm get all`.
26. Jelaskan rollback dengan migration/CRD.
27. Bandingkan repository klasik dan OCI.
28. Rancang release evidence redacted.
29. Tentukan lima stop condition sebelum install.
30. Jelaskan mengapa Helm bukan pengganti GitOps.

## Penilaian

Pilihan ganda 40, esai 60, lulus minimal 80. Pelanggaran guardrail secret/destructive = tidak lulus.
