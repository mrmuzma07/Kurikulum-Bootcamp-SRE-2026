# Latihan Modul 5.1 — Helm Fundamental

1. Gambarkan hubungan chart, release, values, namespace, repository, dan cluster.
2. Rancang struktur chart aplikasi internal dan jelaskan owner setiap file.
3. Tulis values contract non-secret untuk image, replica, Service, resources, dan probes.
4. Jelaskan kapan memakai `required`, `default`, `quote`, `toYaml`, `nindent`, `with`, dan `range`.
5. Buat schema yang menolak replica bertipe string dan image tanpa digest.
6. Rancang review rendered YAML tanpa menyimpan raw Secret.
7. Bandingkan `helm lint`, `helm template`, `helm package`, dan runtime install.
8. Susun runbook install/upgrade dengan context, namespace, `--wait`, timeout, dan approval.
9. Analisis kapan rollback tidak cukup karena migration atau CRD.
10. Jelaskan perbedaan chart repository klasik dan OCI registry.

## Rubrik

Chart/values/template 30%, lifecycle 30%, repository/versioning 20%, safety/evidence 20%. Minimal 80%; pelanggaran secret atau destructive guardrail menggugurkan.

## Kaitan

Gunakan objek dari Modul 2.1 dan operasi dari Modul 2.4. Lanjutkan ke [Kuis](kuis-dan-jawaban.md).
