# Kuis & Kunci Jawaban — Modul 0.3 Git & Kolaborasi

> **Cara kerja:** jawab sendiri tanpa buka catatan. Cek di bagian bawah. Target ≥ 80% (16 dari 20).

---

## Bagian A — Pilihan Ganda (10 soal)

**1.** Saat sebuah file baru dibuat lalu `git add`, file itu berada di area...
- A. History
- B. Staging (index)
- C. Working directory saja
- D. Remote

**2.** Perintah untuk melihat perubahan yang **sudah di-staging** (akan difoto commit) adalah...
- A. `git diff`
- B. `git diff --staged`
- C. `git status --diff`
- D. `git log -p`

**3.** `git reset --hard HEAD~1` berakibat...
- A. Commit terakhir dibatalkan, perubahan tetap di working
- B. Commit terakhir dibatalkan, perubahan di staging
- C. Commit terakhir & perubahan HILANG total dari working
- D. Membuat commit baru yang membatalkan

**4.** Beda `git revert` vs `git reset` yang benar...
- A. Keduanya sama
- B. `revert` menambah commit pembatal (aman untuk history publik), `reset` menghapus commit (rewrite history)
- C. `reset` aman untuk main, `revert` tidak
- D. `revert` hanya untuk branch lokal

**5.** Conventional commit `fix(ansible): k3s gagal di ARM64` — type & scope-nya...
- A. type=`fix`, scope=`ansible`
- B. type=`ansible`, scope=`fix`
- C. type=`fix(ansible)`, scope=tidak ada
- D. type=`k3s`, scope=`ARM64`

**6.** Tanda `!` dalam `feat(api)!: ganti format` menandakan...
- A. Commit urgent
- B. Breaking change
- C. Branch baru
- D. Hotfix

**7.** Trunk-based development memiliki ciri...
- A. Branch `develop` terpisah dari `main`
- B. Branch fitur berumur panjang (berminggu-minggu)
- C. Branch fitur pendek (1–2 hari) langsung di-MR ke `main`
- D. Tidak pernah pakai branch

**8.** Setelah `git reset --hard`, commit yang "hilang"...
- A. Hilang selamanya
- B. Masih bisa dipulihkan via `git reflog` (selama ~90 hari)
- C. Hanya bisa dipulihkan dari remote
- D. Masih ada di staging

**9.** Opsi "Squash" saat merge MR di GitLab berfungsi...
- A. Menghapus semua commit fitur
- B. Menggabungkan semua commit fitur menjadi 1 commit di main
- C. Membuat branch baru
- D. Membatalkan merge

**10.** Cara agar MR **otomatis menutup** issue #5 saat di-merge...
- A. Beri label `close` pada MR
- B. Tulis `Closes #5` di deskripsi MR (dan target branch = default)
- C. Beri nama branch `close-5`
- D. Tidak mungkin otomatis

---

## Bagian B — Perintah (4 soal)

**11.** Tulis urutan perintah untuk: membuat branch `fitur-dns`, commit perubahan, lalu menyatukan ke `main` dengan **merge commit** (bukan fast-forward).

**12.** Tulis perintah untuk: memperbaiki **pesan** commit terakhir menjadi `docs: perbaikan readme` (tanpa menambah file).

**13.** Tulis perintah untuk: membatalkan perubahan pada `config.yaml` yang **belum di-`git add`** (kembalikan ke versi HEAD).

**14.** Tulis baris untuk `.gitignore` agar file berikut tidak masuk repo: semua file `.env` (termasuk `.env.prod`), file `id_ed25519`, dan direktori `__pycache__/`.

---

## Bagian C — Skenario (2 soal)

**15.** Anda sedang di branch `fitur-logging`. Anda sadar branch ini tertinggal 20 commit di belakang `main`, dan Anda ingin history tetap linear tanpa merge commit. Tulis urutan perintah untuk menyesuaikan, serta apa yang harus Anda lakukan jika muncul conflict di tengah proses.

