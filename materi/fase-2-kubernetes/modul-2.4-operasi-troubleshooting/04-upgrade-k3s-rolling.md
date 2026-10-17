# 04 — Rolling Maintenance & Upgrade k3s: Mengelola Siklus Hidup Kluster Tanpa Downtime

> **Ringkasan:** Di dunia Site Reliability Engineering (SRE), pemeliharaan dan pembaruan versi kluster (*cluster upgrade*) adalah uji kedewasaan arsitektur yang sesungguhnya. Target operasionalnya bukan sekadar "sukses memasang binary baru", melainkan menjamin ketersediaan layanan (*Zero-Downtime Service Availability*), menjaga integritas konsensus *Quorum*, serta memiliki kondisi berhenti (*Stop Condition*) dan rencana mundur (*Rollback Plan*) yang jelas. Dokumen ini mengupas tuntas teknik *cordon*, *drain*, konfigurasi *PodDisruptionBudget* (PDB), aturan *Version Skew Policy*, hingga eksekusi *Rolling Upgrade* satu per satu pada kluster k3s High Availability.

---

## 1. Analogi Pemula: Pit Stop Tim Balap F1 Saat Mobil Melaju

Bagi orang awam, melakukan upgrade pada server produksi yang sedang sibuk melayani ribuan pengguna terdengar mengerikan. Mari kita gunakan analogi tim balap mobil Formula 1:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PIT STOP & ROLLING MAINTENANCE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PENDEKATAN AMATIR: MATIKAN MOBIL DI TENGAH SIRKUIT (DOWNTIME TOTAL) ]    │
│  Sopir mematikan mesin di tengah lintasan, seluruh penonton kecewa, balapan │
│  terhenti, lalu mekanik mengganti keempat ban sekaligus selama 30 menit.    │
│  Pengguna aplikasi mengalami "Maintenance Mode 503" berjam-jam.             │
│                                                                             │
│  [ PENDEKATAN SRE PROFESIONAL: PIT STOP DINAMIS BERGANTIAN (ROLLING) ]      │
│  Mobil balap memiliki sistem multi-roda dan mesin cadangan (HA Multi-Node): │
│  1. Manajer tim memasang rambu larangan parkir di ban nomor 1               │
│     (**kubectl cordon node-1**).                                            │
│  2. Penumpang di atas ban nomor 1 dipindahkan secara tertib dan perlahan ke │
│     kursi roda nomor 2 yang sudah siap melayani (**kubectl drain**).        │
│  3. Sabuk pengaman memastikan bahwa minimal 2 roda wajib menapak di aspal   │
│     setiap saat (**PodDisruptionBudget / PDB**).                             │
│  4. Mekanik mengganti ban nomor 1 dengan versi terbaru.                     │
│  5. Setelah roda baru berputar kencang dan stabil, rambu dilepas            │
│     (**kubectl uncordon**).                                                 │
│  6. Mekanik beralih ke roda nomor 2, lalu roda nomor 3.                     │
│                                                                             │
│  HASILNYA: Mobil balap tidak pernah berhenti melaju, dan penonton (klien)   │
│  tidak pernah merasakan adanya gangguan layanan!                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Tiga Perintah Sakral Node Maintenance

Sebelum menyentuh proses instalasi binary baru pada sistem operasi, Anda wajib menguasai 3 tahapan pengosongan node di Kubernetes:

```text
 ┌──────────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
 │  1. KUBECTL CORDON   │ ───► │   2. KUBECTL DRAIN   │ ───► │  3. KUBECTL UNCORDON │
 │ (Gembok Penjadwalan) │      │  (Evakuasi Pods)     │      │ (Buka Kembali Akses) │
 └──────────────────────┘      └──────────────────────┘      └──────────────────────┘
```

### 1. `kubectl cordon <nama-node>`
* **Fungsi:** Mengunci node agar berstatus **`SchedulingDisabled`**.
* **Perilaku:** Pod yang saat ini sedang berjalan di node tersebut **tetap hidup dan melayani traffic**. Namun, *kube-scheduler* dilarang keras menempatkan Pod baru apa pun di node ini.
* **Tujuan SRE:** Mencegah Pod baru mampir ke node yang sebentar lagi akan dimatikan.

```bash
$ kubectl cordon k3s-w1
node/k3s-w1 cordoned

$ kubectl get nodes
NAME      STATUS                     ROLES                       AGE   VERSION
k3s-cp1   Ready                      control-plane,etcd,master   2d    v1.31.5+k3s1
k3s-cp2   Ready                      control-plane,etcd,master   2d    v1.31.5+k3s1
k3s-w1    Ready,SchedulingDisabled   worker                      2d    v1.31.5+k3s1
```

