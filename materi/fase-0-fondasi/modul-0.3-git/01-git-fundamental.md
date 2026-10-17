# 01 — Git Fundamental

> **Ringkasan:** Memahami cara kerja Git dari level paling mendasar menggunakan analogi dunia nyata: menguasai tiga area kerja (Working Directory, Staging Area, History), mengelola percabangan garis waktu (*branching*), menyatukan kode (*merge* vs *rebase*), menyelesaikan konflik (*conflict resolution*) dengan kepala dingin, serta membatalkan kesalahan secara aman di lingkungan kerja nyata.

---

## Tujuan Pembelajaran
Setelah membaca materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Menjelaskan perbedaan dan alur perpindahan file di **Tiga Area Git** (*Working Directory*, *Staging Area*, dan *Repository/History*).
- Menginisialisasi repositori Git dan membuat rekaman perubahan (*commit*) pertama dengan pesan yang bermakna.
- Membuat, berpindah, dan menghapus *branch* menggunakan perintah modern (`git switch`).
- Memahami perbedaan mendasar antara **Merge** dan **Rebase**, serta mengetahui kapan waktu yang tepat menggunakan masing-masing metode.
- Mengidentifikasi dan menyelesaikan benturan kode (**Merge Conflict**) tanpa panik.
- Memperbaiki atau membatalkan perubahan menggunakan alat yang tepat (`amend`, `restore`, `reset`, dan `revert`).
- Memanfaatkan **`git reflog`** sebagai jaring pengaman darurat saat kehilangan jejak *commit*.
- Menghubungkan repositori lokal ke remote server (*GitLab / GitHub*) dengan pemahaman yang benar tentang `fetch`, `pull`, dan `push`.

---

## 1. Konsep Dasar: Mengapa Git Sangat Dibutuhkan?

### Masalah Klasik: Tragedi Penamaan File Manual
Bagi siapa pun yang pernah mengerjakan dokumen tugas, skripsi, atau desain grafis, skenario folder seperti ini pasti tidak asing:

```text
📁 Laporan-Kerja/
├── Laporan_Akhir.docx
├── Laporan_Akhir_v2.docx
├── Laporan_Akhir_Revisi_Pak_Bos.docx
├── Laporan_Akhir_Beneran_FIX.docx
└── Laporan_Akhir_FINAL_BISMILLAH_JANGAN_DIUBAH_LAGI.docx
```

Cara manual di atas memiliki banyak kelemahan fatal:
1. **Boros ruang penyimpanan:** Setiap file menduplikasi 99% isi yang sebenarnya sama.
2. **Siapa mengubah apa dan kapan?** Kita tidak tahu perubahan spesifik apa yang ada di `v2` dibanding `FINAL`.
3. **Risiko tertimpa (Overwrite):** Jika dua orang bekerja bersamaan dan saling kirim file lewat chat/email, perubahan salah satu orang pasti akan hilang tertimpa.

### Git: Mesin Waktu dan Kotak Hitam (Black Box) Proyek
**Git** adalah sistem pengontrol versi terdistribusi (**Distributed Version Control System / DVCS**). Git bertindak seperti **mesin waktu** sekaligus **buku jurnal pencatat sejarah** otomatis bagi proyekmu.

```
       [Commit A] ───► [Commit B] ───► [Commit C] ───► [Commit D]
      (Inisialisasi)   (Tambah File)  (Ubah Port 80)  (Perbaiki Bug)
            ▲                                               │
            └─────────────── Kamu bisa kembali ─────────────┘
                            ke titik mana pun!
```

Dengan Git:
- Kamu hanya butuh **satu folder kerja yang bersih**.
- Git mencatat setiap perubahan dalam bentuk snapshot (foto keadaan) yang disebut **Commit**.
- Setiap *commit* memiliki identitas pembuat (*author*), waktu pasti (*timestamp*), ringkasan alasan perubahan (*commit message*), dan ID unik (*hash*).
- Kamu bisa melompat mundur ke masa lalu untuk melihat kondisi sistem saat masih berjalan normal, atau membuat cabang eksperimen tanpa takut merusak sistem yang sedang berjalan.

### Relevansi Git bagi Praktisi SRE & DevOps
Di dunia **Site Reliability Engineering (SRE)** modern, kita menganut prinsip **"Infrastructure as Code" (IaC)** dan **"GitOps"**:
- Konfigurasi server, playbook otomasi, file deklarasi cluster Kubernetes, hingga aturan alarm monitoring **semuanya disimpan dalam bentuk kode di repositori Git**.
- Git menjadi **Single Source of Truth** (sumber kebenaran tunggal). Jika server fisik di data center terbakar atau mengalami kerusakan total, kita dapat membangun ulang seluruh infrastruktur dalam hitungan menit hanya bermodalkan repositori Git.
- Tidak ada lagi perubahan manual diam-diam via SSH langsung ke server (*no snowflake infrastructure*). Setiap perubahan konfigurasi harus melalui review dan tercatat di Git.

