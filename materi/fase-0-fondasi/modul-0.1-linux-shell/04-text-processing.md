# 04 — Text Processing

> `grep`, `awk`, `sed`, `jq`, pipe, redirect. Senjata inti SRE.

## Tujuan
- Filter & cari teks dalam file besar
- Transformasi data dengan `sed` / `awk`
- Baca JSON output API dengan `jq`
- Bikin pipeline: input → filter → output

## 1. Pipes & Redirect — Pondasi

```bash
command1 | command2                          # stdout cmd1 -> stdin cmd2
command > file.txt                           # stdout redirect (overwrite)
command >> file.txt                          # append
command 2> err.log                           # stderr redirect
command > out.log 2>&1                       # gabung stdout+stderr
command < input.txt                          # stdin dari file
command <> file.txt                          # both directions
command1 | tee log.txt | grep "ERROR"        # tampil + simpan
```

**`tee` = fork output** — tampilkan di layar sekaligus simpan ke file.

## 2. `grep` — Cari Pola

```bash
grep "ERROR" file.txt
grep -i "error" file.txt                     # case-insensitive
grep -n "error" file.txt                     # nomor baris
grep -v "DEBUG" file.log                    # excluded
grep -E "(error|warn)" file.log            # regex extended
grep -A 3 "ERROR" file.log                  # 3 baris setelah match
grep -B 3 "ERROR" file.log                  # 3 baris sebelum match
grep -C 3 "ERROR" file.log                  # konteks 3
grep -r "TODO" /opt/app/src/                # rekursif
grep -L "test" *.go                          # file yang TIDAK match
grep -c "200" access.log                     # hitung jumlah
grep -l "ERROR" *.log                        # daftar file yang match
grep -P "\d{3,}" file.txt                   # Perl regex
```

**Regex mini:**
- `^` awal baris, `$` akhir baris
- `.` satu karakter, `.*` zero or more
- `[abc]` salah satu, `[^abc]` bukan
- `\b` word boundary
- `+` (ext) satu atau lebih, `?` nol/satu, `{n,m}` n..m

## 3. `sed` — Edit Stream

`sed` = editor non-interaktif. Default tidak mengubah file, cetak ke stdout.

```bash
sed 's/old/new/' file.txt                   # first match per line
sed 's/old/new/g' file.txt                  # global, all matches
sed 's/old/new/gi' file.txt                 # case-insensitive
sed -E 's/(foo)(bar)/\2\1/' file.txt        # group
sed -n '1,5p' file.txt                      # print line 1-5
sed -n '/ERROR/p' file.log                  # print baris match
sed '/DEBUG/d' file.log                     # hapus baris DEBUG
sed -i.bak 's/old/new/g' file.txt           # in-place + backup
```

**Contoh berguna:**
```bash
# Hapus baris kosong
sed '/^$/d' file.txt

# Replace IPAddress in-place (cadangkan dulu)
sed -i.bak 's/192.168.1.10/10.0.0.10/g' /etc/app/config.yaml

# Prefix setiap baris dengan timestamp epoch
sed "s/^/$(date +%s) /" file.txt
```

## 4. `awk` — Olah Data Kolom

```bash
awk '{print $1}' file.txt                   # kolom 1
awk '{print $1, $3}' file.txt               # kolom 1 & 3
awk -F: '{print $1}' /etc/passwd            # field separator
awk -F, 'NR>1 {sum+=$3} END {print sum}' data.csv
awk '$3 > 80 {print $1, $3}' file.txt       # filter numerik
awk '/ERROR/ {print}' file.log              # match pola
awk 'BEGIN{OFS="|"} {print $1,$2}' file    # output separator
```

**Variabel bawaan:**
- `NR` — nomor baris saat ini
- `NF` — jumlah field di baris
- `$0` — seluruh baris
- `$1..$N` — field ke-N
- `FILENAME`, `FS`, `OFS`, `RS`, `ORS`

Teknik:
```bash
# Top 5 IP yang paling sering di access.log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Rata-rata dari kolom
awk '{s+=$1; n++} END {if(n) print s/n}' file.txt

# Konversi CSV ke TSV
awk -F, 'BEGIN{OFS="\t"} {$1=$1; print}' file.csv
```

## 5. `jq` — Olah JSON

Penting banget. Output API, Terraform, dan semuanya JSON.