### 2. `kubectl drain <nama-node>`
* **Fungsi:** Mengusir (*evict*) seluruh Pod yang ada di node tersebut secara terhormat (*graceful*) menggunakan **Kubernetes Eviction API**.
* **Perilaku:** Kontainer menerima sinyal `SIGTERM`, menyelesaikan request yang sedang berjalan dalam batas `terminationGracePeriodSeconds` (default 30s), lalu Deployment Controller membuat Pod pengganti di node lain yang masih aktif.
* **Parameter Wajib yang Harus Dipahami:**

```bash
$ kubectl drain k3s-w1 \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --timeout=10m
```

| Parameter Flag | Penjelasan & Rationale SRE |
|---|---|
| `--ignore-daemonsets` | **Wajib!** Pod DaemonSet (seperti flannel CNI, kube-proxy, speaker MetalLB) terikat permanen pada setiap node dan tidak bisa dipindahkan. Tanpa flag ini, perintah drain akan langsung gagal. |
| `--delete-emptydir-data` | Menyetujui penghapusan data sementara (*ephemeral storage*) pada Pod yang menggunakan volume `emptyDir`. Data ini akan hilang saat Pod dimatikan. |
| `--timeout=10m` | Membatasi waktu maksimal evakuasi. Mencegah proses drain menggantung (*hang*) selamanya jika ada Pod yang menolak mati. |

> [!CAUTION]
> **Bahaya Parameter `--force`:**
> Jangan pernah menambahkan flag `--force` secara serampangan saat menjalankan `kubectl drain`. Flag `--force` akan menghapus Pod yang tidak dikelola oleh controller (seperti Pod standalone tanpa Deployment/ReplicaSet), menyebabkan data dan workload tersebut musnah seketika tanpa dibuatkan penggantinya!

### 3. `kubectl uncordon <nama-node>`
* **Fungsi:** Membuka kembali gembok penjadwalan pada node yang telah selesai di-upgrade dan dinyatakan sehat.
* **Perilaku:** Node kembali berstatus `Ready`, dan *kube-scheduler* kembali diizinkan menempatkan Pod baru di node tersebut.

```bash
$ kubectl uncordon k3s-w1
node/k3s-w1 uncordoned
```

---

## 3. PodDisruptionBudget (PDB): Sabuk Pengaman Workload

Sering kali terjadi bencana konyol: seorang operator menjalankan `kubectl drain` pada Node 1, tanpa menyadari bahwa **seluruh 3 replika aplikasi web kebetulan sedang berkumpul di Node 1**. Akibatnya, ketiga Pod dimatikan bersamaan dan aplikasi mengalami padam total (*downtime*)!

Untuk mencegah hal ini, SRE memasang **PodDisruptionBudget (PDB)**:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api-pdb
  namespace: sre-lab
spec:
  minAvailable: 1          # Setidaknya WAJIB ada minimal 1 Pod yang Ready setiap saat!
  # atau bisa juga:
  # maxUnavailable: 1      # Maksimal hanya 1 Pod yang boleh mati dalam satu waktu!
  selector:
    matchLabels:
      app: payment-api
