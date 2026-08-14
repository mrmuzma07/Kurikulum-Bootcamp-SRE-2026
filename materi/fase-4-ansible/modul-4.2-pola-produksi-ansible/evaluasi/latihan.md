# Latihan Modul 4.2 — Pola Produksi

1. Rancang struktur role `common` dan jelaskan ownership setiap directory.
2. Buat defaults non-secret dengan flag firewall konservatif.
3. Tulis template Jinja2 marker dengan `default` dan validasi input.
4. Jelaskan variable precedence dan policy override.
5. Bandingkan role lokal dan collection eksternal.
6. Rancang handler reload yang aman.
7. Jelaskan batas `no_log` dan redaction.
8. Susun pipeline lint → syntax → check → approval.
9. Analisis partial failure pada `serial: 1`.
10. Tulis Vault workflow tanpa memasukkan password/payload.

## Rubrik

Role/repository 25%, templating/variables 25%, safety/Vault 25%, CI/failure/evidence 25%. Minimal 80%; pelanggaran secret/destructive guardrail menggugurkan.

## Kaitan

Gunakan bersama [Kuis](kuis-dan-jawaban.md) dan lanjut ke Modul 4.3.
