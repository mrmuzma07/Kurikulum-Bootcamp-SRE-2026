# 07 — Bash Scripting untuk SRE

> **Ringkasan:** Belajar menulis skrip Bash yang **aman, mudah dibaca, dan idempoten** (aman dijalankan berulang kali) untuk mengotomasi tugas operasional: mode ketat `set -euo pipefail`, variabel & *quoting*, exit code, kondisional, loop, fungsi, parsing argumen, `trap` untuk *cleanup*, lock file, hingga teknik debugging. Inilah fondasi untuk mengganti kerja manual berulang (*toil*) dengan otomasi.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal, kamu akan mampu:
- Membuat dan menjalankan skrip Bash pertama (shebang, `chmod +x`, cara memanggil).
- Menerapkan **mode ketat** `set -euo pipefail` dan memahami alasan setiap opsinya.
- Memakai variabel dan **quoting** dengan benar agar skrip tidak rusak oleh spasi atau karakter khusus.
- Membaca dan memakai **exit code** (`$?`) sebagai bahasa komunikasi antar perintah.
- Menulis `if`, `case`, `for`, `while`, fungsi, dan parsing argumen (`getopts`).
- Menggunakan `trap` untuk membersihkan file sementara dan **lock file** untuk mencegah skrip berjalan ganda.
- Menulis skrip yang **idempoten** dan men-debug skrip dengan `bash -x` / `bash -n`.

---

## 1. Mengapa SRE Menulis Skrip?

> 🤖 **Analogi Robot Asisten:** Skrip adalah **daftar instruksi tertulis untuk robot asisten**. Alih-alih kamu mengetik 15 perintah manual setiap malam (rentan salah ketik, lupa langkah), kamu menulisnya sekali, lalu robot (shell) mengeksekusinya persis sama setiap saat.

Prinsip dari kurikulum ini: **No Snowflake Infrastructure** dan **kurangi toil**. Perubahan yang dilakukan manual di server sulit dilacak dan tidak bisa diulang. Skrip yang tersimpan di Git adalah *resep yang bisa diaudit dan diulang*.

Ciri skrip operasional yang baik:
1. **Aman gagal:** berhenti saat ada kesalahan, bukan melanjutkan dan merusak data.
2. **Idempoten:** dijalankan 2x hasilnya sama, tidak error "already exists".
3. **Jelas:** punya pesan `usage`, logging, dan exit code bermakna.
4. **Bersih:** tidak meninggalkan file sampah atau proses menggantung.

---

## 2. Skrip Pertama: Shebang & Cara Menjalankan

```bash
mkdir -p ~/latihan-bash && cd ~/latihan-bash

cat > halo.sh << 'EOF'
#!/usr/bin/env bash
echo "Halo dari skrip! User: $(whoami), Host: $(hostname)"
EOF

ls -l halo.sh            # belum ada 'x' (belum executable)
chmod +x halo.sh         # beri izin eksekusi
./halo.sh                # jalankan
bash halo.sh             # alternatif: tanpa perlu izin x
```

**Bedah baris pertama `#!/usr/bin/env bash` (Shebang):**
- `#!` dibaca *"shebang"*. Memberi tahu kernel: *"Jalankan file ini menggunakan interpreter berikut."*
- `/usr/bin/env bash` mencari `bash` di `$PATH`, sehingga lebih portabel daripada menulis `/bin/bash` (misal di macOS bash ada di lokasi berbeda).

> 💡 Mengapa `./halo.sh` dan bukan `halo.sh`? Karena folder saat ini (`.`) sengaja **tidak** ada di `$PATH` demi keamanan. Kamu harus menyebut lokasinya eksplisit.

---

## 3. Mode Ketat: `set -euo pipefail` (Wajib di Setiap Skrip)

Tanpa pengaman, Bash secara bawaan **melanjutkan eksekusi meski perintah sebelumnya gagal**. Ini berbahaya.

> 🚗 **Analogi:** Bayangkan skrip deploy: langkah 1 `cd /opt/app` gagal (folder tak ada), tetapi skrip terus lanjut ke langkah 2 `rm -rf *`. Kini `rm -rf *` dijalankan di folder yang salah!

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