**16.** Dua developer mengubah baris yang sama di `values.yaml` dari branch berbeda. Branch A di-MR duluan & di-merge. Saat branch B di-MR, pipeline gagal dengan conflict. Jelaskan langkah-langkah menyelesaikan conflict ini **di GitLab** (UI dan/atau lokal), dan bagaimana memastikan hasil akhir benar.

---

## Bagian D — Troubleshooting (2 soal)

**17.** Anda menjalankan `git push` dan dapat `! [rejected] main -> main (non-fast-forward)`. Apa penyebabnya & urutan langkah memperbaikinya tanpa kehilangan commit lokal Anda?

**18.** Anda tidak sengaja `git commit` file `ansible-vault-password.txt` (secret) ke branch yang **sudah di-push** ke GitLab. Langkah apa yang **wajib** dilakukan (urutan), dan kenapa sekadar `git rm` + commit baru **tidak cukup**?

---

## Bagian E — Esai Singkat (2 soal)

**19.** Jelaskan kenapa trunk-based development dengan **MR kecil & sering** mengurangi risiko conflict dibanding Git Flow dengan branch berumur panjang. Beri analogi.

**20.** Bootcamp ini menjadikan Git sebagai **source of truth** untuk infrastruktur (OpenTofu, Ansible, GitOps/ArgoCD). Sebutkan 3 keuntungan mengelola infrastruktur lewat Git + MR dibanding mengkonfigurasi server dengan SSH manual, dan 1 risiko yang harus dikelola (petunjuk: secret).

---

## Kunci Jawaban

### A — Pilihan Ganda
1. **B** — `git add` memindahkan ke staging (index)
2. **B** — `git diff --staged` (= `--cached`) lihat perubahan staging vs HEAD
3. **C** — `--hard` hapus commit + perubahan working hilang total
4. **B** — revert menambah commit pembatal (history utuh, aman publik); reset rewrite (hapus)
5. **A** — type=`fix`, scope dalam kurung = `ansible`
6. **B** — `!` setelah scope = breaking change
7. **C** — trunk-based: fitur branch pendek → MR ke main
8. **B** — reflog menyimpan gerakan HEAD ~90 hari; `git reset --hard <hash>` pulihkan
9. **B** — squash gabung semua commit fitur → 1 commit di main
10. **B** — `Closes #N` (atau `Fixes`/`Resolves`) di deskripsi MR, target = default branch

### B — Perintah
11. ```bash
    git switch -c fitur-dns
    # ... edit file ...
    git add .
    git commit -m "feat(dns): tambah config"
    git switch main
    git merge --no-ff fitur-dns -m "Merge branch 'fitur-dns'"
    ```
12. ```bash
    git commit --amend -m "docs: perbaikan readme"
    ```
13. ```bash
    git restore config.yaml
    ```
14. ```gitignore
    .env
    .env.*
    id_ed25519
    __pycache__/
    ```

### C — Skenario
15. Rebase (linear, tanpa merge commit):
    ```bash
    git switch fitur-logging
    git rebase main
    # jika conflict:
    #   - edit file, hapus marker <<<<<<< ======= >>>>>>>
    #   - git add <file>
    #   - git rebase --continue
    #   (ulangi per commit yang conflict)
    # jika ingin batal: git rebase --abort
    git switch main
    git merge fitur-logging      # fast-forward → linear
    ```
    Kunci: `--continue` per commit, `--abort` untuk urungkan seluruh rebase.

16. Langkah resolve conflict branch B setelah A merge:
    - **Lokal**: `git switch fitur-b`, `git fetch origin`, `git rebase origin/main` (atau `git merge origin/main`) → conflict muncul.
    - Buka `values.yaml`, pilih/gabungkan nilai benar (A sudah merge → ambil perubahan A + tambahan B yang tidak bentrok), hapus marker.
    - `git add values.yaml`, `git rebase --continue` (atau commit merge).
    - `git push --force-with-lease` (karena rebase rewrite; **`--force-with-lease` lebih aman dari `--force`** — menolak kalau remote berubah).
    - **Verifikasi**: jalankan pipeline/test, review diff final, pastikan nilai `values.yaml` sesuai intensi gabungan A+B, baru merge MR.
    - Pencegahan ke depan: rebase/sync ke main sering agar conflict kecil.