---

## 2. Peta Mental: Tiga Area Utama Git

Kunci paling penting untuk menguasai Git—dan menghilangkan 80% kebingungan pemula—adalah memahami bahwa Git membagi wilayah kerjanya menjadi **Tiga Area**.

### Diagram Tiga Area
```
 ┌───────────────────────┐    git add <file>    ┌───────────────────────┐    git commit -m "..."    ┌───────────────────────┐
 │   Working Directory   │ ───────────────────► │  Staging Area (Index) │ ────────────────────────► │  Repository (History) │
 │                       │                      │                       │                           │                       │
 │  File nyata di disk   │ ◄─────────────────── │ Panggung foto kurasi  │ ◄──────────────────────── │  Brankas snapshot     │
 │  (bisa diedit bebas)  │     git restore      │ (siap dibungkus)      │        git revert         │  (album foto abadi)   │
 └───────────────────────┘      --staged        └───────────────────────┘                           └───────────────────────┘
```

### Analogi Studio Foto
Agar mudah dibayangkan, bayangkan kamu sedang bekerja di sebuah **Studio Foto Profesional**:

1. **Working Directory (Ruang Tunggu / Meja Kerja):**
   - Ini adalah folder fisik di laptopmu yang kamu buka di Finder atau VS Code.
   - Di sini kamu mengetik baris kode, membuat file baru, atau menghapus file lama.
   - Semua perubahan di area ini masih **mentah, bebas, dan belum dilindungi secara resmi oleh Git**.

2. **Staging Area / Index (Panggung Pemotretan):**
   - Tidak semua file yang kamu ubah di meja kerja ingin kamu foto bersamaan.
   - Misalnya: kamu mengubah 5 file, tapi baru 2 file yang selesai dan saling berkaitan.
   - Perintah `git add` adalah tindakan memanggil file tertentu untuk **naik ke atas panggung pemotretan**. File-file di area ini adalah kandidat resmi yang akan masuk ke dalam foto berikutnya.

3. **Repository / History (Album Foto Resmi / Brankas Sejarah):**
   - Saat kamu mengetik `git commit`, kamera menjepret foto!
   - Semua file yang sedang berdiri di panggung (*Staging Area*) dibungkus menjadi sebuah **Snapshot** resmi dan disimpan permanen di dalam album foto (`.git/`).
   - Begitu foto tersimpan di album, file tersebut memiliki catatan sejarah abadi yang tidak akan hilang begitu saja.

---

## 3. Status File dalam Siklus Hidup Git

File di dalam komputermu akan berpindah-pindah status di mata Git sesuai siklus berikut:

```
                  ┌─────────────── git add ──────────────┐
                  ▼                                      │
[ Untracked ] ────────► [ Staged ] ─── git commit ───► [ Unmodified ]
                            ▲                                │
                            │                                ▼
                            └───────── git add ──────── [ Modified ]
```

| Status File | Arti Sederhana | Perintah untuk Cek |
|---|---|---|
| **Untracked** | File baru dibuat di folder, Git belum pernah mencatatnya sama sekali. | `git status` (tampil merah) |
| **Modified** | File yang sudah pernah dicatat sebelumnya mengalami perubahan baris isi, tetapi belum dimasukkan ke panggung staging. | `git status` (tampil merah) |
| **Staged** | Perubahan file sudah dinaikkan ke panggung kurasi lewat `git add`, tinggal menunggu jepretan commit. | `git status` (tampil hijau) |
| **Unmodified / Committed** | File sudah aman tersimpan di snapshot commit terakhir; tidak ada perbedaan antara working directory dan repository. | `git status` ("nothing to commit, working tree clean") |

---

## 4. Konfigurasi Awal & Snapshot Pertama

Sebelum mulai membuat snapshot pertama, Git perlu mengetahui identitasmu agar setiap perubahan tercatat jelas siapa pelakunya.

### Langkah 1: Atur Identitas Global (Sekali di Komputer)
Jalankan perintah ini di terminal macOS / Linux:

```bash
# Ganti dengan nama asli dan email aktifmu
git config --global user.name "Budi Santoso"
git config --global user.email "budi.santoso@example.com"

# Tetapkan nama branch utama default menjadi 'main' (standar modern)
git config --global init.defaultBranch main
```

> 💡 **Mengapa email ini penting?**
> Email yang kamu daftarkan di sini nantinya akan dicocokkan dengan akun GitLab atau GitHub milikmu saat kamu melakukan push kode.

Untuk memverifikasi pengaturan:
```bash
git config --list
```

---

### Langkah 2: Menginisialisasi Repositori (`git init`)
Ketik perintah berikut untuk membuat proyek latihan baru:

```bash
# 1. Buat folder baru dan masuk ke dalamnya
mkdir belajar-git && cd belajar-git

# 2. Inisialisasi folder menjadi repositori Git
git init
```

