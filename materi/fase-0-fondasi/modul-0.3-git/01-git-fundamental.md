# 01 — Git Fundamental

> Tiga area, branch, merge, rebase, dan menyelesaikan conflict tanpa panik.

## Tujuan
- Paham tiga area: working directory, staging, history
- Bisa membuat/berpindah branch, merge, dan resolve conflict
- Bisa memperbaiki & membatalkan perubahan dengan aman (amend, restore, reset, revert)
- Mengerti dampak rewrite history & kapan menghindarinya

## 1. Tiga Area Git

Git melacak perubahan lewat tiga area. Paham ini = setengah dari semua masalah git hilang.

```
 ┌─────────────┐    git add     ┌─────────────┐    git commit    ┌─────────────┐
 │  Working    │ ─────────────► │   Staging   │ ──────────────► │   History   │
 │  Directory  │ ◄───────────── │   (Index)   │ ◄────────────── │  (commits)  │
 └─────────────┘   git restore  └─────────────┘                  └─────────────┘
   file di disk                   snapshot yang                  graph commit
   (bisa diubah bebas)            akan difoto                     (immutable*)
```

```bash
git status                  # lihat posisi tiap file di area mana
# Untracked  → file baru, belum dikenal git
# Modified   → berubah di working, belum di-staging
# Staged     → sudah di-staging, siap di-commit
# Committed  → sudah di history
```

**Analogi:** staging = panggung foto. `git add` = naik ke panggung. `git commit` = jepret foto. History = album foto berurutan.

## 2. Inisialisasi & Snapshot Pertama

```bash
mkdir belajar-git && cd belajar-git
git init                          # buat repo (.git/)
echo "# Belajar Git" > README.md
git add README.md                 → ke staging
git commit -m "feat: init repo"   → ke history
git log --oneline
```

```bash
# Lihat isi staging sebelum commit:
git diff --staged                 # (atau --cached, sinonim)
git diff                          # working vs staging (yang belum di-add)
```

## 3. Branch — Garis Hidup Paralel

Branch = pointer ke sebuah commit. Membuat branch = cabang garis hidup tanpa ganggu main.

```bash
git branch fitur-backup           # buat (tanpa pindah)
git switch fitur-backup           # pindah ke branch itu
git switch -c fitur-backup        # buat + pindah sekaligus (lebih umum)
git branch                        # daftar branch, * = aktif
git switch main                   # balik ke main
```

**Kenapa branch penting:** main selalu stabil. Eksperimen/fitur di branch sendiri. Kalau gagal, tinggal hapus branch — main tidak tersentuh.

## 4. Merge vs Rebase

Dua cara menyatukan branch. **Hasil akhir kode sama, tapi history berbeda.**

### Merge — tambah node, jaga history asli
```
main:      A──B────────M      ← merge commit (M) punya 2 parent
            \        /
fitur:       C──D──E          ← history fitur tetap utuh
```
```bash
git switch main
git merge fitur-backup         # buat merge commit
```

### Rebase — tulis ulang, history linear
```
SEBELUM                      SETELAH rebase fitur ke main
main:  A──B                  main:  A──B──C'──D'──E'    ← fitur "ditempel"
        \                                              (commit di-copy ulang)
fitur:   C──D──E
```
```bash
git switch fitur-backup
git rebase main               # pindahkan base fitur ke ujung main
git switch main
git merge fitur-backup        # fast-forward (tanpa merge commit) → linear
```

### Kapan pakai yang mana

| Aspek | Merge | Rebase |
|---|---|---|
| History | jaga asli (bisa berantakan/"buble") | rapi & linear |
| Rewrite commit? | tidak | **ya** (commit di-copy, hash berubah) |
| Aman untuk branch publik/team? | ya | **hati-hati** — rewrite bisa bikin chaos |
| Conflict | sekali di merge | bisa berulang per commit saat rebase |

**Aturan emas:** **jangan rebase commit yang sudah di-push & dipakai orang lain** (kecuali branch pribadi fitur). Rebase hanya untuk branch lokal/fitur sendiri sebelum di-MR.

## 5. Conflict Resolution — Tanpa Panik

Conflict terjadi saat dua branch mengubah baris yang sama.

```bash
git switch -c fitur-a
echo "versi A" >> file.txt
git commit -am "fitur-a: ubah file"

git switch main
echo "versi main" >> file.txt
git commit -am "main: ubah file"

git merge fitur-a              # CONFLICT
```

Git menandai file:
```
<<<<<<< HEAD
versi main
=======
versi A
>>>>>>> fitur-a
```

**Langkah resolve:**
1. Buka file, pilih versi yang benar (boleh gabungan).
2. Hapus semua marker (`<<<<<<<`, `=======`, `>>>>>>>`).
3. `git add file.txt` — tandai sudah resolve.
4. `git commit` (untuk merge) atau `git rebase --continue` (untuk rebase).

```bash
git mergetool                  # GUI/CLI tool bantu (opsional)
git merge --abort              # batal, kembali ke sebelum merge
git diff                       # lihat apa yang belum di-resolve
```

**Tips:** conflict kecil → edit manual. Conflict besar/3-arah → pakai `git mergetool` atau VS Code (buka file, klik "Accept Current/Incoming/Both").

