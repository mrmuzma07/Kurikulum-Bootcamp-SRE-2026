# 02 — Git Workflow: Conventional Commit, Trunk-Based & GitLab

> Pesan commit yang bermakna, strategi branch, dan mengelola proyek via Merge Request.

## Tujuan
- Bisa menulis conventional commit (type, scope, breaking change)
- Mengerti trunk-based development & kenapa cocok untuk tim SRE kecil
- Bisa membuat Merge Request, review, squash & merge di GitLab
- Bisa mengelola issue & milestone untuk melacak pekerjaan

## 1. Kenapa Pesan Commit Penting

Tiga bulan dari sekarang, kamu akan baca `git log` untuk tahu "kapan ini rusak & kenapa." Commit `update`, `fix`, `wip` = sampah. Commit `fix(mimir): batasi retention ke 30 hari` = harta. History yang baik = dokumentasi otomatis.

## 2. Conventional Commits

Format standar yang dibaca manusia **dan** mesin (auto-generate changelog, auto-bump versi).

```
<type>(<scope>): <deskripsi singkat>

<body panjang opsional>

<footer: BREAKING CHANGE / refs issue>
```

| Type | Arti | Contoh |
|---|---|---|
| `feat` | fitur baru | `feat(mimir): tambah retention 30 hari` |
| `fix` | perbaikan bug | `fix(ansible): k3s install gagal di ARM64` |
| `docs` | dokumentasi | `docs: tambah runbook node-down` |
| `refactor` | ubah struktur tanpa ubah perilaku | `refactor(backup): pisahkan fungsi kompres` |
| `test` | tambah/ubah test | `test(helm): uji chart rollback` |
| `chore` | maintenance (deps, config) | `chore: update .gitignore` |
| `ci` | pipeline CI/CD | `ci: tambah job tofu plan di MR` |
| `perf` | perbaikan performa | `perf(promql): pakai recording rule` |

**Scope** = area kode (opsional): `(mimir)`, `(ansible)`, `(caddy)`.

**Breaking change** — dua cara:
```
feat(api)!: ganti format respons ke JSON         ← tanda ! setelah scope
BREAKING CHANGE: format respons kini JSON         ← di footer
```

**Aturan deskripsi:**
- Imperatif sekarang: "tambah", "perbaiki" — bukan "menambahkan", "diperbaiki"
- ≤ 72 karakter di baris pertama
- Jelaskan **kenapa** di body, bukan **apa** (diff sudah tunjukkan apa)

Contoh baik:
```
fix(ansible): k3s agent gagal join di ARM64

Binary k3s diunduh untuk amd64 karena tidak set arch dari facts.
Set `ansible_architecture` → `arm64` mapping di role.

Fixes #42
```

## 3. Strategi Branch — Trunk-Based Development

```
        main (trunk, selalu stabil & deployable)
  ───────●───────●───────●───────●──────────●──────►
          \      /        \      /            \
           ●──●●           ●──●●              ●   ← fitur branch
         (merge cepat)   (merge cepat)     (merge cepat)
         fitur-backup    fitur-dns         fitur-git
```

**Trunk-based:** semua orang cabang pendek dari `main`, kerja 1–2 hari, lalu MR kembali ke `main`. Branch berumur pendek.

**Lawan: Git Flow** (main + develop + release + hotfix branch) — cocok untuk produk dengan rilis terjadwal, **overkill untuk tim SRE kecil**.

| Aspek | Trunk-Based | Git Flow |
|---|---|---|
| Branch | `main` + fitur pendek | `main`+`develop`+`release`+`hotfix` |
| Umur branch | 1–2 hari | bisa berminggu-minggu |
| Integrasi | sering (MR kecil) | jarang (merge besar) |
| Cocok untuk | tim kecil, deploy terus | produk rilis terjadwal |
| Conflict | kecil & sering (mudah) | besar & jarang (menakutkan) |

**Untuk bootcamp ini:** trunk-based. Tiap lab = satu fitur branch = satu MR ke `main` milik sendiri. MR kecil = review mudah = belajar kebiasaan baik.

## 4. `.gitignore` — Jangan Commit Sampah

File yang tidak boleh masuk history: secret, build artifact, file OS, virtualenv.

```gitignore
# .gitignore
# Secret — JANGAN PERNA commit
*.pem
*.key
.env
ansible-vault-password.txt

# Build artifact
*.pyc
__pycache__/
node_modules/
dist/

# OS
.DS_Store

# Editor
.vscode/
*.swp

# Terraform/OpenTofu
*.tfstate
*.tfstate.*
.terraform/
```

**Aturan emas:** secret yang sudah ter-commit **tidak benar-benar hilang** dengan hapus file + commit baru — masih ada di history. Butuh `git filter-repo`/BFG + **rotasi secret**. Lebih baik jangan commit dari awal.

## 5. Merge Request di GitLab

MR = permintaan menyatukan branch fitur ke `main`, **lewat review**. Ini pusat kolaborasi.

### Alur MR
```
1. Buat branch fitur     → git switch -c fitur-dns
2. Kerja + commit        → (conventional commit)
3. Push                  → git push -u origin fitur-dns
4. Buat MR di GitLab UI  → source: fitur-dns → target: main
5. Review (diri/team)    → komentar per baris, diskusi
6. Pipeline lulus        → CI jalan otomatis (Fase 6)
7. Squash & merge        → hapus source branch
8. Pull main             → git pull
```

