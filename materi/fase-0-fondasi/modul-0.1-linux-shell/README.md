# Modul 0.1 — Linux & Shell

> **Tujuan akhir:** nyaman mengetik perintah Linux di terminal, dan paham apa yang terjadi di balik skrip bash sederhana.

## Capaian Modul (Wajib)

- [ ] Bisa navigasi, membuat, mencari, menghapus file/folder tanpa ragu
- [ ] Bisa menjelaskan konsep permission numeric (`chmod 644`) dan mengatur pemilik/group
- [ ] Bisa melihat, memantau, dan menghentikan proses yang bermasalah
- [ ] Bisa menulis/mengelola systemd unit sederhana
- [ ] Bisa menggunakan `grep`, `awk`, `sed`, `jq`, dan pipe untuk membongkar data
- [ ] Bisa login SSH tanpa password ke VM OrbStack, transfer file, dan setup tunneling dasar
- [ ] Bisa menulis skrip bash yang idempotent (aman dijalankan berulang)
- [ ] Bisa menjadwalkan skrip dengan cron dan systemd timer

## Rencana 5 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-navigasi-filesystem](01-navigasi-filesystem.md), [02-permission](02-permission.md) | [Latihan:Fondasi-1-2](evaluasi/latihan.md) |
| 2 | [03-proses-systemd](03-proses-systemd.md), [04-text-processing](04-text-processing.md) | [Latihan:Dasar-3-4](evaluasi/latihan.md) |
| 3 | [05-networking-cli](05-networking-cli.md), [06-ssh](06-ssh.md) | [LAB-01](lab/LAB-01-orbstack-vm.md) |
| 4 | [07-bash-scripting](07-bash-scripting.md) | [LAB-02](lab/LAB-02-backup-script.md) |
| 5 | Review | [LAB-03](lab/LAB-03-debug-proses.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- MacBook Air M5 (Apple Silicon)
- Homebrew terpasang (`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`)
- Sudah membaca [Fase 0 README](../README.md)

## Deliverables Modul

1. **VM Sovereign** — VM Ubuntu bisa diakses SSH tanpa password dari Mac.
2. **Repo `sre-bootcamp/m0.1`** di GitLab yang berisi:
   - Skrip backup lengkap di `lab/backup.sh`
   - Service unit systemd untuk skrip backup di `lab/backup.service` + `.timer`
   - Laporan singkat debug proses di `lab/lab03-report.md`
3. **Nilai kuis ≥ 80%**

## Cara Memulai

Buka materi hari 1 di tab baru, buka terminal VM OrbStack di tab lain, dan **jalankan setiap perintah yang ditulis**. Topik ini 80% muscle memory — praktik yang menyelamatkan, bukan membaca.