*Output yang muncul:*
```text
Initialized empty Git repository in /Users/anda/belajar-git/.git/
```

> 🔍 **Apa yang terjadi di balik layar?**
> Git membuat folder tersembunyi bernama `.git/`. Di dalam folder inilah seluruh database snapshot, branch, dan riwayat revisi disimpan. **Jangan pernah menghapus folder `.git/`** kecuali kamu memang sengaja ingin membuang seluruh riwayat Git dari proyek tersebut.

---

### Langkah 3: Mengetik Kode, Staging, dan Commit

Mari kita buat file pertama:

```bash
echo "# Belajar Git SRE" > README.md
```

Periksa status proyek:
```bash
git status
```

*Output terminal:*
```text
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	README.md

nothing added to commit but untracked files present (use "git add" to track)
```
> Perhatikan teks merah pada `README.md`. File ini berstatus **Untracked**.

Sekarang, naikkan file ke panggung staging:
```bash
git add README.md
git status
```

*Output terminal:*
```text
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   README.md
```
> File berubah menjadi warna hijau di bawah tulisan `Changes to be committed`. Artinya file telah berada di **Staging Area**.

Sekarang, ambil snapshot permanen pertama:
```bash
git commit -m "feat: inisialisasi repositori dan tambah README"
```

*Output terminal:*
```text
[main (root-commit) 4a8b1c2] feat: inisialisasi repositori dan tambah README
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
```

Periksa status kembali:
```bash
git status
```
*Output terminal:*
```text
On branch main
nothing to commit, working tree clean
```

---

### Mengintip Perbedaan Isi: `git diff`
Ketika kamu mengubah file yang sudah ada, bagaimana cara melihat perubahan baris demi baris sebelum melakukan commit?

```bash
# Tambahkan satu baris baru ke README.md
echo "Git adalah fondasi GitOps dan otomasi infrastruktur." >> README.md

# 1. Cek perbedaan antara Working Directory vs Staging Area (yang belum di-add):
git diff
```

*Contoh tampilan output `git diff`:*
```diff
diff --git a/README.md b/README.md
index 28a192f..90ab3f1 100644
--- a/README.md
+++ b/README.md
@@ -1 +1,2 @@
 # Belajar Git SRE
+Git adalah fondasi GitOps dan otomasi infrastruktur.
```
> Tanda `+` berwarna hijau menunjukkan baris yang baru ditambahkan. Tanda `-` berwarna merah menunjukkan baris yang dihapus.

Jika kamu sudah menjalankan `git add README.md`, perintah `git diff` biasa akan kosong. Untuk melihat perubahan yang sudah berada di panggung staging, gunakan:
```bash
git diff --staged
# (catatan: 'git diff --cached' adalah perintah yang sama persis)
```

Simpan perubahan kedua:
```bash
git add README.md
git commit -m "docs: jelaskan peran Git untuk GitOps"
```

---

### Melihat Buku Sejarah: `git log`
Untuk melihat daftar snapshot yang pernah dibuat:

```bash
# Tampilan lengkap (Author, tanggal, pesan):
git log

# Tampilan ringkas satu baris per commit (sangat sering dipakai):
git log --oneline

# Tampilan visual diagram pohon:
git log --oneline --graph --all
```

*Contoh tampilan `git log --oneline`:*
```text
90ab3f1 (HEAD -> main) docs: jelaskan peran Git untuk GitOps
4a8b1c2 feat: inisialisasi repositori dan tambah README
```

> 🔑 **Mengenal Commit Hash:**
> Karakter acak seperti `90ab3f1` adalah kependekan dari **SHA Hash** (ID sidik jari digital unik commit tersebut). Tidak akan pernah ada dua commit yang memiliki hash yang sama di dunia!

---

## 5. Branch: Garis Hidup Paralel (Parallel Timeline)

### Apa itu Branch Sebenarnya?
Banyak pemula membayangkan membuat *branch* (cabang) berarti meng-copy seluruh isi folder proyek ke tempat lain, sehingga memakan kapasitas harddisk. **Itu keliru!**

Di Git, sebuah branch hanyalah **sebuah pointer (penunjuk/stiker nama kecil)** berukuran sekitar 41 byte yang menempel pada salah satu nomor commit. Membuat branch baru di Git berjalan instan (< 1 milidetik).

```
                      fitur-backup (pointer branch baru)
                             │
                             ▼
[Commit A] ───► [Commit B] ◄─┘
                      ▲
                      │
               main (pointer branch utama)
```

### Konsep `HEAD`: Jarum Pemutar Piringan Hitam
Jika kita punya banyak branch, bagaimana Git tahu kita sedang berdiri di branch yang mana?
Git memiliki sebuah jarum penunjuk bernama **`HEAD`**. 
- Bayangkan `HEAD` seperti jarum pada pemutar piringan hitam musik (*turntable*). Di lagu mana jarum itu menempel, lagu itulah yang sedang berputar di ruang kerjamu.
- Jika `HEAD -> main`, artinya kamu sedang berada di branch `main`.