```bash
# Push branch & buka MR dari CLI (GitLab CLI "glab" — opsional):
brew install glab
glab auth login
glab mr create --title "feat(m0.2): reverse proxy Caddy" --description "Closes #5"
```

### Opsi saat merge (di GitLab UI)
| Opsi | Kapan |
|---|---|
| **Merge commit** | pertahankan merge commit (history cabang terlihat) |
| **Squash** | gabung semua commit fitur jadi 1 di main (history main rapi) |
| **Fast-forward** | tempel tanpa merge commit (harus tidak ada commit baru di main) |
| **Delete source branch** | hapus branch fitur setelah merge (jaga repo bersih) |

**Rekomendasi bootcamp:** **Squash + Delete source branch**. History main jadi 1 commit per MR, bersih, dan conventional.

### Review etika (walau ke diri sendiri)
- Baca diff penuh sebelum approve.
- Komentar per baris: "kenapa begini?" bukan "ini salah".
- Cek: ada secret? ada `console.log`/debug? ada file sampah?
- Tanda `LGTM` (looks good to me) hanya setelah benar-benar baca.

## 6. Issue & Milestone — Melacak Pekerjaan

### Issue = satuan tugas
Setiap lab = 1 issue. Beri label, assignee, due date.

```markdown
# Issue: LAB-01 Reverse Proxy Caddy
## Acceptance Criteria
- [ ] Caddy running as systemd
- [ ] 2 situs dengan TLS internal
- [ ] Caddyfile di repo
## Linked
- MR !12
```

### Milestone = kelompok tugas dengan target
```
Milestone "Fase 0 — Fondasi"
  ├── Issue: m0.1 LAB-01 OrbStack VM
  ├── Issue: m0.1 LAB-02 Backup script
  ├── Issue: m0.1 LAB-03 Debug proses
  ├── Issue: m0.2 LAB-01 Reverse proxy
  ├── Issue: m0.2 LAB-02 DNS lokal
  ├── Issue: m0.2 LAB-03 Trace koneksi
  └── Issue: m0.3 LAB-01 Git repo
  Start: ...  End: ...  ← tutup milestone saat fase selesai
```

Di GitLab: **Plan → Issues** buat issue; **Plan → Milestones** buat milestone; link issue ke milestone via dropdown. MR yang menulis `Closes #N` di deskripsi akan **otomatis menutup issue N** saat merge.

### Labels standar (opsional tapi rapi)
| Label | Arti |
|---|---|
| `m0.1`, `m0.2`, `m0.3` | modul mana |
| `lab` | pekerjaan lab |
| `docs` | dokumentasi |
| `blocked` | butuh sesuatu dulu |

## 7. Struktur Repo `sre-bootcamp`

Target akhir modul — satu repo yang menampung semua lab:

```
sre-bootcamp/
├── README.md                    ← overview + progres
├── .gitignore
├── m0.1/
│   ├── lab/backup.sh
│   ├── lab/backup.service
│   ├── lab/backup.timer
│   └── lab/lab03-report.md
├── m0.2/
│   ├── lab/Caddyfile
│   ├── lab/dnsmasq-lab.conf
│   ├── lab/hosts-lab.txt
│   └── lab/lab03-report.md
└── m0.3/
    └── lab/git-setup.md         ← bukti lab ini
```

Setiap direktori modul = satu MR (atau beberapa MR per lab). Dengan squash, `git log main` = daftar fitur yang rapi.

## Latihan Cepat (20 menit)

```bash
# 1. Latih conventional commit
mkdir coba-commit && cd coba-commit
git init
echo "# Demo" > README.md
git add . && git commit -m "docs: init readme"
echo "config" > app.conf
git add . && git commit -m "feat(core): tambah config dasar"
echo "bug" > fix.txt
git add . && git commit -m "fix(core): perbaiki config typo"
git log --oneline

# 2. Buat branch & simulasi MR flow (lokal)
git switch -c fitur-readme
echo "## Progress" >> README.md
git commit -am "docs: tambah section progress"
git switch main
git merge --no-ff fitur-readme -m "Merge branch 'fitur-readme'"   # simulasi merge commit MR
git branch -d fitur-readme

# 3. Latih squash beberapa commit jadi satu
git switch -c fitur-banyak
echo a >> notes.md && git add . && git commit -m "wip: a"
echo b >> notes.md && git add . && git commit -m "wip: b"
echo c >> notes.md && git add . && git commit -m "wip: c"
git switch main
git merge --squash fitur-banyak
git commit -m "docs: tambah notes (squash 3 wip)"
git log --oneline

# 4. Bersihkan
cd .. && rm -rf coba-commit
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Pesan commit bermakna | conventional: `type(scope): desc` |
| Strategi branch tim kecil | trunk-based (fitur branch pendek → MR) |
| Hindari commit sampah | `.gitignore` |
| Usulkan perubahan | Merge Request (source → main) |
| History main rapi | Squash + delete source branch saat merge |
| Tutup issue otomatis | `Closes #N` di deskripsi MR |
| Kelompok tugas | Milestone |

## Cek Pemahaman

1. Apa beda `feat` vs `fix` vs `chore` dalam conventional commit? Beri contoh masing-masing untuk konteks SRE.
2. Kenapa trunk-based lebih cocok untuk tim SRE kecil daripada Git Flow?
3. Apa risiko commit file `.env` (secret), dan mengapa sekadar hapus + commit baru tidak cukup?
4. Apa efek pilih "Squash" saat merge MR di GitLab, dan kenapa direkomendasikan?
5. Bagaimana sebuah MR bisa menutup issue secara otomatis?