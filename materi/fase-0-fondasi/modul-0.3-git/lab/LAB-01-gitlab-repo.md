# LAB-01 — Repo `sre-bootcamp` di GitLab + MR ke Diri Sendiri

> **Target:** punya repo `sre-bootcamp` di gitlab.com yang menampung **semua lab modul 0.1–0.3**, dikelola lewat **Merge Request ke diri sendiri** dengan conventional commit, issue, dan milestone — fondasi GitOps untuk fase berikutnya.

## Latar Belakang
Sejak Fase 3 (OpenTofu), Fase 4 (Ansible), Fase 6 (GitOps/ArgoCD), semua infrastruktur = code di repo. Kebiasaan MR, conventional commit, dan issue tracking yang ditanam sekarang akan dipakai terus-menerus. Lab ini menggabungkan seluruh lab sebelumnya ke satu repo terorganisir.

## Prasyarat
- [ ] Modul 0.1 & 0.2 selesai (file lab backup, Caddyfile, dnsmasq config, report sudah ada di lokal)
- [ ] Akun gitlab.com + SSH key terdaftar (lihat [README modul](../README.md) prasyarat)
- [ ] Sudah baca [01-git-fundamental](../01-git-fundamental.md) & [02-git-workflow](../02-git-workflow.md)

## Waktu
± 2–3 jam (termasuk memindahkan file lab sebelumnya)

## Langkah

### 1. Buat Repo di GitLab

1. Login gitlab.com → **New project** → **Create blank project**.
2. Project name: `sre-bootcamp`.
3. Visibility: **Private** (ini latihan pribadi).
4. **Uncheck** "Initialize repository with README" (kita akan push dari lokal agar tidak ada conflict).
5. **Create project**.
6. Catat URL SSH: `git@gitlab.com:<username>/sre-bootcamp.git`.

### 2. Buat Struktur Lokal & Init Git

```bash
cd ~/Developer           # atau folder kerja utama Anda
mkdir -p sre-bootcamp/m0.1/lab sre-bootcamp/m0.2/lab sre-bootcamp/m0.3/lab
cd sre-bootcamp
git init
git branch -M main       # pastikan default branch main
```

Buat `.gitignore` & README utama:
```bash
cat > .gitignore <<'EOF'
# Secret — JANGAN PERNA commit
*.pem
*.key
.env
.env.*
*_rsa
ansible-vault-password.txt
id_ed25519

# OS / editor
.DS_Store
*.swp
.vscode/
.idea/

# Build artifact
*.pyc
__pycache__/
EOF

cat > README.md <<'EOF'
# SRE Bootcamp — Lab Repository

Repositori semua lab bootcamp SRE 2026 (on-prem track).

## Struktur
- `m0.1/` — Linux & Shell
- `m0.2/` — Networking untuk SRE
- `m0.3/` — Git & Kolaborasi

## Progres
Lihat milestone "Fase 0" di GitLab.
EOF
```

### 3. Komit Pertama & Push

```bash
git add .
git commit -m "docs: init sre-bootcamp repo + gitignore"
git remote add origin git@gitlab.com:<username>/sre-bootcamp.git
git push -u origin main
```

Cek di GitLab UI: file `README.md` & `.gitignore` muncul.

### 4. Buat Milestone & Issue di GitLab

**Milestone "Fase 0 — Fondasi":**
1. GitLab UI → **Plan → Milestones** → **New milestone**.
2. Title: `Fase 0 — Fondasi`. Start/End date: sesuaikan jadwal.
3. **Create milestone**.

**Issue per lab** (buat minimal 7 issue, satu per lab modul 0.1–0.3):
1. **Plan → Issues → New issue**.
2. Tiap issue:
   - Title: `m0.1 LAB-01 OrbStack VM`, dst.
   - Description: tempel **Acceptance Criteria** dari file lab masing-masing.
   - Milestone: `Fase 0 — Fondasi`.
   - Label: `lab` + `m0.1`/`m0.2`/`m0.3`.