| Opsi | Arti | Efek Nyata |
|---|---|---|
| **`-e`** | *errexit* | Skrip **berhenti** begitu ada perintah yang exit code-nya bukan 0. |
| **`-u`** | *nounset* | Memakai variabel yang belum di-set → **error** (mencegah `rm -rf "$DIR/"` saat `$DIR` kosong). |
| **`-o pipefail`** | pipe gagal = gagal | Di `cmd1 \| cmd2`, jika `cmd1` gagal, seluruh pipeline dianggap gagal (default hanya melihat `cmd2`). |
| **`IFS=$'\n\t'`** | pemisah kata | Memecah kata hanya pada newline/tab, bukan spasi → nama file ber-spasi lebih aman. |

**Demo perbedaannya:**
```bash
# TANPA pipefail: dianggap sukses walau 'cat' gagal
bash -c 'cat /tidak-ada | wc -l; echo "exit=$?"'
# exit=0   <-- menipu!

# DENGAN pipefail:
bash -c 'set -o pipefail; cat /tidak-ada | wc -l; echo "exit=$?"'
# exit=1   <-- jujur
```

> ⚠️ **Jebakan `set -e`:** Perintah yang *sengaja* boleh gagal harus ditangani, misal `grep -q "x" file || true`, atau dipakai dalam `if`. Contoh klasik: `((count++))` ketika `count=0` menghasilkan exit code 1 dan **mematikan skrip**. Gunakan `count=$((count + 1))`.

---

## 4. Variabel & Quoting (Sumber Bug Nomor 1)

```bash
NAME="alice"                       # TANPA spasi di sekitar '='!
echo "Halo $NAME"                  # double quote: variabel DIEKSPANSI
echo 'Literal $NAME'               # single quote: apa adanya, TIDAK diekspansi
echo "Hari ini: $(date +%F)"       # command substitution: hasil perintah
echo "File: ${NAME}_backup.txt"    # kurung kurawal: batas nama variabel jelas
```

> ❌ `NAME = "alice"` (pakai spasi) → Bash mengira `NAME` adalah *perintah*. Tulis `NAME="alice"`.

### Aturan Emas: **Selalu beri tanda kutip pada variabel (`"$VAR"`)**
```bash
FILE="laporan akhir.txt"           # nama file mengandung spasi

rm $FILE                           # SALAH: jadi rm "laporan" "akhir.txt" (dua argumen!)
rm "$FILE"                         # BENAR: satu argumen utuh
```

### Nilai Default & Variabel Penting
```bash
DAYS="${RETENTION_DAYS:-7}"        # pakai $RETENTION_DAYS, jika kosong → 7
echo "Nama skrip: $0"              # $0 = nama skrip
echo "Argumen 1: ${1:-kosong}"     # $1, $2, ... = argumen posisi
echo "Jumlah argumen: $#"          # $# = banyak argumen
echo "Semua argumen: $@"           # $@ = semua argumen (pakai "$@" dengan kutip)
echo "PID skrip: $$"               # $$ = PID proses skrip ini
```

---

## 5. Exit Code: Bahasa Sukses & Gagal

Setiap perintah Linux mengakhiri diri dengan **angka status** (0–255). Angka itu disimpan di **`$?`**.

```bash
ls /tmp
echo "exit=$?"                     # 0 = sukses

ls /folder-tidak-ada 2>/dev/null
echo "exit=$?"                     # bukan 0 = gagal (misal 2)
```

| Exit Code | Arti Konvensional |
|:---:|---|
| **`0`** | Sukses |
| **`1`** | Error umum |
| **`2`** | Salah penggunaan perintah/opsi |
| **`126`** | File ditemukan tetapi tidak executable |
| **`127`** | *Command not found* |
| **`128+N`** | Mati karena sinyal N (mis. **143** = `SIGTERM`, **137** = `SIGKILL`) |

> 🔗 Itulah sebabnya di Kubernetes **Exit Code 137** berarti `OOMKilled` (128 + 9). Pengetahuan ini akan terpakai di Fase 2!

**Memakai exit code di skrip:**
```bash
if ! grep -q "ERROR" app.log; then
  echo "[INFO] Tidak ada error" >&2
fi

# Operator logika singkat:
mkdir -p /tmp/x && echo "dibuat"       # && : jalankan kanan HANYA jika kiri sukses
rm /tmp/tidak-ada 2>/dev/null || echo "gagal, lanjut"   # || : jalankan kanan HANYA jika kiri gagal

exit 0                                  # akhiri skrip dengan status sukses
```

---

## 6. Kondisional: `if`, `[[ ]]`, dan `case`

```bash
if [[ -f "/etc/hosts" ]]; then
  echo "File ada"
elif [[ -d "/etc/hosts" ]]; then
  echo "Ini direktori"
else
  echo "Tidak ada"
fi
```