```

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 BAGAIMANA DRAIN BERINTERAKSI DENGAN PDB                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Operator mengeksekusi: kubectl drain k3s-cp1                               │
│                         │                                                   │
│                         ▼                                                   │
│  Eviction API Kubelet membaca PDB: "minAvailable: 1"                        │
│                         │                                                   │
│                         ▼                                                   │
│  Kubelet mengecek: "Berapa Pod payment-api yang sedang Ready di node lain?" │
│  ├── Jika Pod pengganti di k3s-cp2 SUDAH READY:                             │
│  │   ──► Kubelet mengizinkan Pod di k3s-cp1 dimatikan (Drain SUKSES).       │
│  │                                                                          │
│  └── Jika Pod di k3s-cp2 MASIH STARTING / BELUM READY:                      │
│      ──► Kubelet MENAHAN DRAIN (Menolak membunuh Pod sampai ada yang Ready).│
│          Aplikasi dijamin NOL DOWNTIME!                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Aturan Version Skew Policy Kubernetes

Kubernetes memiliki aturan resmi batas perbedaan versi (*Version Skew Policy*) yang wajib dipatuhi:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ATURAN KUBERNETES VERSION SKEW POLICY                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. CONTROL PLANE DULUAN, BARU WORKER:                                      │
│     Kube-apiserver harus selalu sama atau lebih baru dari Kubelet worker.   │
│     Worker Kubelet TIDAK BOLEH LEBIH BARU daripada Control Plane!           │
│                                                                             │
│  2. BATAS PERBEDAAN VERSI MINOR (N s.d. N-2):                               │
│     Kubelet worker maksimal boleh tertinggal hingga 2 versi minor di        │
│     belakang Kube-apiserver (misal: Server v1.31, Worker boleh v1.29).      │
│                                                                             │
│  3. JANGAN LOMPAT VERSI MINOR SEKALIGUS:                                    │
│     Jika ingin upgrade dari v1.29 ke v1.31, Anda WAJIB meng-upgrade ke       │
│     v1.30 terlebih dahulu, baru kemudian ke v1.31 (v1.29 ──► v1.30 ──► v1.31)│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Urutan Emas (The Golden Sequence) Rolling Upgrade HA k3s

Berikut adalah panduan operasional langkah demi langkah untuk melakukan rolling upgrade pada kluster k3s High Availability (3 Control-Plane Server + 2 Worker Node):

```text
  TAHAP 0: Preflight, Snapshot etcd, & Baseline Health Check
     │
     ▼
  TAHAP 1: Upgrade Server Control-Plane 1 (k3s-cp1) ──► Validasi Quorum
     │
     ▼
  TAHAP 2: Upgrade Server Control-Plane 2 (k3s-cp2) ──► Validasi Quorum
     │
     ▼
  TAHAP 3: Upgrade Server Control-Plane 3 (k3s-cp3) ──► Seluruh Server Selesai!
     │
     ▼
  TAHAP 4: Upgrade Worker Node 1 (k3s-w1) ──► Validasi Workload Pods
     │
     ▼
  TAHAP 5: Upgrade Worker Node 2 (k3s-w2) ──► Validasi Akhir & Uncordon All
```

### Prosedur Eksekusi Per Node (Dijalankan Satu Node pada Satu Waktu!)

Misalkan kita ingin melakukan upgrade kluster dari versi `v1.31.5+k3s1` ke target versi stabil baru yang telah di-review: `v1.31.6+k3s1`.

#### 1. Persiapan & Backup (Preflight)
Sebelum menyentuh satu mesin pun, buat snapshot etcd darurat sesuai [Modul 2.4 Topik 03](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.4-operasi-troubleshooting/03-etcd-backup-restore-k3s.md):
```bash
$ ssh k3s-cp1 'sudo k3s etcd-snapshot save --name pre-upgrade-baseline'
```

#### 2. Siklus Upgrade untuk Node Control-Plane (Contoh: `k3s-cp1`)

```bash
# A. Kunci node dan evakuasi workload dari terminal Mac
$ kubectl cordon k3s-cp1
$ kubectl drain k3s-cp1 --ignore-daemonsets --delete-emptydir-data --timeout=10m

# B. Masuk ke server k3s-cp1 dan jalankan script upgrade resmi terpin
$ ssh k3s-cp1
k3s-cp1:~$ export INSTALL_K3S_VERSION="v1.31.6+k3s1"
k3s-cp1:~$ curl -sfL https://get.k3s.io | sh -s - server

# C. Tunggu proses restart systemd selesai dan periksa status service
k3s-cp1:~$ sudo systemctl is-active k3s
active
k3s-cp1:~$ k3s --version
k3s version v1.31.6+k3s1 (commit ...)

# D. Dari terminal Mac, validasi kesehatan node dan API Server
$ kubectl get node k3s-cp1
NAME      STATUS                     ROLES                       AGE   VERSION
k3s-cp1   Ready,SchedulingDisabled   control-plane,etcd,master   2d    v1.31.6+k3s1

$ kubectl get --raw='/readyz?verbose' | grep -i etcd
[+]etcd ok

# E. Buka kembali gembok penjadwalan
$ kubectl uncordon k3s-cp1
node/k3s-cp1 uncordoned
```

> [!IMPORTANT]
> **Jeda Observasi Quorum (Quorum Cool-Down):**
> Setelah `k3s-cp1` selesai di-upgrade, **JANGAN LANGSUNG MELOMPAT KE SERVER 2!** Tunggu selama 2 hingga 3 menit. Periksa log k3s (`journalctl -u k3s -f`) dan pastikan ketiga server control-plane telah kembali sinkron dalam satu konsensus Raft etcd yang sehat. Baru setelah itu, ulangi langkah di atas untuk `k3s-cp2`, lalu `k3s-cp3`.

---

#### 3. Siklus Upgrade untuk Worker Node (Contoh: `k3s-w1`)
Setelah seluruh server control-plane sukses di-upgrade ke versi target, barulah giliran node worker yang diproses:

```bash
# A. Cordon dan Drain worker dari terminal Mac
$ kubectl cordon k3s-w1
$ kubectl drain k3s-w1 --ignore-daemonsets --delete-emptydir-data --timeout=10m