3. Buat 7 issue:
   - `m0.1 LAB-01 OrbStack VM`
   - `m0.1 LAB-02 Backup script`
   - `m0.1 LAB-03 Debug proses`
   - `m0.2 LAB-01 Reverse proxy`
   - `m0.2 LAB-02 DNS lokal`
   - `m0.2 LAB-03 Trace koneksi`
   - `m0.3 LAB-01 Git repo`

Catat nomor tiap issue (`#1`, `#2`, …) — dipakai di deskripsi MR (`Closes #N`).

### 5. MR #1 — Modul 0.1 Labs

Pindahkan file lab modul 0.1 (dari LAB-01/02/03 modul 0.1) ke `m0.1/lab/`.

```bash
# Sesuaikan path sumber dengan tempat Anda menyimpan hasil lab modul 0.1
git switch -c m0.1-labs

# Contoh: salin file backup dari hasil LAB-02 modul 0.1
cp <path-lama>/backup.sh         m0.1/lab/backup.sh
cp <path-lama>/backup.service    m0.1/lab/backup.service
cp <path-lama>/backup.timer      m0.1/lab/backup.timer
cp <path-lama>/lab03-report.md   m0.1/lab/lab03-report.md
# (tambah/ubah sesuai file yang Anda hasilkan di modul 0.1)

git add m0.1/
git commit -m "feat(m0.1): tambah lab backup, systemd unit, debug report

- backup.sh idempotent + rotasi 7 hari
- backup.service & backup.timer (systemd)
- lab03-report.md (diagnosa CPU hog & respawn)

Closes #1, #2, #3"

git push -u origin m0.1-labs
```

**Buat MR di GitLab:**
1. UI → **Merge requests → New** → source `m0.1-labs` → target `main`.
2. Title: `feat(m0.1): lab Linux & Shell`.
3. Description: `Closes #1, #2, #3` + ringkasan singkat.
4. **Create merge request**.

**Review diri sendiri:**
1. Buka tab **Changes** — baca seluruh diff.
2. Pastikan: tidak ada secret, tidak ada file sampah (`.DS_Store`).
3. Approve → **Squash and merge** + **Delete source branch**.
4. Verifikasi issue #1, #2, #3 otomatis tertutup.

```bash
git switch main
git pull
git log --oneline -3             # lihat commit squash rapi
```

### 6. MR #2 — Modul 0.2 Labs

Ulangi pola untuk file lab modul 0.2:

```bash
git switch -c m0.2-labs
cp <path>/Caddyfile            m0.2/lab/Caddyfile
cp <path>/dnsmasq-lab.conf     m0.2/lab/dnsmasq-lab.conf
cp <path>/hosts-lab.txt        m0.2/lab/hosts-lab.txt
cp <path>/lab03-report.md      m0.2/lab/lab03-report.md

git add m0.2/
git commit -m "feat(m0.2): lab reverse proxy, DNS lokal, trace koneksi

- Caddyfile (host-based routing + TLS internal)
- dnsmasq config + SRV record
- lab03-report.md (3 skenario troubleshooting)

Closes #4, #5, #6"

git push -u origin m0.2-labs
```

Buat MR → squash & merge → delete branch → `git pull`.

### 7. MR #3 — Modul 0.3 (Lab ini sendiri)

```bash
git switch -c m0.3-gitlab
cat > m0.3/lab/git-setup.md <<'EOF'
# m0.3 LAB-01 — Git & GitLab Setup

## Bukti
- Repo: gitlab.com/<username>/sre-bootcamp
- Jumlah MR: (catat di sini setelah selesai)
- Milestone "Fase 0" status: (closed/open)
- Issue tertutup otomatis via Closes #N: (catat)

## Catatan
- Strategi: trunk-based, squash + delete source branch
- Konvensi commit: conventional commits
- Pelajaran: (1 hal yang paling berguna dari modul ini)
EOF

git add m0.3/
git commit -m "feat(m0.3): dokumentasi lab git & gitlab setup

- bukti repo, MR, milestone
- catatan trunk-based + conventional commit

Closes #7"

git push -u origin m0.3-gitlab
```

Buat MR → squash & merge → `Closes #7` → issue #7 tertutup.

