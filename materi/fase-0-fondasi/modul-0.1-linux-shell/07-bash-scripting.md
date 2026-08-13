# 07 — Bash Scripting

> Tulis skrip yang tidak bikin orang termenung 10 menit membaca.

## Tujuan
- Tulis skrip yang idempotent (aman dijalankan berulang)
- Paham `exit code`, `$?`, trap, `set -euo pipefail`
- Bikin fungsi, argumen parsing,.help
- Pakai `jq`/`curl` dalam skrip

## 1. Shebang & Mode Ketat

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

**Arti:**
- `-e` — keluar saat ada perintah gagal (non-zero exit)
- `-u` — error jika variabel tak di-set
- `-o pipefail` — pipe gagal = skrip gagal (default: exit code pipe = command terakhir)
- `IFS` dibatasi — cegah bug kelihatannya spasi sebagai separator

## 2. Variabel & Quoting

```bash
NAME="alice"
echo "Hello $NAME"
echo 'Literal $NAME'                     # single-quote: tidak ada ekspansi
echo "Today: $(date)"                    # command substitution
echo "File: $NAME.txt"                   # hanya variabel saja
echo "Path: ${NAME}_file"                # eksplisit
```

**Selalu quote** variabel: `"$NAME"` bukan `$NAME`. Spasi & glob aman.

## 3. Exit Code

```bash
ls /tmp
echo "exit=$?"                           # 0 = sukses
ls /nonexistent 2>/dev/null
echo "exit=$?"                           # 2 (file/dir tidak ada)
```

Di skrip:
```bash
if ! command; then
  echo "fail" >&2
  exit 1
fi
```

Konvensi exit code:
- `0` — sukses
- `1` — error umum
- `2` — salah penggunaan opsi
- `126` — file tidak executable
- `127` — command not found
- `128+N` — proses mati dengan sinyal N (mis. 143 = SIGTERM)

## 4. Kondisional

```bash
if [[ -f file.txt ]]; then
  echo "file exists"
elif [[ -d dir ]]; then
  echo "directory exists"
else
  echo "missing"
fi

# Test operator
[[ -e file ]]   # exists
[[ -f file ]]   # regular file
[[ -d dir ]]    # directory
[[ -r file ]]   # readable
[[ -s file ]]   # size > 0
[[ -z "$str" ]] # empty string
[[ -n "$str" ]] # non-empty
[[ "$a" == "$b" ]]
[[ "$a" -eq 5 ]] # numeric
[[ "$a" =~ ^[0-9]+$ ]] # regex
```

**Catatan:** `[[ ]]` (bash) lebih kaya daripada `[ ]` (sh/posix).

## 5. Loop

```bash
for file in *.log; do
  echo "Processing $file"
done

for i in $(seq 1 5); do
  echo "$i"
done

for ((i=0; i<10; i++)); do
  echo "$i"
done

while read -r line; do
  echo "$line"
done < input.txt

count=0
until [[ "$count" -ge 3 ]]; do
  echo "$count"
  ((count++))
done
```

## 6. Fungsi

```bash
log() {
  local level="$1"
  local msg="$2"
  echo "[$(date -Is)] [$level] $msg" >&2
}

log INFO "starting"
log DEBUG "debug me"

# Argumen
greet() {
  local name="${1:-world}"
  echo "Hello $name"
}
greet "alice"
greet
```

**Selalu `local`** variabel di fungsi untuk tidak bocor ke global scope.

## 7. Argument Parsing

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
  echo "Usage: $0 -f FILE -n COUNT"
  exit 1
}

FILE=""
COUNT=10
while getopts ":f:n:" opt; do
  case $opt in
    f) FILE="$OPTARG" ;;
    n) COUNT="$OPTARG" ;;
    \?) echo "Invalid: -$OPTARG" >&2; usage ;;
    :) echo "Missing argument for -$OPTARG" >&2; usage ;;
  esac
done

[[ -z "$FILE" ]] && usage
echo "file=$FILE count=$COUNT"
```

## 8. Trap — Cleanup yang Andal

```bash
TMPDIR=$(mktemp -d)
trap 'rm -rf "$TMPDIR"' EXIT

# Interrupt trap
cleanup() {
  echo "interrupted, cleaning up" >&2
  rm -rf "$TMPDIR"
}
trap cleanup INT TERM
```

## 9. Template Skrip Produksi

```bash
#!/usr/bin/env bash
#
# backup.sh — Backup direktori ke arsip terkompresi dengan retensi 7 hari.
#
set -euo pipefail
IFS=$'\n\t'

SRC="${1:-}"
DEST="${2:-}"
DAYS="${RETENTION_DAYS:-7}"

usage() {
  cat <<EOF
Usage: $0 <SRC> <DEST>

Env:
  RETENTION_DAYS   # default 7
EOF
}

log() { printf '%s [%s] %s\n' "$(date -Is)" "$1" "$2" >&2; }

main() {
  [[ -z "$SRC" || -z "$DEST" ]] && { usage; exit 1; }
  [[ -d "$SRC" ]]             || { log ERROR "SRC not dir: $SRC"; exit 2; }

  mkdir -p "$DEST"
  local stamp ts out
  stamp=$(date +%Y%m%d-%H%M%S)
  ts=$(date +%s)
  out="${DEST%/}/backup-${stamp}.tar.gz"
  log INFO "packing $SRC -> $out"
  tar -czf "$out" -C "$(dirname "$SRC")" "$(basename "$SRC")"

  log INFO "retention: drop >${DAYS}d"
  find "$DEST" -maxdepth 1 -name "backup-*.tar.gz" -mtime +"$DAYS" -delete
  log INFO "OK"
}

main "$@"
```

## 10. Patterns Penting

### Idempotent
Fungsi boleh dijalankan 2x, hasilnya sama:
```bash
# Bagus
create_user() {
  if ! id "$1" &>/dev/null; then
    useradd "$1"
  fi
}

# Buruk
create_user() {
  useradd "$1"   # error 'already exists' saat run 2
}
```

### Lock File
```bash
LOCKFILE="/var/lock/backup.lock"
if [[ -e "$LOCKFILE" ]]; then
  echo "already running" >&2
  exit 1
fi
echo $$ > "$LOCKFILE"
trap 'rm -f "$LOCKFILE"' EXIT
```

### Parallel & xargs
```bash
ls *.log | xargs -I {} -P 4 gzip {}
seq 1 10 | xargs -I {} -P 4 curl -sO https://example.com/file{}.html
```

## 11. Debugging

```bash
set -x           # print setiap command
set +x           # stop
PS4='+ [${LINENO}] '

bash -x script.sh
bash -n script.sh    # syntax check only
```

## 12. Latihan

Lihat [evaluasi/latihan.md](evaluasi/latihan.md) hari 4.

## Ringkasan

| Aspek | Praktik terbaik |
|---|---|
| Shebang | `#!/usr/bin/env bash` |
| Mode ketat | `set -euo pipefail` |
| Quoting | `"$var"` selalu |
| Logging | Tulis ke `stderr` |
| Cleanup | `trap` |
| Lock | Cegah concurent run |
| Idempotent | Aman dijalankan berulang |

## Cek Pemahaman

1. Kenapa `set -euo pipefail` di skrip produksi?
2. Beda `[[ ]]` vs `[ ]`?
3. Kapan `trap` diperlukan?
4. Buat exit code yang disepakati tim (Lihat `/usr/include/sysexits.h`).