# B. Masuk ke server k3s-w1 dan jalankan upgrade k3s agent
$ ssh k3s-w1
k3s-w1:~$ export INSTALL_K3S_VERSION="v1.31.6+k3s1"
k3s-w1:~$ curl -sfL https://get.k3s.io | sh -s - agent

# C. Validasi service agent
k3s-w1:~$ sudo systemctl is-active k3s-agent
active

# D. Buka kembali penjadwalan dari terminal Mac
$ kubectl uncordon k3s-w1
node/k3s-w1 uncordoned
```

---

## 6. Validasi Ketersediaan Eksternal (Continuity Smoke Test)

Sebagai SRE, Anda harus membuktikan bahwa proses rolling upgrade tidak menimbulkan gangguan pada pengguna luar. Buka satu jendela terminal terpisah di Mac Anda dan jalankan loop pengujian HTTP ke Virtual IP MetalLB selama proses maintenance berlangsung:

```bash
# Menguji traffic ke VIP MetalLB secara terus-menerus setiap 2 detik
$ VIP="192.168.97.200"
$ while true; do
    CODE=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 2 http://${VIP}/)
    TIME=$(date +"%T")
    if [ "$CODE" == "200" ]; then
      echo "[$TIME] OK: Status HTTP $CODE (Layanan Normal)"
    else
      echo "[$TIME] WARNING: Status HTTP $CODE (Terjadi Gangguan!)"
    fi
    sleep 2
  done
```

Output terminal saat rolling upgrade berjalan sukses:
```text
[14:15:02] OK: Status HTTP 200 (Layanan Normal)
[14:15:04] OK: Status HTTP 200 (Layanan Normal)
[14:15:06] OK: Status HTTP 200 (Layanan Normal)  <-- k3s-cp1 sedang di-drain
[14:15:08] OK: Status HTTP 200 (Layanan Normal)  <-- k3s-cp1 sedang restart
[14:15:10] OK: Status HTTP 200 (Layanan Normal)  <-- traffic dilayani k3s-w1
[14:15:12] OK: Status HTTP 200 (Layanan Normal)
```

Nol kegagalan koneksi (*Zero Failed Requests*)! Inilah bukti nyata keunggulan arsitektur High Availability yang dipadukan dengan disiplin SRE.

---

## 7. Kondisi Berhenti (Stop Condition) & Prosedur Abort Darurat

Kapan seorang SRE wajib membunyikan alarm darurat dan **MENGHENTIKAN PROSES UPGRADE SEKETIKA**?

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KRITERIA STOP CONDITION (KONDISI BERHENTI)               │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. API Server kehilangan quorum etcd atau menolak koneksi.                 │
│ 2. Node yang baru di-upgrade menolak masuk status Ready (> 5 menit).        │
│ 3. Perintah drain macet karena pelanggaran PodDisruptionBudget (PDB).      │
│ 4. Error rate aplikasi pada continuous smoke test melonjak di atas batas 1%.│
│ 5. Terjadi split-brain atau desinkronisasi data etcd antar server.          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Rencana Tindakan Abort Darurat:
1. **Jangan Menyentuh Node Lain:** Segera batalkan rencana upgrade ke node berikutnya. Pertahankan node lain yang masih sehat agar tetap berjalan pada versi stabil sebelumnya.
2. **Karantina Node yang Bermasalah:** Biarkan node yang gagal dalam status `cordon` agar tidak menerima beban traffic dari pengguna.
3. **Investigasi Journal Systemd:** Periksa pesan error kernel atau k3s pada node yang bermasalah:
   ```bash
   sudo journalctl -u k3s -n 200 --no-pager
   ```
4. **Keputusan Rollback:** Jika binary baru mengalami *kernel incompatibility*, downgrade binary ke versi lama yang didukung. Jika state etcd terkorupsi, eksekusi prosedur Disaster Recovery snapshot etcd pada cluster maintenance window.

---

## 8. Matriks Pemecahan Masalah (Troubleshooting) Rolling Upgrade

| Gejala Masalah | Titik Investigasi | Penyebab Akar (*Root Cause*) | Solusi Pemulihan |
|---|---|---|---|
| Perintah `kubectl drain` macet berjam-jam | `kubectl describe pdb -A`<br>`kubectl get pods -o wide` | Aturan PDB terlalu ketat (misal `minAvailable: 2` tapi replika total hanya 2 dan pod kedua gagal start). | Naikkan jumlah replika Deployment di node lain sebelum melanjutkan drain, atau sesuaikan PDB. |
| Node berstatus `Ready,SchedulingDisabled` | `kubectl get nodes` | Operator lupa menjalankan perintah `uncordon` setelah proses upgrade selesai. | Jalankan `kubectl uncordon <nama-node>`. |
| Setelah upgrade, Pod aplikasi berstatus `CrashLoopBackOff` | `kubectl describe pod`<br>`kubectl logs` | Aplikasi menggunakan API Kubernetes yang sudah *deprecated* atau dihapus di versi Kubernetes baru. | Selalu baca *Deprecation Guide* rilis Kubernetes sebelum menaikkan versi kluster. |
| MetalLB VIP tidak bisa diakses saat node leader di-drain | `kubectl logs -n metallb-system -l component=speaker` | Mekanisme *Gratuitous ARP* lambat diperbarui oleh switch lokal atau Pod speaker crash. | Periksa apakah speaker DaemonSet berjalan di seluruh node dan dukung failover L2. |

---

## 9. Acceptance Criteria (Kriteria Keberhasilan Upgrade SRE)

Sebelum menandatangani dokumen persetujuan pemeliharaan (*Change Record Sign-Off*), pastikan seluruh kriteria berikut terpenuhi:

- [ ] Konteks kluster dan versi target telah diverifikasi secara tertulis.
- [ ] Snapshot etcd *pre-upgrade* berhasil dibuat dan diverifikasi nilai hash SHA256-nya.
- [ ] Seluruh PDB aktif dan tidak ada Pod berstatus unmanaged standalone.
- [ ] Seluruh server control-plane di-upgrade satu per satu dan berstatus `Ready`.
- [ ] Seluruh worker node di-upgrade satu per satu dan berstatus `Ready`.
- [ ] Tidak ada node yang tertinggal dalam status `SchedulingDisabled` (semua telah di-*uncordon*).
- [ ] Continuous smoke test `curl` membuktikan tidak ada lonjakan error rate selama upgrade.
- [ ] Endpoint `/readyz` mengembalikan status kesehatan kluster `[+]ok`.

---

## 10. Ringkasan & Cheatsheet Upgrade Rolling

```bash
# 1. Mengunci node dari penjadwalan baru
kubectl cordon <node>