### 8. Tutup Milestone & Lihat Hasil

1. GitLab UI → **Plan → Milestones** → `Fase 0 — Fondasi`.
2. Pastikan semua 7 issue **Closed** (merge auto-close). Kalau ada yang belum, tutup manual.
3. **Close milestone**.
4. Lihat `git log --oneline` lokal — history main rapi (1 commit per MR).

```bash
git switch main
git pull
git log --oneline --graph -15
```

### 9. (Opsional) Latih Conflict Resolution

Sengaja buat conflict antara dua branch untuk latihan:

```bash
git switch -c latih-conflict-a
echo "versi A" >> README.md
git commit -am "docs(readme): catatan versi A"

git switch main
echo "versi MAIN" >> README.md
git commit -am "docs(readme): catatan versi main"

git switch latih-conflict-a
git rebase main              # CONFLICT
# → edit README.md, gabungkan, hapus marker
git add README.md
git rebase --continue
git switch main
git merge latih-conflict-a   # fast-forward
git branch -d latih-conflict-a
```

Ini tidak perlu di-push (latihan lokal saja), atau jadikan MR ekstra bila ingin.

## Acceptance Criteria

- [ ] Repo `sre-bootcamp` ada di gitlab.com (private), bisa di-clone via SSH
- [ ] `.gitignore` mengecualikan secret & file sampah
- [ ] Struktur `m0.1/`, `m0.2/`, `m0.3/` ada di repo
- [ ] Milestone "Fase 0 — Fondasi" dibuat & berisi 7 issue
- [ ] ≥ 3 Merge Request dibuat, masing-masing pakai conventional commit
- [ ] Setiap MR: squash + delete source branch
- [ ] Issue tertutup otomatis lewat `Closes #N` di deskripsi MR
- [ ] `git log --oneline` history main rapi (1 commit per MR)
- [ ] Milestone "Fase 0" berstatus Closed
- [ ] `m0.3/lab/git-setup.md` berisi bukti & catatan
- [ ] Tidak ada secret yang ter-commit (`git log -p | grep -i 'password\|key'` bersih — atau secret sudah di-rotate jika pernah)

## Troubleshooting

| Gejala | Solusi |
|---|---|
| `Permission denied (publickey)` saat push | SSH key belum terdaftar di GitLab → Settings → SSH Keys, tambah `~/.ssh/id_ed25519.pub`; tes `ssh -T git@gitlab.com` |
| `fatal: remote origin already exists` | `git remote set-url origin <URL>` ganti, bukan `add` lagi |
| `rejected — non-fast-forward` saat push | Ada commit baru di remote: `git pull --rebase origin main` dulu, resolve conflict, baru push |
| Issue tidak tertutup otomatis | Cek `Closes #N` tepat (huruf, spasi); MR harus merge ke branch default; issue & repo sama |
| Squash tidak muncul opsi | GitLab Project → Settings → Repository → Merge requests → centang "Squash commits" |
| Tak sengaja commit secret | **Segera rotate secret**; lalu `git filter-repo`/BFG untuk hapus dari history; jangan hanya hapus file + commit baru |
| Conflict marker tersisa di file | `git diff` cari `<<<<<<<`; edit semua; `git add` sebelum continue |

## Catatan SRE
- Repo ini **hidup** — fase berikutnya (OpenTofu, Ansible, GitOps) akan menambah direktori `m3/`, `m4/`, `gitops/` dengan pola MR yang sama.
- Kebiasaan **squash + conventional commit + issue link** sekarang = changelog otomatis & audit trail nanti. Saat insiden, `git log` + issue adalah arsip "apa yang berubah & kenapa".
- **Secret di Git = insiden menunggu waktu.** Fase 6 akan bahas Sealed Secrets/SOPS; sekarang cukup jangan commit.

## Lanjut
[Evaluasi: Latihan & Kuis](../evaluasi/latihan.md)

---

**Selamat — Fase 0 (Fondasi) selesai!** Lanjut ke [Fase 1 — Container & OrbStack](../../fase-1-container-orbstack/README.md) (menyusul).