### Operator Tes yang Sering Dipakai
| Ekspresi | Benar jika... |
|---|---|
| `[[ -e "$p" ]]` | path ada (file/dir apa pun) |
| `[[ -f "$p" ]]` | file biasa |
| `[[ -d "$p" ]]` | direktori |
| `[[ -r "$p" ]]` / `-w` / `-x` | bisa dibaca / ditulis / dieksekusi |
| `[[ -s "$p" ]]` | file ada dan ukuran > 0 |
| `[[ -z "$s" ]]` | string kosong |
| `[[ -n "$s" ]]` | string tidak kosong |
| `[[ "$a" == "$b" ]]` | string sama |
| `[[ "$n" -eq 5 ]]` | angka sama (`-ne`, `-gt`, `-ge`, `-lt`, `-le`) |
| `[[ "$s" =~ ^[0-9]+$ ]]` | cocok dengan regex |

> 💡 **`[[ ]]` vs `[ ]`:** `[[ ]]` adalah fitur Bash yang lebih aman (tidak error pada variabel kosong, mendukung `&&`, `||`, `=~`). `[ ]` adalah versi POSIX lama. **Di skrip Bash, pakai `[[ ]]`.**

### `case` untuk Banyak Pilihan
```bash
case "${1:-}" in
  start)   echo "Memulai..." ;;
  stop)    echo "Menghentikan..." ;;
  restart) echo "Restart..." ;;
  *)       echo "Pakai: $0 {start|stop|restart}" >&2; exit 2 ;;
esac
```

---

## 7. Perulangan (Loop)

```bash
# 1. for: daftar file
for file in *.log; do
  [[ -e "$file" ]] || continue         # lewati jika tidak ada file cocok
  echo "Memproses $file"
done

# 2. for: rentang angka
for i in $(seq 1 5); do echo "ke-$i"; done
for ((i = 0; i < 3; i++)); do echo "hitung $i"; done

# 3. while read: membaca file baris demi baris (CARA AMAN)
while IFS= read -r line; do
  echo "Baris: $line"
done < input.txt

# 4. until: ulangi sampai kondisi benar
count=0
until [[ "$count" -ge 3 ]]; do
  echo "count=$count"
  count=$((count + 1))              # AMAN dengan set -e (bukan ((count++)))
done
```

**Pola retry untuk health check (sering dipakai SRE):**
```bash
for attempt in 1 2 3 4 5; do
  if curl -fsS --max-time 3 http://localhost:8080/health >/dev/null; then
    echo "[INFO] Service sehat"
    break
  fi
  echo "[WARN] Percobaan $attempt gagal, tunggu 2 detik" >&2
  sleep 2
done
```

---

## 8. Fungsi

Fungsi membungkus langkah yang dipakai berulang agar skrip rapi.

```bash
log() {
  local level="$1"                 # 'local' = variabel hanya hidup di dalam fungsi
  local msg="$2"
  printf '%s [%s] %s\n' "$(date -Is)" "$level" "$msg" >&2
}

greet() {
  local name="${1:-dunia}"         # default 'dunia' jika tak ada argumen
  echo "Halo $name"
}

log INFO "memulai skrip"
greet "alice"
greet
```

- **Selalu pakai `local`** untuk variabel di fungsi agar tidak "bocor" dan menimpa variabel global.
- **Log ke `stderr` (`>&2`)**, sedangkan hasil data ke `stdout`. Dengan begitu `hasil=$(./skrip.sh)` hanya menangkap data, bukan pesan log.
- Fungsi "mengembalikan" status lewat `return N` (exit code), dan "mengembalikan" nilai lewat `echo` yang ditangkap `$(...)`.

---

## 9. Parsing Argumen dengan `getopts`

Agar skrip bisa dipanggil seperti perintah profesional: `./skrip.sh -f data.txt -n 5`.

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  echo "Usage: $0 -f FILE [-n COUNT]" >&2
  exit 2
}

FILE=""
COUNT=10
while getopts ":f:n:h" opt; do
  case "$opt" in
    f) FILE="$OPTARG" ;;
    n) COUNT="$OPTARG" ;;
    h) usage ;;
    \?) echo "[ERROR] Opsi tidak dikenal: -$OPTARG" >&2; usage ;;
    :)  echo "[ERROR] Opsi -$OPTARG butuh argumen" >&2; usage ;;
  esac
done

