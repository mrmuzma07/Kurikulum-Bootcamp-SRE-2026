# Latihan — Modul 0.3 Git & Kolaborasi

Lakukan latihan di repo latihan lokal (boleh `sre-bootcamp` atau repo coba). Target: muscle memory + paham konsep, bukan hafal flag.

> **Aturan:** kerjakan di terminal. Catat output perintah penting di `m0.3/lab/log-latihan.md` (atau `lab/log-latihan-m0.3.md`).

---

## Hari 1 — Git Fundamental

### 1.1 Tiga Area
```bash
# 1. Buat repo latihan
mkdir latihan-git && cd latihan-git
git init

# 2. Buat file, amati posisinya di tiap tahap
echo "halo" > a.txt
git status                  # Untracked
git add a.txt
git status                  # Staged
git commit -m "feat: tambah a.txt"
git status                  # nothing to commit (clean)

# 3. Ubah file, lihat diff di tiap area
echo "dunia" >> a.txt
git diff                    # working vs staging (ada perubahan)
git add a.txt
git diff                    # (kosong — sudah di-staging)
git diff --staged           # staging vs HEAD (ada perubahan)
git commit -m "feat: tambah kata dunia"
```
Catat: kapan `git diff` kosong dan kapan `git diff --staged` berisi? Jelaskan.

### 1.2 Branch & Merge
```bash
# 1. Buat dua branch fitur dari main
git switch -c fitur-b
echo "B" >> b.txt && git add . && git commit -m "feat-b: file b"
git switch main
git switch -c fitur-c
echo "C" >> c.txt && git add . && git commit -m "feat-c: file c"

# 2. Merge fitur-b ke main (tanpa conflict)
git switch main
git merge fitur-b
git log --oneline --graph --all

# 3. Hapus branch yang sudah ter-merge
git branch -d fitur-b
```

### 1.3 Rebase vs Merge
```bash
# 1. Reset ke sebelum merge (latih ulang)
# Buat skenario: main maju, fitur-c tertinggal
git switch main
echo "main update" >> a.txt && git commit -am "chore: update di main"
git switch fitur-c
git rebase main             # tempel fitur-c di atas main baru
git log --oneline --graph --all    # lihat history linear
git switch main
git merge fitur-c           # fast-forward (tanpa merge commit)
git branch -d fitur-c
```
Bandingkan graph history: merge (ada cabang & merge commit) vs rebase (linear). Catat perbedaannya.

### 1.4 Conflict Resolution
```bash
# 1. Sengaja bikin conflict di baris yang sama
git switch -c fitur-x
echo "versi X" >> a.txt && git commit -am "feat-x: versi X"
git switch main
echo "versi MAIN" >> a.txt && git commit -am "chore: versi main"
git merge fitur-x           # CONFLICT

# 2. Resolve: edit a.txt → gabungkan jadi dua baris (versi MAIN lalu versi X), hapus marker
# 3. Selesaikan
git add a.txt
git commit                  # merge commit
git branch -d fitur-x
```

### 1.5 Amend, Restore, Reset, Revert
```bash
# 1. Amend: tambahkan file terlupa ke commit terakhir
echo "note" > note.txt
git add note.txt
git commit --amend --no-edit -m "chore: update + note"

# 2. Restore: buang perubahan yang belum di-add
echo "sampah" >> a.txt
git restore a.txt
cat a.txt                   # "sampah" hilang

# 3. Reset --hard + reflog penyelamat
git log --oneline | head -1        # catat hash commit terakhir
echo "eksperimen" >> a.txt
git add . && git commit -m "exp: eksperimen"
git reset --hard HEAD~1            # hilangkan commit eksperimen
git reflog                         # cari hash "exp: eksperimen"
git reset --hard <hash-eksperimen> # pulihkan!

# 4. Revert: batalkan commit dengan commit baru (simulasi main publik)
git revert HEAD --no-edit          # buat commit pembatal
git log --oneline -3
```
Jelaskan beda `reset --hard` vs `revert` — mana yang aman untuk `main`?

