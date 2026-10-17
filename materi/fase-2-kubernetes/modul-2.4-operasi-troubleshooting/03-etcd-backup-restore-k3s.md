# 03 — Snapshot & Restore Embedded etcd k3s: Menjaga State Kluster & Prosedur Disaster Recovery

> **Ringkasan:** Di dalam arsitektur Kubernetes, *Node* dan *Pod* dirancang untuk bersifat fana (*ephemeral*) dan dapat dimusnahkan kapan saja. Namun, etcd adalah **jantung dan satu-satunya sumber kebenaran (*single source of truth*)** tempat seluruh state kluster disimpan. Dokumen ini membedah teori konsensus Raft, perhitungan *quorum*, pembuatan snapshot manual dan terjadwal pada embedded etcd k3s, verifikasi integritas data, hingga prosedur sakral pemulihan bencana (*Disaster Recovery*) menggunakan `--cluster-reset`.

---

## 1. Analogi Pemula: Buku Besar Notaris & Brankas Master

Bagi pemula, membayangkan hilangnya etcd bisa dianalogikan dengan kebakaran di kantor notaris tanah:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ANALOGI BUKU BESAR NOTARIS TANAH                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Bayangkan sebuah kota satelit (Kluster Kubernetes):                        │
│  - Rumah, ruko, dan pabrik adalah kontainer aplikasi (**Pod**).             │
│  - Lahan tanah tempat bangunan berdiri adalah server fisik (**Node**).      │
│  - Mandor dan kuli bangunan adalah Kubelet dan Scheduler.                   │
│                                                                             │
│  Di tengah kota berdiri Kantor Notaris & Pertanahan Pusat (**etcd**).       │
│  Di dalam kantor tersebut tersimpan satu-satunya **Buku Besar Sertifikat**  │
│  yang mencatat:                                                             │
│  - Siapa pemilik kavling nomor berapa (Service & Endpoints).                │
│  - Berapa jumlah rumah yang wajib dibangun di blok A (Replika Deployment).  │
│  - Password brankas rahasia masing-masing pemilik ruko (Secret & ConfigMap).│
│                                                                             │
│  JIKA TERJADI BENCANA:                                                      │
│  1. Jika sebuah rumah roboh (Pod crash) ──► Mandor tinggal melihat Buku     │
│     Besar, lalu membangun rumah yang identik dalam hitungan detik.          │
│  2. Jika satu kavling gempa (Node mati) ──► Mandor memindahkan pembangunan  │
│     ke kavling lain sesuai catatan Buku Besar.                              │
│  3. TAPI JIKA BUKU BESAR NOTARIS HANGUS TERBAKAR (etcd hancur tanpa backup):│
│     Seluruh mandor hilang ingatan! Tidak ada yang tahu siapa pemilik tanah, │
│     apa password sistem, dan bagaimana kota harus dioperasikan.             │
│     KOTA LUMPUH TOTAL DAN TIDAK BISA DIPULIHKAN!                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Apa yang Sebenarnya Disimpan di etcd vs Apa yang Tidak?

Sering kali timbul kesalahpahaman berbahaya: seorang engineer mengira dengan melakukan backup etcd, data database MySQL mereka sudah otomatis aman. Ini adalah asumsi yang salah!

```text
┌────────────────────────────────────────┬────────────────────────────────────────┐
│ DISIMPAN DI DALAM ETCD                 │ TIDAK DISIMPAN DI DALAM ETCD           │
│ (Dilindungi oleh Snapshot etcd)        │ (Membutuhkan Backup Terpisah!)         │
├────────────────────────────────────────┼────────────────────────────────────────┤
│ Seluruh manifes objek Kubernetes:      │ Data isi database aplikasi (tabel SQL  │
│ Deployments, StatefulSets, DaemonSets  │ MySQL, isi dokumen MongoDB di disk).   │
├────────────────────────────────────────┼────────────────────────────────────────┤
│ Seluruh konfigurasi aplikasi:          │ File data di dalam PersistentVolume    │
│ ConfigMaps dan Kubernetes Secrets      │ (PV) lokal atau NFS/Ceph storage.      │
├────────────────────────────────────────┼────────────────────────────────────────┤
│ Definisi jaringan & rute internal:     │ Image kontainer yang tersimpan di      │
│ Services, Endpoints, NetworkPolicies   │ Docker/containerd cache node.          │
├────────────────────────────────────────┼────────────────────────────────────────┤
│ Sumber daya kustom (CRD):              │ Konfigurasi OS Linux host server       │
│ IPAddressPool MetalLB, Ingress rules   │ (`/etc/hosts`, iptables host, sysctl). │
├────────────────────────────────────────┼────────────────────────────────────────┤
│ Hak akses keamanan kluster:            │ File token k3s server yang ada di      │
│ ServiceAccounts, Roles, RoleBindings   │ direktori filesystem `/var/lib/...`.   │
└────────────────────────────────────────┴────────────────────────────────────────┘
```

