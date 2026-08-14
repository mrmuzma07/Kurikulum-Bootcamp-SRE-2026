# Latihan Modul 9.1 — Platform Rebuild

Kerjakan tanpa memakai credential nyata. Untuk runtime, gunakan disposable target yang telah disetujui; bila tool/context tidak tersedia, tulis `belum diverifikasi` dan lengkapi static contract.

## Latihan

1. Gambar desired topology Capstone dan bedakan VM `infra`, VM `servers`, k3s production, serta k3d staging.
2. Buat ownership matrix OpenTofu, Ansible, k3s, Helm/GitOps, MetalLB, Observability, dan SRE.
3. Tulis target allowlist dan lima stop condition sebelum destroy/rebuild.
4. Buat contoh handoff metadata non-secret dari OpenTofu ke Ansible.
5. Jelaskan mengapa `kubectl get nodes` dan `Ansible changed=0` tidak cukup membuktikan readiness.
6. Susun urutan dependency dari resource OrbStack sampai k3s handoff.
7. Rancang recovery bila access bastion hilang sebelum bootstrap selesai.
8. Review simulated plan: dua VM disposable dan satu VM dengan owner tidak dikenal akan dihancurkan. Apa keputusan Anda?
9. Tulis evidence schema before/after rebuild tanpa raw state, plan, kubeconfig, atau token.
10. Buat readiness rubric untuk quorum, worker capacity, storage, MetalLB prerequisite, dan access recovery.

## Tugas Refleksi

- Bagian mana yang tetap dapat dibuktikan pada static lane?
- Evidence apa yang belum tersedia jika OpenTofu/Ansible/k3s tidak ada pada workstation?
- Mengapa state backup dan application backup harus dibedakan?

## Penilaian

Setiap soal 4 poin, total 40. Minimal 32/40 dan tidak ada pelanggaran guardrail. Pelanggaran credential, target scope, atau destructive safety berarti gagal otomatis dan harus mengulang review.

## Lanjut

Bandingkan jawaban dengan [kuis dan jawaban](kuis-dan-jawaban.md), lalu lanjut ke [Modul 9.2](../../modul-9.2-delivery-observability/README.md).
