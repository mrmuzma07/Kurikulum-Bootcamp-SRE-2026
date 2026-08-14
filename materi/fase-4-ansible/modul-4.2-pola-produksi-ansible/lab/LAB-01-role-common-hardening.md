# LAB-01 — Role `common` dan Hardening Review

## Tujuan

Menyusun role common dengan defaults aman, template marker, handler, dan hardening flags yang tidak aktif secara default.

## Struktur Minimal

```text
roles/common/
├── defaults/main.yml
├── tasks/main.yml
├── handlers/main.yml
├── templates/marker.conf.j2
└── meta/main.yml
```

## Langkah

1. Definisikan `common_packages`, `common_timezone`, dan flag hardening.
2. Buat task package/service memakai module state.
3. Buat template marker non-secret dan handler reload.
4. Tambahkan `assert` untuk environment dan supported OS.
5. Review firewall/SSH/fail2ban task sebagai change yang membutuhkan approval terpisah.
6. Jalankan lint/syntax atau lakukan static review bila binary tidak ada.
7. Rancang first run/rerun dan failure evidence.

## Guardrail

Jangan mengaktifkan firewall, mengganti SSH policy, atau menjalankan patch pada host yang tidak disposable dan memiliki access recovery. Jangan menyimpan Vault password, private key, atau token.

## Acceptance Criteria

- [ ] Role contract dan defaults tertulis.
- [ ] Task idempotent dan handler terarah.
- [ ] Hardening flags memiliki blast-radius note.
- [ ] Static scan tidak menemukan secret.
- [ ] Runtime yang tidak dijalankan ditandai belum diverifikasi.