### 1.6 Reflog & Blame
```bash
git reflog                         # semua gerakan HEAD
git blame a.txt                    # siapa ubah tiap baris, kapan
```

---

## Hari 2 — Git Workflow & GitLab

### 2.1 Conventional Commit
```bash
# 1. Buat 5 commit dengan type berbeda di repo latihan
echo "1" >> a.txt && git commit -am "feat(core): fitur satu"
echo "2" >> a.txt && git commit -am "fix(core): perbaikan dua"
echo "3" >> a.txt && git commit -am "docs: update readme"
echo "4" >> a.txt && git commit -am "chore: bersihkan tmp"
echo "5" >> a.txt && git commit -am "ci: tambah job lint"
git log --oneline
```
Tulis: mana commit yang akan masuk kategori "fitur baru" vs "perbaikan bug" di changelog?

### 2.2 Squash Beberapa Commit
```bash
# 1. Buat branch dengan 3 commit "wip"
git switch -c fitur-squash
echo a >> sq.md && git add . && git commit -m "wip: a"
echo b >> sq.md && git add . && git commit -m "wip: b"
echo c >> sq.md && git add . && git commit -m "wip: c"

# 2. Squash jadi satu di main
git switch main
git merge --squash fitur-squash
git commit -m "feat: tambah sq.md (squash 3 wip)"
git log --oneline -2
git branch -d fitur-squash
```

### 2.3 .gitignore
```bash
# 1. Buat file yang seharusnya di-ignore
echo "secret" > .env
echo "build" > app.bin
touch .DS_Store
git status                    # muncul sebagai untracked

# 2. Tambahkan gitignore
cat > .gitignore <<'EOF'
.env
*.bin
.DS_Store
EOF
git add .gitignore && git commit -m "chore: tambah gitignore"
git status                    # .env, app.bin, .DS_Store kini ter-abaikan

# 3. Verifikasi tidak bisa di-add
git add .env 2>&1 || true     # git menolak (atau diam — tergantung versi)
```

### 2.4 GitLab Praktik (di repo `sre-bootcamp`)
Kerjakan di repo nyata (LAB-01). Catat di laporan:
```bash
# 1. Buat 1 MR ekstra (selain 3 di LAB-01) — mis. perbaikan README
git switch -c docs-perbaikan
# edit README.md ...
git commit -am "docs: perbaikan struktur readme"
git push -u origin docs-perbaikan
# → buat MR di GitLab, squash & merge

# 2. Setelah merge, pull & lihat history
git switch main
git pull
git log --oneline --graph -10

# 3. Cek issue tertutup otomatis
# (di GitLab UI: Plan → Issues → lihat status Closed)
```

### 2.5 Soal Refleksi
Tulis jawaban singkat di `lab/log-latihan-m0.3.md`:
1. Kenapa `git diff` kadang kosong padahal file berubah? Kapan harus pakai `git diff --staged`?
2. Anda salah commit file `.env` (berisi password) ke `main` yang sudah di-push. Urutan langkah yang benar: (a) hapus dari history, atau (b) rotate password dulu? Jelaskan kenapa urutan itu penting.
3. Trunk-based vs Git Flow: kalau Anda satu-satunya SRE di perusahaan startup, mana yang dipilih & kenapa?
4. Saat merge MR, opsi "Squash" menggabungkan 5 commit jadi 1. Apa keuntungan & kapan sebaiknya **tidak** dipakai?

---

## Catatan Performa

- [ ] Semua latihan dilakukan di terminal
- [ ] Output penting disimpan di repo `sre-bootcamp` di `m0.3/lab/log-latihan-m0.3.md`
- [ ] Bisa menjelaskan tiap perintah git yang dipakai & kenapa
- [ ] Bisa membedakan kapan merge vs rebase, reset vs revert, amend vs commit baru
- [ ] Repo GitLab punya history main yang rapi (squash + conventional commit)