### Perintah Modern: `git switch` (Bukan `git checkout` Kuno)
Dahulu pengguna Git menggunakan perintah serbaguna `git checkout` untuk berpindah branch sekaligus membatalkan file. Sejak Git versi 2.23+, Git memisahkan tugas tersebut agar jauh lebih mudah dipahami pemula:
- **`git switch`** khusus untuk berpindah branch.
- **`git restore`** khusus untuk mengembalikan/membatalkan file.

Mari kita praktikkan:

```bash
# 1. Membuat branch baru bernama 'fitur-backup' TANPA langsung pindah:
git branch fitur-backup

# 2. Berpindah ke branch tersebut:
git switch fitur-backup

# TIPS PRAKTIS: Buat branch baru sekaligus langsung pindah ke sana (-c = create):
git switch -c fitur-backup

# 3. Melihat daftar semua branch lokal (* menandakan branch aktif):
git branch

# 4. Kembali ke branch utama (main):
git switch main
```

> ⚠️ **Aturan Emas SRE:**
> Branch `main` mencerminkan kondisi sistem yang sedang berjalan di production. **Jangan pernah mengutak-atik kode atau konfigurasi secara langsung di branch `main`!** Selalu buat branch fitur baru untuk eksperimen atau penambahan konfigurasi, lalu gabungkan kembali setelah melalui pengujian.

---

## 6. Menyatukan Kode: Merge vs Rebase

Ketika pekerjaan di branch fitur sudah selesai, bagaimana cara menyatukan perubahan tersebut kembali ke branch `main`? Ada dua cara utama: **Merge** dan **Rebase**.

### 1. Merge (Membuat Simpul Baru, Menjaga Sejarah Apa Adanya)
Metode Merge mengambil seluruh riwayat dari branch fitur dan menyatukannya ke branch tujuan dengan membuat sebuah **Merge Commit** baru yang memiliki 2 induk (*parents*).

```
main:        A ─── B ────────────── M  (Merge Commit)
                    \             /
fitur-backup:        C ─── D ────┘
```

**Kapan menggunakan Merge?**
- Ketika kamu menggabungkan branch fitur ke branch publik (`main`, `staging`, `production`).
- Kamu ingin menjaga catatan sejarah yang jujur: kapan branch dibuat, siapa yang mengerjakan, dan kapan digabungkan.

**Cara Eksekusi Merge:**
```bash
# 1. Pindah dulu ke branch penerima (tujuan)
git switch main

# 2. Tarik dan satukan isi branch fitur
git merge fitur-backup
```

---

### 2. Rebase (Memindahkan Pondasi, Menghasilkan Riwayat Linear)
*Rebase* secara harfiah berarti "mengganti base (pondasi)". Rebase mencabut commit-commit kita di branch fitur satu per satu, lalu menempelkannya kembali di ujung commit terbaru branch `main`.

```
SEBELUM REBASE:
main:        A ─── B ─── C
                    \
fitur:               D ─── E

SETELAH REBASE (fitur di-rebase ke main):
main:        A ─── B ─── C
                          \
fitur:                     D' ─── E'  (Commit disalin ulang dengan Hash baru!)
```

Setelah rebase selesai, kita tinggal melakukan merge cepat (*Fast-Forward*) ke `main` tanpa ada simpul merge commit yang bercabang-cabang. Riwayat log proyek menjadi satu garis lurus yang sangat rapi.

**Cara Eksekusi Rebase:**
```bash
# 1. Pindah ke branch fitur yang ingin dirapikan
git switch fitur-backup

# 2. Pindahkan base-nya ke ujung main terbaru
git rebase main

# 3. Pindah kembali ke main, lalu gabungkan secara fast-forward
git switch main
git merge fitur-backup
```

---

### Tabel Perbandingan: Merge vs Rebase

| Aspek Pertimbangan | `git merge` | `git rebase` |
|---|---|---|
| **Bentuk Riwayat** | Bercabang dan ada simpul penggabungan (*graph*). | Satu garis lurus (*linear history*). |
| **Keaslian Sejarah** | Menjaga waktu dan hash commit asli 100%. | **Menulis ulang sejarah (*rewrite history*)**, hash commit berubah menjadi baru. |
| **Penyelesaian Konflik** | Cukup diselesaikan **sekali** di merge commit. | Harus diselesaikan bertahap pada **tiap commit** yang bentrok. |
| **Tingkat Risiko** | Sangat aman untuk pemula dan branch bersama. | Berbahaya jika dilakukan pada branch publik yang dipakai orang lain. |

> 🚨 **Aturan Emas Rebase (The Golden Rule of Rebasing):**
> **JANGAN PERNAH me-rebase commit yang sudah kamu push ke remote repositori publik dan sedang dikerjakan bersama anggota tim lain!**
> Rebase akan mengubah ID Hash semua commit. Jika temanmu menarik (*pull*) branch yang sudah kamu rebase secara sepihak, Git mereka akan mengalami kekacauan (*diverged history*) yang sangat sulit diperbaiki. Rebase hanya boleh digunakan untuk merapikan branch pribadimu di laptop lokal sebelum membuat Merge Request.