```bash
echo '{"name":"alice","age":30}' | jq .

# Pilih field
jq '.name' api.json
jq '.users[0].email' data.json
jq '.users[] | .email' data.json

# Filter
jq '.users[] | select(.active)' data.json
jq '.items[] | select(.price > 100)' data.json

# Map & transform
jq '.items | map(.price * 2)' data.json
jq '.items | length' data.json

# Tabular output
jq -r '.[] | [.id, .name] | @tsv' data.json

# Stream JSON besar
jq -c '.events[]' big.json
```

**Latihan jq:**
```bash
# Buat data contoh
cat > /tmp/jq-test.json <<EOF
[
  {"name":"alice","role":"dev","salary":9000},
  {"name":"bob","role":"ops","salary":7500},
  {"name":"carol","role":"dev","salary":11000}
]
EOF

# 1. Tampilkan semua nama
jq -r '.[].name' /tmp/jq-test.json

# 2. Siapa yang gajinya > 8000?
jq -r '.[] | select(.salary > 8000) | .name' /tmp/jq-test.json

# 3. Rata-rata gaji
jq '[.[].salary] | add / length' /tmp/jq-test.json

# 4. Group by role
jq 'group_by(.role) | map({role: .[0].role, count: length, avg: ([.[].salary] | add / length)})' /tmp/jq-test.json
```

## 6. Utilities Pendukung

```bash
sort file.txt                # urut
sort -n file.txt             # numeric
sort -u file.txt             # unik
sort -k2 file.txt            # by kolom 2
sort -t: -k3 -n /etc/passwd

uniq                         # adj dedupe (sort dulu!)
uniq -c                      # hitung
wc -l file.txt               # baris
wc -c file.txt               # bytes
cut -d: -f1,7 /etc/passwd    # kolom 1 & 7
tr 'a-z' 'A-Z' < file.txt    # translate
xargs                        # ubah stdin jadi argumen
```

## 7. Pipeline Penting — SRE

```bash
# Top 10 endpoint status 5xx
awk '$9 ~ /^5/' access.log | awk '{print $7}' | sort | uniq -c | sort -rn | head

# Error unik dari log (abaikan timestamp)
grep "ERROR" app.log | sed -E 's/^\S+ //' | sort -u

# Dari proses melingkar, lihat PID saat ini
ps -ef | grep "worker" | grep -v grep | awk '{print $2}'

# Hitung jumlah 504 per menit
grep -oE '...[0-9]{2}:"504"' access.log | sort | uniq -c

# Disk usage terbesar di /var
du -ah /var/ | sort -h | tail -20

# List pod belum Running
kubectl get pods -A | awk '$4 != "Running"'
```

## 8. Latihan Terpadu

```bash
# Bikin data contoh
cat > /tmp/access.log <<'EOF'
192.168.1.1 - - [10/Oct/2024:13:55:00] "GET /api/v1/users HTTP/1.1" 200 1234
192.168.1.2 - - [10/Oct/2024:13:55:01] "POST /api/v1/login HTTP/1.1" 401 56
192.168.1.1 - - [10/Oct/2024:13:55:02] "GET /api/v1/orders HTTP/1.1" 200 8901
192.168.1.3 - - [10/Oct/2024:13:55:03] "GET /api/v1/users/1 HTTP/1.1" 500 23
192.168.1.3 - - [10/Oct/2024:13:55:04] "GET /api/v1/users/2 HTTP/1.1" 500 23
EOF

# 1. Hitung request per IP
awk '{print $1}' /tmp/access.log | sort | uniq -c

# 2. Tampilkan hanya 5xx
grep -E ' 5[0-9]{2} ' /tmp/access.log

# 3. Hitung status code
awk '{print $9}' /tmp/access.log | sort | uniq -c

# 4. Endpoint path paling sering
awk '{print $7}' /tmp/access.log | sort | uniq -c | sort -rn
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Cari teks | `grep` |
| Edit/sunting | `sed` |
| Olah kolom | `awk` |
| Olah JSON | `jq` |
| Pipeline + simpan | `tee` |
| Hitung | `wc`, `uniq -c` |

## Cek Pemahaman

1. Beda `sed 's/x/y/'` vs `sed 's/x/y/g'`?
2. Tulis one-liner: 5 IP terbesar pengakses error 500 dari access.log.
3. Bagaimana output JSON besar di-log dipotong + dicerna dengan `jq`?
