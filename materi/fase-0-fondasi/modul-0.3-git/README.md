# Modul 0.3 — Git & Kolaborasi

> **Tujuan akhir:** nyaman bekerja dengan branch, merge, rebase, dan conflict resolution — serta mengelola proyek via GitLab (MR, issue, milestone) sebagai fondasi GitOps di fase berikutnya.

## Capaian Modul (Wajib)

- [ ] Bisa menjelaskan working area, staging, dan history — serta memindahkan perubahan di antaranya
- [ ] Bisa membuat & berpindah branch, merge, dan menyelesaikan conflict tanpa panik
- [ ] Bisa menjelaskan kapan pakai merge vs rebase, dan dampak rewrite history
- [ ] Bisa memperbaiki commit terakhir (amend) dan membatalkan perubahan dengan aman (`restore`, `reset`, `revert`)
- [ ] Bisa menulis conventional commit yang jelas (type, scope, breaking change)
- [ ] Bisa menjelaskan trunk-based development & kenapa cocok untuk tim kecil/SRE
- [ ] Bisa membuat Merge Request di GitLab, review, dan merge — termasuk squash & delete source branch
- [ ] Bisa mengelola issue & milestone di GitLab untuk melacak pekerjaan

## Rencana 2 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-git-fundamental](01-git-fundamental.md) | [Latihan:Fundamental](evaluasi/latihan.md) |
| 2 | [02-git-workflow](02-git-workflow.md) | [LAB-01](lab/LAB-01-gitlab-repo.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 0.1 & 0.2 selesai (VM `lab01` & repo lokal siap)
- Akun [gitlab.com](https://gitlab.com) (free tier cukup)
- Git terpasang di Mac (`git --version`); konfigurasi user:

```bash
git config --global user.name "Nama Anda"
git config --global user.email "email@anda.com"
git config --global init.defaultBranch main
# SSH key ke GitLab (pakai key dari modul 0.1, atau buat baru):
# Tambahkan ~/.ssh/id_ed25519.pub di GitLab → Settings → SSH Keys
```

- Sudah membaca [Fase 0 README](../README.md)

## Deliverables Modul

1. **Repo `sre-bootcamp`** di gitlab.com, berisi semua lab modul 0.1–0.3 dalam struktur:
   ```
   sre-bootcamp/
   ├── m0.1/   (lab backup, systemd, debug report)
   ├── m0.2/   (Caddyfile, dnsmasq config, trace report)
   └── m0.3/   (lab Git ini)
   ```
2. **≥ 5 Merge Request** ke diri sendiri (satu per modul/lab), masing-masing pakai conventional commit & di-link ke issue.
3. **≥ 1 milestone** "Fase 0" yang men-track semua lab, tertutup saat fase selesai.
4. **Nilai kuis ≥ 80%**

## Cara Memulai

Git adalah **sistem versi terdistribusi** — semua orang punya salinan history lengkap. Ini berarti: (1) bisa kerja offline, (2) tiap clone = backup, (3) **bisa rusak history kalau salah perintah**. Jadi sebelum mengetik `git` apa pun, pahami tiga area (working → staging → history). Modul ini 60% konsep + 40% praktik di GitLab. Buat repo **nyata** di gitlab.com sejak hari 2 — semua lab sebelumnya akan di-MR ke sana.

## Kaitan dengan Modul Berikutnya

Git bukan sekadar simpan kode. Di bootcamp ini, Git adalah **source of truth** untuk semua:
- **IaC** (OpenTofu) & **Ansible** playbook → review lewat MR (Fase 3 & 4)
- **GitOps** — ArgoCD menarik manifest dari repo Git, bukan `kubectl apply` manual (Fase 6)
- **Dashboard/alert sebagai code** → version-controlled (Fase 7)

Belajar conventional commit & MR sekarang = praktek review yang sama dipakai sepanjang bootcamp.