---

## 7. Mengatasi Benturan Kode (Conflict Resolution) Tanpa Panik

### Mengapa Konflik Bisa Terjadi?
Git adalah program yang sangat pintar. Jika Budi mengubah baris 1 sampai 10 di file A, dan Siti mengubah baris 50 sampai 60 di file A yang sama, Git akan otomatis menggabungkan keduanya tanpa masalah.

Namun, **Konflik terjadi ketika dua orang (atau dua branch) mengubah baris YANG SAMA pada file YANG SAMA dengan isi yang BERBEDA**, lalu mencoba disatukan. Git tidak berani menebak sendiri versi mana yang benar, sehingga Git menghentikan proses penggabungan dan meminta manusia untuk memutuskan.

---

### Membedah Anatomi Marker Konflik
Ketika terjadi konflik, Git akan menyisipkan tanda pembatas (*marker*) khusus langsung di dalam file yang bermasalah. Bentuknya seperti ini:

```text
<<<<<<< HEAD
Port = 8080
=======
Port = 9090
>>>>>>> fitur-ganti-port
```

Penjelasan arti simbol di atas:
- `<<<<<<< HEAD` : Awal dari kode yang ada di branch tempat kamu sedang berdiri sekarang (misal: `main`).
- `=======` : Garis pemisah antara dua versi yang bertentangan.
- `>>>>>>> fitur-ganti-port` : Akhir dari kode yang datang dari branch yang sedang kamu coba masukkan (`fitur-ganti-port`).

---

### 4 Langkah Mudah Menyelesaikan Konflik (Step-by-Step)

#### Langkah 1: Ketahui File Apa yang Konflik
Jalankan perintah andalan:
```bash
git status
```
Git akan memberi tahu dengan jelas file mana saja yang berstatus **`both modified`**.

#### Langkah 2: Buka File dan Pilih Versi yang Benar
Buka file tersebut menggunakan code editor pilihanmu (VS Code, Cursor, Vim, atau nano).
Kamu memiliki 3 pilihan keputusan:
1. Mempertahankan versi saat ini (`HEAD`).
2. Mengambil versi yang baru masuk (`fitur-ganti-port`).
3. Menggabungkan ide keduanya secara manual.

Setelah menentukan pilihan, **WAJIB HAPUS semua tanda marker** (`<<<<<<<`, `=======`, dan `>>>>>>>`) sampai file kembali menjadi kode yang bersih dan valid.

*Contoh hasil setelah dibersihkan:*
```text
Port = 9090
```

#### Langkah 3: Tandai Bahwa Konflik Sudah Diselesaikan
Katakan kepada Git bahwa kamu sudah selesai menyelesaikan masalah dengan memasukkan file tersebut kembali ke staging:
```bash
git add <nama-file>
```

#### Langkah 4: Selesaikan Proses Penggabungan
- Jika kamu sedang melakukan **Merge**:
  ```bash
  git commit -m "fix: resolve conflict pengaturan port"
  ```
- Jika kamu sedang melakukan **Rebase**:
  ```bash
  git rebase --continue
  ```

> 🛑 **Tombol Darurat (Batal Total):**
> Jika kamu merasa salah langkah, bingung, atau panik di tengah-tengah proses konflik, kamu selalu bisa membatalkan semuanya dan kembali ke kondisi sebelum perintah merge dijalankan dengan satu perintah:
> ```bash
> git merge --abort
> # atau jika sedang rebase:
> git rebase --abort
> ```

---

## 8. Mesin Waktu Penyelamat: Memperbaiki & Membatalkan Kesalahan

Salah satu ketakutan terbesar pemula adalah *"Bagaimana kalau saya salah ketik dan proyek saya rusak selamanya?"*
Kabar baiknya: di Git, hampir **tidak ada yang hilang selamanya** selama perubahan tersebut sudah pernah kamu commit!

Berikut adalah 4 tingkatan alat pembatalan di Git dari yang paling ringan sampai yang paling drastis:

```
Ringan                                                                      Drastis
  │                                                                            │
  ▼                                                                            ▼
git restore       git commit --amend        git revert          git reset --hard
(Batalkan edit    (Perbaiki pesan /         (Bikin commit baru  (Hapus commit &
 belum di-commit)  tambah file terlupa)     pembatal, aman!)     file kerja) ⚠️
```

---

### 1. `git restore` (Membatalkan Ketikan yang Belum Di-Commit)
Gunakan ini jika kamu baru saja mengedit file, tapi ingin mengembalikan isinya persis seperti commit terakhir:

```bash
# Kasus A: Mengedit file tapi belum di-add (buang perubahan di working directory):
git restore nama-file.txt

# Kasus B: Sudah terlanjur git add, tapi ingin diturunkan lagi dari staging:
git restore --staged nama-file.txt

# Kasus C: Mengambil isi file dari commit masa lalu (1 commit sebelum HEAD):
git restore --source=HEAD~1 nama-file.txt
```

---

### 2. `git commit --amend` (Menambal Commit Terakhir)
Pernahkah kamu baru saja mengetik `git commit -m "feat: login"`, lalu sedetik kemudian sadar: *"Aduh, ada satu file tertinggal!"* atau *"Pesan commit-nya salah ketik!"*?
Jangan buat commit baru hanya untuk hal sepele itu. Gunakan `amend`:

```bash
# Tambahkan file yang terlupa ke staging
git add file-yang-ketinggalan.txt

# Gabungkan ke dalam commit terakhir tanpa mengubah pesannya:
git commit --amend --no-edit

# Atau ubah pesan commit-nya:
git commit --amend -m "feat: login dengan validasi email"
```
> ⚠️ **Catatan:** `amend` mengubah ID hash commit terakhir. Gunakan hanya untuk commit lokal yang belum pernah di-push ke remote!

---

### 3. `git revert` (Paling Aman untuk Server Production & Tim)
Bagaimana jika commit yang salah sudah terlanjur di-push ke server GitLab/GitHub dan sudah ditarik oleh rekan tim lain?
**Jangan dihapus menggunakan reset!** Gunakan **`git revert`**.

`git revert` bekerja dengan cara membuat **sebuah commit baru** yang tugasnya membalikkan (*undo*) efek dari commit target.

```
Sebelum revert:
[Commit A] ───► [Commit B (Menyebabkan Error!)]

Setelah 'git revert <Hash-B>':
[Commit A] ───► [Commit B] ───► [Commit C (Revert B: Membatalkan isi B)]
```

```bash
# Buat commit pembatal untuk commit tertentu:
git revert 4a8b1c2
```
*Mengapa ini menjadi standar industri SRE?*
1. Tidak ada sejarah yang dihapus (*no rewrite history*).
2. Rekan kerja yang melakukan `git pull` tidak akan mengalami error bentrok.
3. Tercatat jelas dalam audit log kepatuhan (Siapa mematikan fitur apa, pada jam berapa, dan mengapa).

---

### 4. `git reset` (Memundurkan Jarum Jam — Pahami Perbedaannya)
`git reset` memindahkan pointer branch mundur ke commit masa lalu. Ada 3 tingkatan flag yang wajib dipahami:

```bash
# 1. --soft: Mundurkan commit, tapi kode editanmu tetap ada di STAGING AREA
git reset --soft HEAD~1

# 2. --mixed (Default jika tanpa flag): Mundurkan commit, kode kembali ke WORKING DIRECTORY
git reset --mixed HEAD~1

# 3. --hard: HAPUS COMMIT BESERTA SELURUH KODE DI WORKING DIRECTORY! ⚠️
git reset --hard HEAD~1
```

> ⚠️ **Peringatan Bahaya `--hard`:**
> `git reset --hard` memusnahkan semua perubahan yang belum di-commit secara permanen. Jangan jalankan perintah ini kecuali kamu 100% yakin bahwa pekerjaan terbarumu memang sampah yang tidak terpakai lagi.

---

### 5. `git reflog`: Jaring Pengaman Terakhir (Kotak Hitam Git)
Misalkan kamu tidak sengaja mengetik `git reset --hard` dan mengira pekerjaan 3 hari terakhir musnah selamanya. Tenang! Git mencatat setiap kali jarum `HEAD` berpindah posisi di dalam sebuah buku catatan khusus bernama **Reflog**.

```bash
git reflog
```

*Contoh tampilan `git reflog`:*
```text
90ab3f1 HEAD@{0}: reset: moving to HEAD~1
a1b2c3d HEAD@{1}: commit: feat: logika pembayaran selesai
4a8b1c2 HEAD@{2}: commit: feat: inisialisasi repo
```

Lihat baris `HEAD@{1}`! Commit `a1b2c3d` yang kamu kira sudah terhapus ternyata masih tercatat nomor hash-nya.
Kamu bisa menghidupkannya kembali dengan cara:
```bash
git reset --hard a1b2c3d
```
*Seketika seluruh file dan commit kamu yang hilang kembali utuh seperti sedia kala!*

---

## 9. Menghubungkan ke Server Remote (GitLab / GitHub)

Repositori Git di komputermu bersifat mandiri (*local*). Agar bisa berkolaborasi dengan tim atau dihubungkan ke pipeline CI/CD di server, kamu perlu menghubungkannya ke layanan **Remote Repository**.