> [!CAUTION]
> **Hukum Utama Disaster Recovery SRE:**
> Backup etcd **TIDAK MENGGANTIKAN** backup aplikasi dan Persistent Volume! Snapshot etcd hanya menyelamatkan metadata dan blueprint arsitektur kluster Kubernetes Anda. Untuk database seperti PostgreSQL atau MySQL, Anda tetap wajib memiliki prosedur backup data harian menggunakan `pg_dump`, `mysqldump`, atau snapshot storage level (Velero).

---

## 3. Teori Konsensus Raft & Perhitungan Quorum

K3s yang dipasang dalam mode High Availability (HA) menggunakan **embedded etcd**, di mana setiap server control-plane menjalankan instans etcd internal yang saling berkoordinasi menggunakan algoritma **Raft Consensus**.

Agar kluster dapat menerima penambahan, modifikasi, atau penghapusan data, mayoritas server etcd harus sepakat (*mencapai Quorum*).

### Rumus Matematis Quorum:
$$Q = \left\lfloor \frac{n}{2} \right\rfloor + 1$$

Di mana $n$ adalah jumlah total server control-plane yang terdaftar di kluster.

```text
┌─────────────────┬──────────────────────┬───────────────────────────────┐
│ Jumlah Server   │ Nilai Quorum Minimal │ Toleransi Kegagalan Maksimal  │
│ Control-Plane   │ Yang Wajib Hidup ($Q$)│ Server Yang Boleh Mati ($F$)  │
├─────────────────┼──────────────────────┼───────────────────────────────┤
│ **1 Server**    │ $\lfloor 1/2 \rfloor + 1 = \mathbf{1}$ │ $\mathbf{0\text{ Server}}$ (Tidak ada HA)    │
├─────────────────┼──────────────────────┼───────────────────────────────┤
│ **2 Server**    │ $\lfloor 2/2 \rfloor + 1 = \mathbf{2}$ │ $\mathbf{0\text{ Server}}$ (Sangat Buruk!)   │
├─────────────────┼──────────────────────┼───────────────────────────────┤
│ **3 Server**    │ $\lfloor 3/2 \rfloor + 1 = \mathbf{2}$ │ $\mathbf{1\text{ Server}}$ (Standar Produksi)│
├─────────────────┼──────────────────────┼───────────────────────────────┤
│ **4 Server**    │ $\lfloor 4/2 \rfloor + 1 = \mathbf{3}$ │ $\mathbf{1\text{ Server}}$ (Mubazir!)        │
├─────────────────┼──────────────────────┼───────────────────────────────┤
│ **5 Server**    │ $\lfloor 5/2 \rfloor + 1 = \mathbf{3}$ │ $\mathbf{2\text{ Server}}$ (Enterprise Tier) │
└─────────────────┴──────────────────────┴───────────────────────────────┘
```

### Mengapa Jumlah Server etcd WAJIB GANJIL (1, 3, atau 5)?
Perhatikan perbandingan antara **3 Server** dan **4 Server**:
- Kluster 3 server membutuhkan 2 server hidup. Jika 1 server mati, kluster **tetap hidup**.
- Kluster 4 server membutuhkan 3 server hidup. Jika 2 server mati, kluster **langsung padam**. Toleransi kegagalannya tetap hanya 1 server!
- Menambahkan server keempat **sama sekali tidak menambah ketahanan kluster**, melainkan hanya menambah beban latensi sinkronisasi jaringan antar-server (*Raft sync latency*) dan meningkatkan risiko *Split-Brain*.

---

## 4. Siklus Hidup Snapshot pada Embedded etcd k3s