# 2. Mengosongkan node secara graceful dan aman
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=10m

# 3. Membuka kembali node setelah upgrade selesai
kubectl uncordon <node>

# 4. Memantau transisi versi seluruh node secara real-time
kubectl get nodes -o wide -w

# 5. Memeriksa status pod disruption budget
kubectl get pdb -A

# 6. Memeriksa detail kesehatan internal kluster
kubectl get --raw='/readyz?verbose'
```

---

## 11. Cek Pemahaman Mandiri

1. Apa perbedaan mendasar antara perintah `kubectl cordon` dan `kubectl drain`? Kapan masing-masing perintah tersebut digunakan?
2. Mengapa flag `--ignore-daemonsets` wajib disertakan saat mengeksekusi `kubectl drain`? Apa yang akan terjadi jika flag tersebut tidak disertakan?
3. Jelaskan bagaimana *PodDisruptionBudget (PDB)* melindungi aplikasi bisnis agar tidak mengalami pemadaman total (*downtime*) saat proses `kubectl drain` dijalankan!
4. Menurut aturan *Kubernetes Version Skew Policy*, mengapa kita dilarang keras meng-upgrade worker node Kubelet ke versi yang lebih baru mendahului Kube-apiserver control-plane?
5. Mengapa pada kluster k3s High Availability dengan 3 server control-plane, kita wajib memberikan jeda waktu observasi (*quorum cool-down*) beberapa menit setelah Server 1 selesai di-upgrade sebelum melanjutkan ke Server 2?

---

*Selamat! Anda telah menyelesaikan seluruh modul teori Fondasi Kubernetes Fase 2.* Lanjutkan ke [LAB-01 — Chaos & Debug Workload](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.4-operasi-troubleshooting/lab/LAB-01-chaos-debug-workload.md) dan [LAB-02 — etcd Backup/Restore & Rolling Upgrade](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.4-operasi-troubleshooting/lab/LAB-02-etcd-backup-restore-rolling-upgrade.md) untuk membuktikan seluruh keahlian troubleshooting ini secara langsung di terminal!