```
  ┌───────────────────────┐                                ┌───────────────────────┐
  │      Laptop Lokal     │       git push (unggah)        │     GitLab / GitHub   │
  │                       │ ─────────────────────────────► │                       │
  │ (Working + Staging +  │                                │  (Remote Repository)  │
  │     History lokal)    │ ◄───────────────────────────── │    origin/main        │
  └───────────────────────┘     git fetch / git pull       └───────────────────────┘
                                     (unduh)
```

### Konsep Dasar Remote
1. **`origin`**: Hanyalah nama alias (*nickname*) bawaan default untuk alamat URL remote server proyekmu.
2. **`main` vs `origin/main`**: `main` adalah branch di laptopmu. `origin/main` adalah salinan catatan lokalmu tentang posisi branch `main` yang ada di server remote.

### Perintah Utama Remote
```bash
# 1. Menghubungkan repo lokal ke GitLab via SSH:
git remote add origin git@gitlab.com:nama-user/nama-repo.git

# 2. Melihat daftar remote yang terpasang:
git remote -v

# 3. Push (unggah) pertama kali dan atur upstream (-u):
git push -u origin main
# Argumen '-u' (upstream) memberitahu Git agar di masa depan kita cukup mengetik 'git push' saja.

# 4. Mengunduh perubahan terbaru dari tim:
git fetch   # Mengunduh data baru dari server TANPA mengubah file kerjamu (sangat aman untuk inspeksi)
git pull    # Mengunduh data baru DAN langsung menggabungkannya ke branch kerjamu (fetch + merge)
```

> 💡 **Praktik Terbaik SRE: Fetch Dulu, Baru Pull**
> Jika kamu sedang mengoperasikan server production, biasakan menjalankan `git fetch origin` terlebih dahulu, lalu cek perbedaannya dengan `git log HEAD..origin/main`. Setelah yakin tidak ada perubahan konfigurasi berbahaya, barulah jalankan `git merge origin/main`.

---

## 10. Hands-On Mandiri Step-by-Step (20 Menit)

Mari latih *muscle memory* tanganmu di terminal dengan menjalankan skenario laboratorium nyata dari awal sampai akhir.

### Skenario: Simulasi Konflik dan Penyelamatan Kode

```bash
# --- LANGKAH 1: Persiapan Repositori Uji Coba ---
cd ~
mkdir lab-git-fundamental && cd lab-git-fundamental
git init
echo "# Modul Latihan Git" > README.md
git add README.md
git commit -m "feat: inisialisasi modul latihan"

# --- LANGKAH 2: Membuat Perubahan Paralel yang Bertabrakan ---
# Buat branch fitur-a dan ubah baris 2 pada file app.cfg
git switch -c fitur-a
echo "SERVER_PORT=8080" > app.cfg
git add app.cfg
git commit -m "feat(app): set port ke 8080 di fitur-a"

# Kembali ke main dan ubah file yang sama dengan port berbeda
git switch main
echo "SERVER_PORT=9090" > app.cfg
git add app.cfg
git commit -m "feat(app): set port ke 9090 di main"

# --- LANGKAH 3: Memicu dan Menyelesaikan Konflik ---
git merge fitur-a
# Output: CONFLICT (add/add): Merge conflict in app.cfg

# Periksa status file konflik
git status

# Buka file app.cfg dan selesaikan isinya menjadi port standar:
cat << 'EOF' > app.cfg
SERVER_PORT=9090
EOF

# Tandai selesai dan buat merge commit
git add app.cfg
git commit -m "fix(app): satukan konfigurasi port menjadi 9090"

# Cek riwayat grafik pohon
git log --oneline --graph

# --- LANGKAH 4: Latihan Menambal Commit (Amend) ---
echo "# Konfigurasi Aplikasi Production" >> app.cfg
git add app.cfg
git commit --amend --no-edit

# --- LANGKAH 5: Latihan Penyelamatan Reflog ---
echo "CATATAN_RAHASIA=123" >> app.cfg
git add app.cfg
git commit -m "chore: commit yang tidak sengaja dibuat"

# Hapus commit tersebut secara drastis
git reset --hard HEAD~1

# Buka kotak hitam reflog dan pulihkan
git reflog
# (Cari hash commit sebelum reset, misal: HEAD@{1})
# git reset --hard HEAD@{1}

# --- LANGKAH 6: Bersihkan Folder Latihan ---
cd ..
rm -rf lab-git-fundamental
```

---

## 11. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 4 masalah yang paling sering dialami pengguna saat menggunakan Git beserta cara diagnosa dan solusinya:

### Kasus 1: Mengalami *"Detached HEAD State"*
- **Gejala:** Terminal menampilkan pesan: `You are in 'detached HEAD' state. You can look around, make experimental changes and commit them...`
- **Penyebab:** Kamu mengetik `git switch <nomor-commit>` atau `git checkout <nomor-commit>`. Jarum `HEAD` sekarang menunjuk langsung ke sebuah nomor commit di masa lalu, bukan ke sebuah nama branch! Jika kamu membuat commit di sini, commit tersebut tidak memiliki branch dan akan rawan hilang saat kamu berpindah tempat.
- **Solusi / Perbaikan:**
  - Jika hanya ingin melihat-lihat lalu kembali: `git switch main`
  - Jika ingin menyimpan perubahan eksperimenmu menjadi branch baru: `git switch -c nama-branch-baru`