### D — Troubleshooting
17. Penyebab: ada commit baru di `origin/main` yang belum ada di lokal (orang lain/CI push duluan), sehingga push lokal menolak (bukan fast-forward). Langkah aman tanpa kehilangan commit lokal:
    ```bash
    git fetch origin
    git rebase origin main        # (atau git pull --rebase origin main)
    # resolve conflict jika ada → git add → git rebase --continue
    git push
    ```
    Alternatif: `git pull --rebase`. **Jangan** `git push --force` di main bersama — itu menimpa commit orang lain. `--force-with-lease` hanya untuk branch fitur pribadi.

18. Urutan wajib:
    1. **Rotasi/ganti secret itu segera** — anggap secret sudah bocor (GitLab & siapa pun yang pernah akses bisa lihat, walau repo private; mungkin sudah ter-cache/clone).
    2. Hapus secret dari **history** (bukan hanya working tree): pakai `git filter-repo` atau BFG Repo-Cleaner untuk menghapus file dari semua commit, lalu `git push --force-with-lease` (koordinasi dengan siapa pun yang punya clone).
    3. Tambahkan ke `.gitignore` agar tidak terulang.
    4. Komunikasikan ke tim (jika shared) untuk re-clone (history berubah).
    Kenapa `git rm` + commit baru tidak cukup: secret masih ada di commit lama di history Git — siapa pun `git log -p` atau clone bisa menemukannya. Git追踪 seluruh riwayat; hapus di HEAD saja tidak menghapus masa lalu.

### E — Esai
19. MR kecil & sering = perubahan kecil tiap integrasi → overlap area kode kecil → conflict kecil/mudah. Analogi: merakit LEGO sedikit-sedikit terus dicek vs merakit semua sekaligus lalu disatukan — semakin lama ditunda, semakin banyak bagian yang bentrok & makin sulit menyatukan. Branch berumur panjang (Git Flow) menumpuk perubahan → saat merge, banyak baris sama berubah → conflict besar & menakutkan. Trunk-based memaksa integrasi dini sehingga masalah ketahuan selagi kecil.

20. Keuntungan Git + MR untuk infrastruktur:
    1. **Version control & audit trail** — setiap perubahan tercatat (siapa, kapan, kenapa via commit message/issue). Saat insiden, `git log`/MR adalah arsip "apa yang berubah sebelum rusak".
    2. **Review sebelum produksi** — MR memaksa peninjauan (oleh diri/orang lain) → salah ketik/konfigurasi bahaya tertangkap sebelum sampai server, berbeda dengan SSH manual yang langsung mengubah.
    3. **Reproducibility & rollback** — infra = code → server bisa dibangun ulang dari repo (idempotent, tidak ada snowflake); gagal → `git revert`/MR rollback, bukan mengingat-ingat perubahan manual.
    (bonus) GitOps pull-based (ArgoCD) → state cluster selalu sesuai repo, drift terdeteksi.
    Risiko yang harus dikelola: **secret** — jangan commit plaintext; pakai Sealed Secrets/SOPS/ansible-vault (Fase 4 & 6). Secret bocor di Git = insiden; butuh rotasi + pembersihan history.

---

## Penilaian

| Benar | Skor |
|---|---|
| 18–20 | Expert — lanjut Fase 1 (Container & OrbStack) |
| 16–17 | Lulus — boleh lanjut, perbaiki yang salah |
| 12–15 | Belum lulus — ulang materi, kerjakan ulang latihan & lab |
| < 12 | Ulangi semua materi, lanjut mentor |