Salah satu keunggulan terbesar k3s dibandingkan Kubernetes vanila adalah k3s mengemas etcd secara internal dan menyediakan CLI bawaan `k3s etcd-snapshot` tanpa mewajibkan Anda menginstal tool terpisah seperti `etcdctl`.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ALIRAN PROSES SNAPSHOT EMBEDDED ETCD                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Operator mengeksekusi:                                                     │
│  $ sudo k3s etcd-snapshot save --name backup-manual-01                      │
│                      │                                                      │
│                      ▼                                                      │
│  K3s Process mengunci state transaksi bbolt DB secara internal              │
│  (Point-in-Time Consistent Snapshot)                                        │
│                      │                                                      │
│                      ▼                                                      │
│  File binary snapshot disimpan ke direktori filesystem server:              │
│  /var/lib/rancher/k3s/server/db/snapshots/backup-manual-01-<node>-<epoch>   │
│                      │                                                      │
│                      ▼                                                      │
│  SRE memverifikasi: Ukuran File > 0, Checksum SHA256, dan Backup Off-Site   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Hands-On: Membuat Snapshot Manual & Verifikasi Integritas

Masuk ke server control-plane pertama Anda (misal `k3s-cp1`) melalui terminal Mac:

```bash
# 1. Dari terminal Mac, masuk ke shell server k3s-cp1
$ orb shell k3s-cp1

# 2. Periksa apakah k3s sedang berjalan sehat dan memiliki ruang disk cukup
k3s-cp1:~$ sudo systemctl is-active k3s
active

k3s-cp1:~$ df -h /var/lib/rancher/k3s/server/db
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        16G  2.8G   13G  19% /

# 3. Eksekusi pembuatan snapshot manual dengan menyertakan timestamp UTC
k3s-cp1:~$ sudo k3s etcd-snapshot save --name "sre-prod-backup-$(date -u +%Y%m%dT%H%M%SZ)"
INFO[0000] Snapshot sre-prod-backup-20261017T073000Z-k3s-cp1-1760686200 saved: /var/lib/rancher/k3s/server/db/snapshots/sre-prod-backup-20261017T073000Z-k3s-cp1-1760686200

# 4. Verifikasi keberadaan file dan periksa ukurannya
k3s-cp1:~$ sudo ls -lah /var/lib/rancher/k3s/server/db/snapshots/
total 15M
drwx------ 2 root root 4.0K Oct 17 14:30 .
drwx------ 4 root root 4.0K Oct 16 10:15 ..
-rw------- 1 root root  15M Oct 17 14:30 sre-prod-backup-20261017T073000Z-k3s-cp1-1760686200

# 5. Hitung nilai hash SHA256 sebagai segel integritas (tamper-proof seal)
k3s-cp1:~$ sudo sha256sum /var/lib/rancher/k3s/server/db/snapshots/sre-prod-backup-20261017T073000Z-k3s-cp1-1760686200
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  /var/lib/rancher/k3s/server/db/snapshots/sre-prod-backup-20261017T073000Z-k3s-cp1-1760686200
```

> [!WARNING]
> **Kebersihan Rahasia (*Secrets Hygiene*) File Snapshot:**
> File snapshot etcd menyimpan **seluruh Kubernetes Secret dalam format mentah (decoded)**! Siapa pun yang memiliki akses baca ke file snapshot ini dapat membaca password database, sertifikat private key, dan token API perusahaan Anda.
> - Pastikan izin berkas selalu terkunci pada **`chmod 600`** (hanya `root` yang dapat membaca).
> - Jangan pernah mengunggah file snapshot ke repositori Git publik!

---

## 6. Snapshot Otomatis Terjadwal & Integrasi Remote S3

Menyimpan snapshot hanya di dalam harddisk server yang sama adalah resep bencana—jika mesin VM tersebut terhapus atau rusak, file backup Anda ikut musnah bersamanya.

K3s menyediakan fitur otomatisasi snapshot berkala yang dapat mengirimkan file backup langsung ke penyimpanan objek (*Object Storage*) S3-compatible (seperti MinIO atau AWS S3) secara native.

Tambahkan konfigurasi berikut ke file `/etc/rancher/k3s/config.yaml` pada seluruh node server:

```yaml
# Konfigurasi Snapshot Terjadwal
etcd-snapshot-schedule-cron: "0 */6 * * *"  # Jalankan setiap 6 jam sekali
etcd-snapshot-retention: 12                  # Simpan 12 snapshot terakhir (3 hari)
etcd-snapshot-compress: true                 # Kompresi gzip untuk menghemat kapasitas disk

# (Opsional) Integrasi Off-Site S3 / MinIO
# etcd-s3: true
# etcd-s3-endpoint: "minio.internal:9000"
# etcd-s3-bucket: "k3s-etcd-backups"
# etcd-s3-access-key: "<MINIO_ACCESS_KEY>"
# etcd-s3-secret-key: "<MINIO_SECRET_KEY>"
# etcd-s3-insecure: true
```

---

## 7. Prosedur Sakral: Restore Snapshot (Disaster Recovery)

> [!CAUTION]
> **Peringatan Tingkat Bahaya Maksimum:**
> Prosedur restore etcd adalah operasi bedah destruktif yang memundurkan waktu kluster (*time travel*). Seluruh perubahan objek Kubernetes yang dibuat setelah waktu snapshot akan **HILANG PERMANEN**. Hanya jalankan prosedur ini pada saat kluster mengalami kegagalan total atau pada kluster lab yang telah disetujui!

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR RESTORE ETCD PADA KLUSTER HA 3-NODE                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. HENTIKAN SERVICE K3S DI SELURUH NODE SERVER                             │
│     (k3s-cp1, k3s-cp2, k3s-cp3 ──► systemctl stop k3s)                      │
│                                                                             │
│  2. EKSEKUSI CLUSTER-RESET DI SERVER INISIATOR (k3s-cp1)                    │
│     $ k3s server --cluster-reset --cluster-reset-restore-path=<snapshot>    │
│     (Database lama di-reset, snapshot diinjeksi, k3s-cp1 menjadi leader 1)  │
│                                                                             │
│  3. NYALAKAN KEMBALI K3S DI SERVER 1                                        │
│     (k3s-cp1 ──► systemctl start k3s)                                       │
│                                                                             │
│  4. BERSIHKAN DATABASE LAMA DI SERVER 2 DAN SERVER 3                        │
│     (k3s-cp2 & k3s-cp3 ──► rm -rf /var/lib/rancher/k3s/server/db/)          │
│                                                                             │
│  5. NYALAKAN K3S DI SERVER 2 DAN 3 AGAR BERGABUNG ULANG (RE-JOIN)           │
│     (k3s-cp2 & k3s-cp3 ──► systemctl start k3s)                             │
│     (Server 2 & 3 otomatis menyinkronkan data baru dari Server 1)           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Langkah Eksekusi Operasional Restore:

#### Langkah 1: Matikan Service k3s di Seluruh Server
```bash
# Jalankan di k3s-cp1, k3s-cp2, dan k3s-cp3:
sudo systemctl stop k3s
```

#### Langkah 2: Jalankan Cluster-Reset di Server Utama (`k3s-cp1`)
```bash
# Jalankan HANYA di k3s-cp1:
sudo k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/<nama-file-snapshot>
```
Output terminal akan menampilkan:
`Managed etcd cluster membership has been reset, restart without --cluster-reset flag now.`

#### Langkah 3: Nyalakan Kembali Server Utama
```bash
sudo systemctl start k3s
sudo kubectl get nodes -o wide
```

#### Langkah 4: Sinkronkan Server Peer (`k3s-cp2` dan `k3s-cp3`)
Di kedua server control-plane lainnya, direktori database lama yang sudah tidak sinkron harus dihapus sebelum service dinyalakan kembali:
```bash
# Jalankan di k3s-cp2 dan k3s-cp3:
sudo rm -rf /var/lib/rancher/k3s/server/db/
sudo systemctl start k3s
```

---

## 8. Mengukur RPO dan RTO dalam Praktik SRE

Dua indikator kunci dalam merancang strategi pemulihan bencana kluster adalah **RPO** dan **RTO**:

```text
  Kejadian Transaksi Terakhir        Bencana Terjadi        Kluster Kembali Online
            │                              │                           │
            ▼                              ▼                           ▼
   ─────────┼──────────────────────────────┼───────────────────────────┼──────────► Waktu
            │ ◄─────── RPO ──────────────► │ ◄──────── RTO ──────────► │
            │      Data yang Hilang        │    Waktu Downtime Layanan │
```

