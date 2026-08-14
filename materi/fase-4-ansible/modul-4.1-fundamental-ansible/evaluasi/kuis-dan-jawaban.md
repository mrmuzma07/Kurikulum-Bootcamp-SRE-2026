# Kuis dan Jawaban Modul 4.1

## Petunjuk

Pilih jawaban terbaik 1–20. Jawab esai 21–30. Nilai minimal 80%. **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

## Pilihan Ganda

1. Ansible umumnya berkomunikasi dengan managed node melalui ... A. SSH B. etcd C. registry D. MetalLB
2. `ansible_host` berarti ... A. alamat koneksi B. token C. node role D. package
3. Inventory group berguna untuk ... A. target/variable grouping B. encryption C. hypervisor D. backup
4. `ansible-inventory --graph` adalah ... A. review topology inventory B. apply C. k3s join D. firewall
5. Module deklaratif sebaiknya dipilih karena ... A. state dapat direview B. selalu tanpa side effect C. menghapus approval D. menyimpan secret
6. Handler dipanggil ketika ... A. task notify dan berubah B. play dimulai C. SSH gagal D. inventory kosong
7. `--limit` mengurangi ... A. blast radius B. checksum C. YAML indentation D. DNS TTL
8. `--check` berarti ... A. prediction/dry-run terbatas B. health cluster C. backup D. encryption
9. `changed=0` berarti ... A. task tidak mengubah state B. cluster sehat C. host reachable selamanya D. backup valid
10. Facts sebaiknya ... A. minimum dan relevan B. selalu dicetak semua C. menjadi secret store D. mengganti inventory
11. `become: true` membutuhkan ... A. privilege policy B. kubeconfig C. MetalLB D. registry
12. Loop perlu direview untuk ... A. daftar dan idempotency B. token C. state provider D. quorum
13. Host key policy ... A. harus mengikuti policy, bukan shortcut B. selalu dimatikan C. tidak penting D. mengganti sudo
14. `gather_facts` ... A. mengumpulkan fakta host B. memasang k3s C. membuat VM D. mengunci state
15. Playbook versioned membantu ... A. audit/review B. bypass approval C. menyembunyikan error D. mengganti SSH
16. Jika inventory salah, tindakan tepat ... A. stop dan perbaiki B. apply semua C. retry massal D. delete host
17. `notify` biasa digunakan untuk ... A. handler B. Vault password C. DNS D. API token
18. `shell` tanpa guard dapat ... A. tidak idempotent B. selalu aman C. mengenkripsi output D. membuat quorum
19. Ansible ping membuktikan ... A. respons modul B. k3s health C. storage durability D. production readiness
20. Credential inventory harus ... A. melalui secret mechanism B. ditulis plain text C. dicetak debug D. di-commit

## Kunci

1 A, 2 A, 3 A, 4 A, 5 A, 6 A, 7 A, 8 A, 9 A, 10 A, 11 A, 12 A, 13 A, 14 A, 15 A, 16 A, 17 A, 18 A, 19 A, 20 A.

## Esai dan Panduan

21. Bedakan control/managed node: sebutkan OS, SSH, Python, privilege, dan evidence.
22. Rancang inventory non-secret: stable key, address, role, environment.
23. Jelaskan idempotency dan mengapa `changed=0` bukan health.
24. Rancang handler yang tidak restart tanpa perubahan.
25. Kapan `--check` dapat menyesatkan?
26. Analisis host unreachable dan stop condition.
27. Mengapa group `all` berbahaya untuk mutation?
28. Bagaimana meredact evidence?
29. Mengapa module lebih baik dari shell pada baseline?
30. Buat urutan preflight → review → apply terbatas → rerun.

## Penilaian

Pilihan ganda 40 poin, esai 60 poin, lulus minimal 80. Secret/destructive violation = tidak lulus.