[[ -z "$FILE" ]] && usage
[[ -f "$FILE" ]] || { echo "[ERROR] File tidak ada: $FILE" >&2; exit 1; }
echo "[INFO] file=$FILE count=$COUNT"
head -n "$COUNT" "$FILE"
```
*(Tanda `:` setelah huruf pada `"f:n:"` berarti opsi itu membutuhkan nilai.)*

---

## 10. `trap` — Pembersihan Otomatis Seperti `finally`

Jika skrip membuat file sementara lalu **berhenti di tengah jalan** (error, `Ctrl+C`), file sampah akan tertinggal. `trap` menjamin pembersihan tetap dijalankan.

```bash
WORKDIR="$(mktemp -d)"                 # buat folder sementara unik yang aman
cleanup() {
  rm -rf "$WORKDIR"
  echo "[INFO] Bersih-bersih selesai" >&2
}
trap cleanup EXIT                       # EXIT = jalan saat skrip berakhir, sukses ATAU gagal

echo "kerja di $WORKDIR"
false                                   # (dengan set -e, skrip berhenti di sini, cleanup tetap jalan)
```

Sinyal umum untuk `trap`: **`EXIT`** (selalu saat selesai), **`INT`** (Ctrl+C), **`TERM`** (`kill`). Memakai `EXIT` saja biasanya sudah cukup.

---

## 11. Pola Produksi: Idempoten & Lock File

### A. Idempoten — Aman Dijalankan Berulang
```bash
# BURUK: gagal pada eksekusi kedua ("user already exists")
useradd deploy

# BAGUS: cek dulu keadaan sebelum bertindak
if ! id deploy &>/dev/null; then
  useradd deploy
  echo "[INFO] User deploy dibuat"
else
  echo "[INFO] User deploy sudah ada, lewati"
fi

mkdir -p /opt/app                       # -p sudah idempoten
ln -sfn /opt/app/releases/v2 /opt/app/current   # -f menimpa symlink lama
```
Ukuran keberhasilan: **jalankan skrip dua kali — keadaan akhir dan hasilnya sama, tanpa error.**

### B. Lock File — Cegah Skrip Berjalan Ganda
Bayangkan skrip backup dijadwalkan tiap 5 menit, tetapi satu eksekusi butuh 8 menit. Dua backup berjalan bersamaan → data kacau. Gunakan `flock`, yang atomik (lebih aman daripada mengecek `[[ -e lockfile ]]` manual yang rawan *race condition*):

```bash
LOCKFILE="/tmp/backup.lock"
exec 200>"$LOCKFILE"
flock -n 200 || { echo "[ERROR] Skrip sudah berjalan, keluar" >&2; exit 1; }
# ... pekerjaan utama di bawah sini; lock otomatis lepas saat skrip selesai ...
```

### C. Paralel Sederhana dengan `xargs`
```bash
# Kompres semua .log dengan maksimal 4 proses paralel
ls *.log | xargs -I {} -P 4 gzip {}
```

---

## 12. Template Skrip Produksi: `backup.sh`

Gabungan semua praktik di atas. Pelajari baris demi baris:

```bash
#!/usr/bin/env bash
#
# backup.sh — Backup direktori ke arsip .tar.gz dengan retensi (default 7 hari).
# Usage: ./backup.sh <SRC_DIR> <DEST_DIR>
# Env:   RETENTION_DAYS (default 7)
#
set -euo pipefail
IFS=$'\n\t'

SRC="${1:-}"
DEST="${2:-}"
DAYS="${RETENTION_DAYS:-7}"

usage() {
  cat << EOF
Usage: $0 <SRC_DIR> <DEST_DIR>

Env:
  RETENTION_DAYS   Lama simpan arsip dalam hari (default 7)
EOF
}

log() { printf '%s [%s] %s\n' "$(date -Is)" "$1" "$2" >&2; }

main() {
  [[ -z "$SRC" || -z "$DEST" ]] && { usage; exit 1; }
  [[ -d "$SRC" ]] || { log ERROR "SRC bukan direktori: $SRC"; exit 2; }

  mkdir -p "$DEST"
  local stamp out
  stamp="$(date +%Y%m%d-%H%M%S)"
  out="${DEST%/}/backup-${stamp}.tar.gz"

  log INFO "mengarsip $SRC -> $out"
  tar -czf "$out" -C "$(dirname "$SRC")" "$(basename "$SRC")"

  log INFO "retensi: hapus arsip lebih dari ${DAYS} hari"
  find "$DEST" -maxdepth 1 -name "backup-*.tar.gz" -mtime +"$DAYS" -delete

  log INFO "SELESAI"
}