---

### Kasus 2: Push Ditolak `[rejected - non-fast-forward]`
- **Gejala:** Saat mengetik `git push origin main`, muncul pesan error:
  `! [rejected] main -> main (fetch first) error: failed to push some refs...`
- **Penyebab:** Ada rekan tim lain yang sudah terlebih dahulu melakukan push commit baru ke server remote, sehingga riwayat di remote lebih baru daripada branch lokalmu.
- **Solusi / Perbaikan:**
  1. Unduh dan gabungkan perubahan dari server: `git pull origin main`
  2. Jika ada konflik, selesaikan konfliknya, lalu add dan commit.
  3. Jalankan kembali: `git push origin main`
  4. **JANGAN PERNAH** melakukan `git push --force` di branch `main` tim kecuali ada instruksi resmi dari lead arsitek!

---

### Kasus 3: File Rahasia (`.env` / SSH Key / Password) Tidak Sengaja Ter-Commit
- **Gejala:** Password database atau token API GitLab ikut terunggah ke repositori Git.
- **Penyebab:** Lupa menambahkan nama file ke dalam `.gitignore` sebelum mengetik `git add .`.
- **Solusi / Perbaikan:**
  1. Jika commit **belum di-push**:
     ```bash
     # Hapus file dari pelacakan Git tapi biarkan filenya tetap ada di komputermu:
     git rm --cached .env
     echo ".env" >> .gitignore
     git commit --amend --no-edit
     ```
  2. Jika sudah **terlanjur di-push ke remote**:
     - **Segera putar (*rotate / revoke*) token atau password tersebut di sistem backend!** Mengapa? Sekadar menghapus file di commit berikutnya tidak menghapus password tersebut dari riwayat sejarah Git masa lalu. Siapa pun masih bisa melihatnya di commit lama.
     - Tambahkan file ke `.gitignore`.

---

## 12. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Perintah Harian (Cheat Sheet)

| Kebutuhan Praktis | Perintah Terminal | Keterangan Singkat |
|---|---|---|
| **Cek status area** | `git status` | Perintah paling sering dipakai |
| **Naikkan ke panggung** | `git add <file>` atau `git add .` | Menyiapkan kandidat commit |
| **Ambil snapshot** | `git commit -m "pesan"` | Membungkus staging jadi history |
| **Intip perbedaan belum di-add** | `git diff` | Working Directory vs Staging |
| **Intip perbedaan sudah di-add** | `git diff --staged` | Staging Area vs Commit terakhir |
| **Buat & pindah branch** | `git switch -c <nama-branch>` | Isolasi fitur baru |
| **Pindah branch saja** | `git switch <nama-branch>` | Berpindah rel |
| **Satukan cabang** | `git merge <nama-branch>` | Gabungkan riwayat |
| **Batalkan edit working** | `git restore <file>` | Buang ketikan mentah |
| **Turunkan dari staging** | `git restore --staged <file>` | Unstage file |
| **Perbaiki commit terakhir** | `git commit --amend` | Edit commit lokal |
| **Batalkan commit (aman)** | `git revert <hash>` | Standar production |
| **Lihat riwayat visual** | `git log --oneline --graph --all` | Peta seluruh branch |
| **Cari commit hilang** | `git reflog` | Kotak hitam penyelamat |
| **Unggah ke server** | `git push -u origin <branch>` | Sinkronisasi remote |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan reflektif berikut untuk menguji pemahamanmu sebelum berlanjut ke modul berikutnya:

1. **Mengapa `git add` harus dilakukan sebelum `git commit`? Mengapa Git tidak langsung memotret seluruh working directory saja?**
   *(Petunjuk: Pikirkan tentang kurasi panggung foto dan keleluasaan membagi perubahan ke dalam commit-commit terpisah).*
2. **Jika sebuah server production mengalami insiden bug fatal akibat commit terakhir yang baru saja di-deploy ke branch `main`, apakah kamu akan memilih `git reset --hard` atau `git revert`? Jelaskan alasan operasionalnya!**
3. **Apa perbedaan praktis antara `git fetch` dan `git pull`? Mengapa seorang SRE disarankan lebih sering melakukan `fetch` terlebih dahulu?**
4. **Temanmu berkata: *"Pekerjaan saya hilang karena saya salah mengetik git reset --hard!"*. Alat apa di Git yang bisa kamu rekomendasikan untuk menyelamatkannya, dan bagaimana cara kerjanya?**

---
*Lanjutkan ke materi berikutnya: [02-git-workflow.md](02-git-workflow.md) untuk mempelajari alur kerja tim profesional (Trunk-Based Development), standar Conventional Commits, dan kolaborasi via Merge Request di GitLab.*