## 6. Memperbaiki & Membatalkan — Aman

### Amend — perbaiki commit terakhir
```bash
git commit --amend             # ubah pesan commit terakhir
git commit --amend --no-edit   # tambahkan file yang terlupa ke commit terakhir
```
**Hati-hati:** amend mengubah hash commit. Kalau sudah di-push, harus `git push --force` → berisiko untuk branch team. Pakai hanya untuk commit lokal.

### Restore — buang perubahan working
```bash
git restore file.txt           # buang perubahan di working (belum di-add)
git restore --staged file.txt  # unstage (balik dari staging ke working)
git restore --source=HEAD~1 file.txt   # balik ke versi 1 commit lalu
```
`git restore` (modern) menggantikan `git checkout -- file` yang membingungkan.

### Reset — pindahkan pointer branch (rewrite history)
```bash
git reset --soft HEAD~1        # hapus commit, perubahan tetap di staging
git reset --mixed HEAD~1       # (default) hapus commit, perubahan ke working
git reset --hard HEAD~1        # hapus commit + perubahan HILANG total ⚠️
```
`--hard` = berbahaya. Perubahan yang belum di-commit **tidak bisa dikembalikan** (kecuali lewat reflog).

### Revert — batalkan dengan commit baru (aman untuk history publik)
```bash
git revert <commit>            # buat commit baru yang "membalikkan" commit itu
```
```
A──B──C──D──C'                 ← C' = revert C (history jadi, ada catatan)
```
**Revert vs Reset:** reset menghapus (rewrite). Revert menambah commit pembatal (history utuh, aman untuk branch publik/main). Di production, **main selalu pakai revert, bukan reset.**

## 7. Melihat History & Mencari

```bash
git log --oneline --graph --all        # graph semua branch
git log -p file.txt                    # history + diff per commit untuk file
git log --author="nama" --since="2 weeks ago"
git blame file.txt                     # siapa ubah baris ini, kapan, commit mana
git show <commit>                      # lihat detail satu commit
git reflog                             # "KTP git" — semua gerakan HEAD, penyelamat
```

**`git reflog` adalah penyelamat.** Bahkan setelah `reset --hard`, commit masih ada di reflog selama ~90 hari. Cara pulihkan:

```bash
git reflog                      # cari hash commit yang "hilang"
git reset --hard <hash>         # kembalikan HEAD ke sana
```

## 8. Remote — Push & Pull

```bash
git remote -v                              # daftar remote
git remote add origin git@gitlab.com:Anda/repo.git
git push -u origin main                    # -u = set upstream (sekali saja)
git push                                   # selanjutnya cukup ini
git pull                                   # fetch + merge
git fetch                                  # ambil tanpa merge (lebih aman untuk inspeksi)
```

**`fetch` vs `pull`:** `pull` = `fetch` + `merge`. `fetch` aman — hanya mengunduh, tidak mengubah branch lokal. Saat ragu, `fetch` dulu, lihat `git log origin/main`, baru putuskan.

## Latihan Cepat (20 menit)

```bash
# 1. Buat repo latihan
mkdir latihan-git && cd latihan-git
git init
echo "# Latihan" > README.md
git add . && git commit -m "feat: init"

# 2. Buat branch & dua perubahan paralel (bikin conflict sengaja)
git switch -c fitur-a
echo "baris dari fitur-a" >> file.txt
git add . && git commit -m "feat-a: tambah baris"

git switch main
echo "baris dari main" >> file.txt
git add . && git commit -m "chore: tambah baris di main"

# 3. Merge & resolve conflict
git merge fitur-a                 # CONFLICT
# → edit file.txt, gabungkan dua baris, hapus marker
git add file.txt
git commit                        # selesaikan merge

# 4. Latih amend & restore
echo "coba" >> README.md
git restore README.md             # buang perubahan
git commit --amend -m "feat: init repo (diperbaiki)"

# 5. Latih reset vs revert
echo "eksperimen" >> file.txt
git add . && git commit -m "exp: eksperimen"
git reset --hard HEAD~1           # hilangkan (simpan hash dulu via git reflog!)
git reflog                        # cari hash eksperimen
git revert <hash>                 # (opsional) latih revert di main

# 6. Bersihkan
cd .. && rm -rf latihan-git
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Lihat posisi file | `git status` |
| Naik ke staging | `git add` |
| Foto commit | `git commit -m` |
| Buat+pindah branch | `git switch -c` |
| Satukan branch | `git merge` / `git rebase` |
| Resolve conflict | edit → `git add` → `commit`/`--continue` |
| Perbaiki commit terakhir | `git commit --amend` |
| Buang perubahan working | `git restore` |
| Hapus commit (lokal) | `git reset --hard` ⚠️ |
| Batalkan commit (publik) | `git revert` |
| Selamatkan dari reset | `git reflog` |
| History graph | `git log --oneline --graph --all` |

## Cek Pemahaman

1. Beda working directory, staging, dan history — kapan perubahan masuk masing-masing?
2. Kapan harus pakai `revert` dan bukan `reset` di branch main?
3. Setelah `git reset --hard`, apakah commit benar-benar hilang selamanya? Bagaimana memulihkannya?
4. Kenapa "jangan rebase commit yang sudah di-push & dipakai orang lain"?