main "$@"
```

**Mengapa ini disebut "production-ready"?** Ada mode ketat, `usage`, validasi input, exit code berbeda (`1` salah pakai, `2` input salah), logging ke `stderr` berformat waktu, nilai default yang bisa di-override env, dan pembersihan arsip lama.

---

## 13. Debugging Skrip

```bash
bash -n skrip.sh          # cek SINTAKS saja, tanpa menjalankan
bash -x skrip.sh          # jalankan sambil mencetak SETIAP perintah (trace)
```

Mengaktifkan trace hanya pada bagian tertentu:
```bash
PS4='+ [${LINENO}] '       # tampilkan nomor baris di setiap baris trace
set -x                     # mulai trace
sesuatu_yang_dicurigai
set +x                     # hentikan trace
```

Alat tambahan yang sangat dianjurkan: **`shellcheck`** (linter yang menemukan bug umum seperti variabel tanpa kutip).
```bash
sudo apt install -y shellcheck
shellcheck backup.sh
```

---

## 14. Pola Kegagalan (Failure Modes) yang Sering Terjadi

| Gejala | Penyebab | Solusi |
|---|---|---|
| `rm -rf "$DIR/"*` menghapus hal tak terduga | `$DIR` kosong/tidak di-set | Pakai `set -u`, atau `${DIR:?variabel DIR kosong}` |
| Skrip jalan di manual tapi gagal di `cron` | `PATH` dan environment cron minim | Pakai path absolut (`/usr/bin/curl`), set `PATH` di skrip |
| Skrip mati misterius di tengah jalan | `set -e` + `((count++))` saat nilai 0, atau `grep` tanpa hasil | Pakai `count=$((count+1))`, `grep ... \|\| true` bila boleh gagal |
| Nama file berspasi merusak skrip | Variabel tanpa tanda kutip | Selalu `"$VAR"` |
| `bad interpreter: No such file or directory` | Baris baru Windows (CRLF) di skrip | `sed -i 's/\r$//' skrip.sh` |
| `Permission denied` saat `./skrip.sh` | Belum `chmod +x` atau partisi `noexec` | `chmod +x`, atau jalankan `bash skrip.sh` |

---

## 15. Latihan Praktik Mandiri (15 Menit di Terminal VM OrbStack)

Buat skrip pemeriksa disk sederhana yang idempoten dan aman:

```bash
mkdir -p ~/latihan-bash && cd ~/latihan-bash

cat > cek-disk.sh << 'EOF'
#!/usr/bin/env bash
#
# cek-disk.sh — Peringatan jika pemakaian disk melewati ambang batas.
# Usage: ./cek-disk.sh [-p PATH] [-t THRESHOLD]
#
set -euo pipefail
IFS=$'\n\t'

TARGET="/"
THRESHOLD=80

usage() { echo "Usage: $0 [-p PATH] [-t THRESHOLD]" >&2; exit 2; }
log()   { printf '%s [%s] %s\n' "$(date -Is)" "$1" "$2" >&2; }

while getopts ":p:t:h" opt; do
  case "$opt" in
    p) TARGET="$OPTARG" ;;
    t) THRESHOLD="$OPTARG" ;;
    h) usage ;;
    \?) log ERROR "opsi tidak dikenal: -$OPTARG"; usage ;;
    :)  log ERROR "opsi -$OPTARG butuh argumen"; usage ;;
  esac
done

[[ "$THRESHOLD" =~ ^[0-9]+$ ]] || { log ERROR "THRESHOLD harus angka"; exit 2; }
[[ -d "$TARGET" ]] || { log ERROR "path tidak ada: $TARGET"; exit 2; }

USED="$(df --output=pcent "$TARGET" | tail -n 1 | tr -dc '0-9')"

if (( USED >= THRESHOLD )); then
  log WARN "Disk $TARGET terpakai ${USED}% (ambang ${THRESHOLD}%)"
  exit 1
fi

log INFO "Disk $TARGET aman: ${USED}% (ambang ${THRESHOLD}%)"
EOF

chmod +x cek-disk.sh
bash -n cek-disk.sh                 # 1. cek sintaks
shellcheck cek-disk.sh || true      # 2. (opsional) linter

./cek-disk.sh                       # 3. jalan normal
echo "exit=$?"
./cek-disk.sh -t 1                  # 4. ambang rendah → harus WARN, exit=1
echo "exit=$?"
./cek-disk.sh -p /tidak-ada         # 5. input salah → exit=2
echo "exit=$?"
./cek-disk.sh -t abc                # 6. validasi angka → exit=2
echo "exit=$?"