1. **RPO (Recovery Point Objective):** Batas toleransi maksimal data/state yang boleh hilang saat bencana terjadi.
   * *Contoh:* Jika Anda mengonfigurasi snapshot etcd setiap **6 jam sekali**, maka secara teoritis nilai RPO Anda adalah **6 jam**. Seluruh Deployment atau konfigurasi baru yang dibuat 1 jam lalu akan hilang jika kluster direstore dari snapshot 6 jam lalu.
2. **RTO (Recovery Time Objective):** Batas durasi waktu yang dibutuhkan sejak insiden terdeteksi hingga kluster dan aplikasi kembali melayani traffic secara normal.
   * *Contoh:* Jika tim Anda membutuhkan waktu 10 menit untuk mengambil file backup, 5 menit untuk cluster-reset, dan 5 menit untuk verifikasi, maka RTO aktual Anda adalah **20 menit**.

---

## 9. Matriks Pemecahan Masalah (Troubleshooting) Snapshot

| Gejala Masalah | Penyebab Akar (*Root Cause*) | Tindakan Perbaikan SRE |
|---|---|---|
| Perintah snapshot menghasilkan error `database locked` | Disk IOPS terlalu lambat atau ada proses snapshot lain yang sedang berjalan bersamaan. | Periksa beban IO disk via `iostat` dan pastikan tidak ada proses etcd backup ganda. |
| Ukuran file snapshot bernilai 0 byte atau hanya beberapa KB | Partisi harddisk penuh (*disk full 100%*), atau proses terputus di tengah jalan. | Kosongkan ruang disk pada `/var/lib/rancher/k3s` dan buat ulang snapshot baru. |
| Setelah restore, API server menolak koneksi (`connection refused`) | Token kluster atau sertifikat enkripsi etcd tidak cocok dengan snapshot yang direstore. | Pastikan file `/var/lib/rancher/k3s/server/token` identik dengan token saat snapshot dibuat. |
| Node worker tidak bisa join setelah cluster restore | URL API Server atau CA certificate berubah. | Restart service `k3s-agent` di worker node untuk memperbarui handshake sertifikat. |

---

## 10. Ringkasan & Cheatsheet Backup/Restore etcd

```bash
# 1. Menampilkan daftar seluruh snapshot etcd yang tersimpan
sudo k3s etcd-snapshot list

# 2. Membuat snapshot manual seketika
sudo k3s etcd-snapshot save --name "backup-manual-$(date +%s)"

# 3. Menghapus snapshot lama yang tidak terpakai
sudo k3s etcd-snapshot delete <nama-snapshot>

# 4. Melakukan reset dan restore kluster (Hanya di server inisiator saat k3s stop!)
sudo k3s server --cluster-reset --cluster-reset-restore-path=<path-ke-snapshot>

# 5. Memeriksa log proses etcd di systemd
sudo journalctl -u k3s -f | grep -i etcd
```

---

## 11. Cek Pemahaman Mandiri

1. Mengapa sebuah kluster Kubernetes yang memiliki 3 node control-plane dianggap jauh lebih aman dan efisien dibandingkan dengan kluster yang memiliki 4 node control-plane? Jelaskan menggunakan rumus Quorum!
2. Jika sebuah database MySQL berjalan di dalam kluster dan menyimpan data transaksi di sebuah Persistent Volume, apakah melakukan restore snapshot etcd otomatis mengembalikan isi tabel database yang terhapus? Jelaskan alasannya!
3. Mengapa berkas binary snapshot etcd wajib diberikan hak akses terbatas (`chmod 600`) dan tidak boleh disimpan di sembarang tempat?
4. Apa yang dimaksud dengan *RPO (Recovery Point Objective)*, dan bagaimana cara seorang SRE memperkecil nilai RPO pada kluster Kubernetes?
5. Jelaskan mengapa ketika melakukan pemulihan bencana (*Disaster Recovery*) pada kluster k3s HA 3-node, kita harus menghapus direktori `/var/lib/rancher/k3s/server/db/` pada Server 2 dan Server 3 sebelum menyalakan kembali servicenya!

---

*Langkah Selanjutnya:* Masuk ke [04 — Rolling Maintenance & Upgrade k3s](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.4-operasi-troubleshooting/04-upgrade-k3s-rolling.md) untuk mempelajari bagaimana melakukan pembaruan versi kluster Kubernetes secara bertahap tanpa menimbulkan gangguan layanan (*Zero Downtime*).
