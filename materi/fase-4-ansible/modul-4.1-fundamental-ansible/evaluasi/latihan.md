# Latihan Modul 4.1 — Fundamental Ansible

Kerjakan dengan placeholder dan tanpa mutation production.

## Latihan

1. Gambar alur control node → SSH → managed node dan sebutkan dependency setiap boundary.
2. Buat inventory YAML dua server k3s dan satu agent dengan management address placeholder.
3. Jelaskan perbedaan `ansible_host`, hostname guest, node IP, dan API address.
4. Tulis `ansible.cfg` minimal tanpa credential dan jelaskan kebijakan host key.
5. Buat playbook marker dengan `copy`, handler, tag, `when`, dan `serial: 1`.
6. Prediksi perbedaan first run dan rerun.
7. Jelaskan kapan `changed_when`/`failed_when` berbahaya.
8. Buat readiness gate sebelum package mutation.
9. Analisis host salah akibat inventory group typo.
10. Tulis evidence summary tanpa raw verbose output.

## Rubrik

- Arsitektur/inventory/SSH 25%; YAML/module 25%; idempotency 25%; safety/evidence 25%.
- Nilai minimal 80%; pelanggaran secret/destructive guardrail menggugurkan latihan.

## Kaitan

Lanjutkan ke [Kuis](kuis-dan-jawaban.md), kemudian Modul 4.2.