bash -x ./cek-disk.sh 2>&1 | head -20   # 7. lihat trace eksekusi

# Bersihkan
cd ~ && rm -rf ~/latihan-bash
```

**Acceptance Criteria:**
- [ ] Skrip lolos `bash -n` tanpa error.
- [ ] Exit code `0` saat aman, `1` saat melewati ambang, `2` saat input tidak valid.
- [ ] Pesan log keluar di `stderr` dengan format waktu dan level.
- [ ] Skrip dijalankan berulang kali dengan hasil konsisten (idempoten, tidak mengubah apa pun).

Latihan lanjutan ada di [evaluasi/latihan.md](evaluasi/latihan.md) (Hari 4).

---

## 16. Ringkasan (Cheat Sheet)

| Aspek | Praktik Terbaik |
|---|---|
| Shebang | `#!/usr/bin/env bash` |
| Mode ketat | `set -euo pipefail` |
| Variabel | `"$var"` selalu dikutip; `${VAR:-default}` untuk nilai bawaan |
| Kondisi | `[[ ... ]]`, bukan `[ ... ]` |
| Logging | Tulis ke `stderr` (`>&2`) dengan level `[INFO]/[WARN]/[ERROR]` |
| Exit code | `0` sukses; bedakan `1`, `2`, dst. dan dokumentasikan |
| Cleanup | `trap cleanup EXIT` |
| Anti-ganda | `flock -n` |
| Idempoten | Cek keadaan dulu; pakai `mkdir -p`, `ln -sfn` |
| Debug | `bash -n`, `bash -x`, `shellcheck` |

---

## 17. Cek Pemahaman (Self-Check)

1. **Mengapa skrip produksi sebaiknya diawali `set -euo pipefail`? Jelaskan fungsi masing-masing opsi.**
2. **Apa beda `[[ ]]` dengan `[ ]`, dan mengapa variabel harus selalu diberi tanda kutip?**
3. **Kapan `trap ... EXIT` diperlukan? Berikan contoh.**
4. **Mengapa `((count++))` bisa mematikan skrip yang memakai `set -e`, dan apa gantinya?**
5. **Rancang skema exit code untuk skrip backup tim kamu (sukses, salah penggunaan, sumber tidak ada, disk penuh).**

<details>
<summary><b>🔑 Klik untuk melihat Kunci Jawaban & Penjelasan</b></summary>

1. **`set -euo pipefail`:**
   - `-e`: berhenti saat ada perintah gagal agar kesalahan tidak merambat.
   - `-u`: error jika memakai variabel belum di-set (mencegah bug seperti `rm -rf "$DIR/"*` saat `$DIR` kosong).
   - `-o pipefail`: kegagalan di tengah pipeline terdeteksi, bukan hanya status perintah terakhir.
   Tujuannya: skrip **gagal cepat dan jelas** alih-alih diam-diam merusak data.

2. **`[[ ]]` vs `[ ]`:** `[[ ]]` adalah fitur Bash yang lebih aman dan kaya (mendukung `&&`, `||`, regex `=~`, tidak pecah oleh spasi pada variabel). `[ ]` adalah versi POSIX lama yang lebih rapuh. Variabel dikutip karena tanpa kutip, nilai berspasi atau kosong akan **dipecah menjadi beberapa argumen** sehingga perintah berperilaku salah atau berbahaya.

3. **`trap ... EXIT`:** diperlukan saat skrip membuat sumber daya sementara (folder `mktemp -d`, lock file, mount) yang **harus dibersihkan walaupun skrip gagal atau dihentikan**. Contoh: `trap 'rm -rf "$WORKDIR"' EXIT`.

4. **`((count++))` dan `set -e`:** ekspresi aritmetika `((...))` menghasilkan exit code `1` jika nilainya `0`. Saat `count=0`, `((count++))` bernilai 0 sebelum bertambah sehingga exit code 1 dan `set -e` mematikan skrip. Gantinya: `count=$((count + 1))`.

5. **Contoh skema exit code:**
   - `0` = sukses
   - `1` = salah penggunaan (argumen kurang)
   - `2` = direktori sumber tidak ada
   - `3` = disk tujuan penuh / gagal menulis arsip
   Dokumentasikan di bagian `usage`/komentar agar konsisten bagi seluruh tim (rujukan: `/usr/include/sysexits.h`).

</details>
