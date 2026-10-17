# 01 — Debug Workload: Dari Symptom Menuju Root Cause di Kubernetes

> **Ringkasan:** Di dunia Site Reliability Engineering (SRE), menghapus Pod yang bermasalah secara membabi buta bukanlah solusi pemecahan masalah—tindakan tersebut justru menghancurkan barang bukti forensik. Dokumen ini membedah alur investigasi sistematis 5 langkah (`get → describe/events → logs → logs --previous → exec`), mengupas tuntas 4 status kegagalan paling umum (*The Big Four Failures: CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled*), memahami bahaya salah konfigurasi *Health Probes*, serta menyusun laporan insiden berstandar industri.

---

## 1. Filosofi SRE: Siklus Investigasi Ilmiah vs Anti-Pattern "Hapus Pod"

Bagi seorang pemula atau operator amatir, naluri pertama ketika melihat sebuah Pod berstatus error adalah langsung mengeksekusi `kubectl delete pod <nama-pod>`. Meskipun Pod baru akan segera dibuat oleh Deployment Controller, tindakan ini adalah **anti-pattern fatal** dalam disiplin SRE karena:

1. **Menghapus Bukti Forensik:** Log crash di memori kontainer, status exit code terakhir, dan state internal langsung musnah.
2. **Tidak Menyelesaikan Akar Masalah (*Root Cause*):** Jika akar masalahnya adalah kesalahan konfigurasi database atau kehabisan memori, Pod baru yang dibuat akan kembali crash dalam hitungan detik.
3. **Memperburuk Pemulihan Sistem:** Menghapus Pod berulang kali memicu back-off delay Kubernetes yang semakin lama (hingga 5 menit jeda restart).

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       SIKLUS INVESTIGASI ILMIAH SRE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. SCOPE & CONTEXT   ──► Identifikasi lingkup: 1 Pod, 1 Node, atau Kluster│
│            │                                                                │
│            ▼                                                                │
│   2. GATHER EVIDENCE   ──► Kumpulkan bukti (status, events, logs, metrics)  │
│            │                                                                │
│            ▼                                                                │
│   3. FORM HYPOTHESIS   ──► Buat hipotesis terukur: "Mengapa sistem gagal?"  │
│            │                                                                │
│            ▼                                                                │
│   4. SMALLEST CHANGE   ──► Terapkan 1 perubahan terkecil yang bisa di-revert│
│            │                                                                │
│            ▼                                                                │
│   5. VALIDATE & RECORD ──► Buktikan perbaikan & catat di Blameless Postmortem│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Analogi Pemula: Dokter IGD Memeriksa Pasien Kritis
Bayangkan seorang dokter di Instalasi Gawat Darurat (IGD) menerima pasien yang pingsan:
- Dokter **tidak akan langsung melakukan operasi bedah besar** tanpa pemeriksaan awal.
- Dokter memeriksa tanda vital dasar: tensi dan nadi (**Langkah 1: `kubectl get`**).
- Dokter memeriksa riwayat penyakit dan catatan perawat (**Langkah 2: `kubectl describe & events`**).
- Dokter mendengarkan apa yang dirasakan pasien sesaat sebelum pingsan (**Langkah 3 & 4: `kubectl logs` dan `--previous`**).
- Jika ada indikasi keracunan, dokter mengambil sampel darah untuk tes laboratorium (**Langkah 5: `kubectl exec`**).
- Barulah dokter memberikan obat yang tepat sasaran!

---

## 2. Alur Investigasi 5-Langkah (The 5-Step Diagnostic Funnel)

Dalam mendiagnosa workload Kubernetes, kita menggunakan pola piramida terbalik: mulai dari observasi non-invasif dari luar hingga inspeksi mendalam ke dalam kontainer.

```text
 ┌──────────────────────────────────────────────────────────────┐
 │ 1. kubectl get pod           (Status makro, fase, restarts)  │
 └──────────────┬───────────────────────────────────────────────┘
                │
 ┌──────────────▼───────────────────────────────────────────────┐
 │ 2. kubectl describe pod      (Catatan Kubelet, Events, State)│
 └──────────────┬───────────────────────────────────────────────┘
                │
 ┌──────────────▼───────────────────────────────────────────────┐
 │ 3. kubectl logs              (Output stdout/stderr saat ini) │
 └──────────────┬───────────────────────────────────────────────┘
                │
 ┌──────────────▼───────────────────────────────────────────────┐
 │ 4. kubectl logs --previous   (Log kontainer sebelum crash)   │
 └──────────────┬───────────────────────────────────────────────┘
                │
 ┌──────────────▼───────────────────────────────────────────────┐
 │ 5. kubectl exec / debug      (Inspeksi lingkungan runtime)   │
 └──────────────────────────────────────────────────────────────┘
```

### Langkah 1: `kubectl get` (Menilai Status Makro)
Perintah ini memberikan gambaran status umum secara instan:

```bash
$ kubectl get pods -n sre-lab -o wide
NAME                        READY   STATUS             RESTARTS      AGE   IP           NODE      NOMINATED NODE   READINESS GATES
payment-api-6789b-4k2l1     0/1     CrashLoopBackOff   5 (2m ago)    8m    10.42.1.25   k3s-w1    <none>           <none>
inventory-svc-548c-98zxc    0/1     ImagePullBackOff   0             3m    10.42.2.14   k3s-cp2   <none>           <none>
report-worker-341a-pl09a    0/1     Pending            0             12m   <none>       <none>    <none>           <none>
analytics-db-88ff-m99x2     0/1     OOMKilled          3 (45s ago)   5m    10.42.1.26   k3s-w1    <none>           <none>
```

Informasi krusial yang harus Anda baca dari output di atas:
- `READY (0/1)`: Kontainer belum lolos readiness probe atau belum berjalan sama sekali.
- `STATUS`: Petunjuk awal kategori kegagalan.
- `RESTARTS`: Berapa kali kontainer mati dan dihidupkan ulang oleh kubelet. Jika angka restart tinggi dan terus bertambah, masalahnya terjadi saat runtime.
- `NODE (<none>)`: Pod belum pernah dijadwalkan ke node fisik mana pun (masalah scheduler).

### Langkah 2: `kubectl describe pod` & Events (Rekam Jejak Kubelet)
`kubectl describe` adalah sumber informasi paling kaya di Kubernetes. Selalu baca output dari bagian **Containers** dan bagian **Events** di baris terbawah:

```bash
$ kubectl describe pod payment-api-6789b-4k2l1 -n sre-lab
...
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Sat, 17 Oct 2026 14:02:10 +0700
      Finished:     Sat, 17 Oct 2026 14:02:12 +0700
...
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  8m                    default-scheduler  Successfully assigned sre-lab/payment-api-6789b-4k2l1 to k3s-w1
  Normal   Pulled     7m (x5 over 8m)       kubelet            Container image "registry.local/payment:v1.2" already present on machine
  Normal   Created    7m (x5 over 8m)       kubelet            Created container payment-api
  Normal   Started    7m (x5 over 8m)       kubelet            Started container payment-api
  Warning  BackOff    2m45s (x24 over 7m)   kubelet            Back-off restarting failed container
```

Perhatikan kolom `Last State`! Kubelet mencatat mengapa kontainer sebelumnya mati:
- `Exit Code: 1` menunjukkan aplikasi melempar *unhandled exception* atau *fatal error*.
- `Finished` hanya 2 detik setelah `Started` menunjukkan aplikasi mati sesaat setelah proses inisialisasi (*fail-fast at startup*).

### Langkah 3: `kubectl logs` (Mendengar Suara Aplikasi)
Jika Pod berisi lebih dari satu kontainer (*multi-container pod*), Anda **wajib** menyebutkan nama kontainer dengan flag `-c <nama-kontainer>`:

```bash
# Membaca log dengan timestamp asli
$ kubectl logs payment-api-6789b-4k2l1 -n sre-lab -c payment-api --tail=50 --timestamps
2026-10-17T07:02:10.112Z [INFO] Starting Payment Service v1.2...
2026-10-17T07:02:11.450Z [INFO] Connecting to database at db.internal:5432...
2026-10-17T07:02:12.890Z [FATAL] Database connection failed: dial tcp 10.43.0.55:5432: connect: connection refused
2026-10-17T07:02:12.891Z [FATAL] Exiting application with code 1
```

Akar masalah langsung terungkap: Aplikasi mati karena tidak bisa terhubung ke database PostgreSQL di `db.internal:5432`!

### Langkah 4: `kubectl logs --previous` (Kotak Hitam Pesawat Jatuh)
Sering kali pemula menjalankan `kubectl logs` dan melihat log kosong atau hanya satu baris `Starting server...`. Mengapa demikian?
Karena kontainer yang lama sudah mati, dan kontainer baru yang baru saja di-*restart* oleh Kubelet sedang menunggu inisialisasi.

Untuk membaca log dari kontainer **sebelum ia mati**, gunakan parameter sakti `--previous`:

```bash
$ kubectl logs payment-api-6789b-4k2l1 -n sre-lab --previous --tail=100
```

> [!IMPORTANT]
> **Nilai Emas Flag `--previous`:**
> Parameter `--previous` membaca buffer log dari kontainer generasi sebelumnya yang tersimpan di `/var/log/pods/` pada host node. Ini adalah padanan dari *Flight Data Recorder* (Black Box) pesawat terbang. Tanpa flag ini, Anda akan kehilangan pesan error terakhir yang memicu kematian kontainer.

### Langkah 5: `kubectl exec` / Debug (Inspeksi Runtime Aktif)
Jika kontainer sedang hidup atau berada dalam kondisi macet (*hang*), kita dapat masuk ke dalam namespace kontainer untuk memeriksa DNS, konektivitas, atau variabel lingkungan:

```bash
# 1. Periksa variabel lingkungan di dalam kontainer
$ kubectl exec -it payment-api-6789b-4k2l1 -n sre-lab -c payment-api -- env | grep -E 'DB_|PORT'
DB_HOST=db.internal
DB_PORT=5432
DB_USER=postgres

# 2. Uji resolusi DNS kluster dari dalam kontainer
$ kubectl exec -it payment-api-6789b-4k2l1 -n sre-lab -c payment-api -- nslookup db.internal
Server:    10.43.0.10
Address:   10.43.0.10#53

Name:      db.internal
Address:   10.43.0.55

# 3. Uji sambungan TCP langsung ke port database
$ kubectl exec -it payment-api-6789b-4k2l1 -n sre-lab -c payment-api -- nc -zv 10.43.0.55 5432
nc: connect to 10.43.0.55 port 5432 (tcp) failed: Connection refused
```

> [!CAUTION]
> **Aturan Keamanan SRE untuk `kubectl exec`:**
> `kubectl exec` hanya digunakan untuk **investigasi read-only** (membaca status, menguji konektivitas, melihat isi `/etc/resolv.conf`). **DILARANG KERAS** mengedit file kode aplikasi, menginstal paket dengan `apt-get`, atau mengubah konfigurasi di dalam kontainer yang sedang berjalan. Setiap perubahan wajib dilakukan secara deklaratif melalui Git dan manifest Kubernetes!

---

## 3. Bedah 4 Kegagalan Paling Sering (The Big Four Failures)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       EMPAT KEGAGALAN UTAMA WORKLOAD                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. CrashLoopBackOff ──► Kontainer start sukses lalu exit berulang-ulang     │
│ 2. ImagePullBackOff ──► Kubelet gagal mendownload image dari registry       │
│ 3. Pending          ──► Scheduler tidak menemukan node yang sanggup menampung│
│ 4. OOMKilled (137)  ──► Proses dibunuh oleh kernel Linux karena boros RAM   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. `CrashLoopBackOff`
* **Definisi:** Kubelet berhasil membuat kontainer dan menjalankan perintah entrypoint, tetapi proses aplikasi keluar (*exit*) dengan status error, atau gagal melewati *liveness probe*. Karena kebijakan restart kluster adalah `restartPolicy: Always` (default), Kubelet mencoba merestart kontainer. Untuk melindungi node dari CPU starvation akibat restart jutaan kali per detik, Kubelet memberlakukan jeda eksponensial (*back-off delay*: 10s, 20s, 40s, 80s, hingga maksimal 300s / 5 menit).
* **Daftar Exit Code Umum:**
  * **Exit Code 0:** Aplikasi selesai menjalankan tugasnya dengan sukses (*normal exit*). Ini umum terjadi jika Anda men-deploy batch script atau migration tool menggunakan objek `Deployment` (bukan `Job`). Karena Deployment mengharapkan proses berjalan selamanya sebagai daemon/server web, keluarnya proses kode 0 dianggap sebagai kegagalan dan di-restart berulang kali!
  * **Exit Code 1 / 2:** Kesalahan internal aplikasi (misal: *null pointer exception*, sintaks konfigurasi YAML salah, database connection timeout).
  * **Exit Code 137:** Proses dibunuh secara paksa oleh `SIGKILL` (sering kali karena `OOMKilled`).
  * **Exit Code 143:** Proses menerima sinyal `SIGTERM` (dimatikan secara terhormat oleh Kubelet saat rolling update atau evict).
* **Penyebab Akar Umum:**
  1. Variabel lingkungan (*environment variables*) wajib tidak ada atau salah ketik.
  2. Kunci Secret / ConfigMap tidak ditemukan di namespace.
  3. Layanan dependensi (database, Redis, auth server) belum siap menerima koneksi.
  4. Path atau port *liveness probe* salah sehingga Kubelet terus membunuh kontainer sehat.
  5. Permission ditolak saat membaca direktori data lokal (*filesystem permission denied*).

### 2. `ImagePullBackOff` / `ErrImagePull`
* **Definisi:** Kubelet tidak dapat mengunduh (*pull*) image kontainer dari registry remote. Kubelet akan mencoba lagi dengan jeda bertahap (*back-off*).
* **Matriks Investigasi:**

| Pesan Error pada Bagian Events | Penyebab Akar (*Root Cause*) | Langkah Solusi SRE |
|---|---|---|
| `manifest unknown: manifest unknown` | Nama image atau tag salah ketik, atau image belum di-push ke registry. | Periksa nama image di Dockerfile/CI dan pastikan tag ada di registry. |
| `unauthorized: authentication required` | Registry bersifat privat, dan Pod belum memiliki `imagePullSecrets` yang valid. | Buat Secret bertipe `kubernetes.io/dockerconfigjson` dan kaitkan ke ServiceAccount atau Pod. |
| `no matching manifest for linux/arm64` | Image hanya di-build untuk arsitektur x86_64, sementara node Anda berjalan di ARM64 (Apple Silicon). | Build ulang image menggunakan `docker buildx build --platform linux/amd64,linux/arm64`. |
| `x509: certificate signed by unknown authority` | Registry lokal/self-hosted menggunakan sertifikat SSL self-signed yang belum dipercaya Kubelet. | Tambahkan Root CA registry ke `/etc/ssl/certs` di setiap node atau gunakan flag insecure jika lab. |
| `i/o timeout` / `connection refused` | Masalah jaringan fisik, firewall memblokir port 443, atau server registry sedang tumbang. | Uji koneksi dari node host via `curl -v https://<registry-url>/v2/`. |

### 3. `Pending`
* **Definisi:** Pod telah disimpan di etcd oleh API Server, tetapi **kube-scheduler** belum menetapkan Pod tersebut ke node worker mana pun, ATAU Pod tertahan menunggu *PersistentVolumeClaim (PVC)* terikat (*Bound*).
* **Penyebab Akar Umum:**
  1. **Resource Starvation (`Insufficient cpu` / `Insufficient memory`):** Total `requests.cpu` atau `requests.memory` pada Pod lebih besar daripada sisa kapasitas alokasi (*allocatable*) yang tersedia di seluruh node kluster.
  2. **Taints and Tolerations:** Seluruh node memiliki *Taint* (misalnya `node-role.kubernetes.io/control-plane:NoSchedule`), dan Pod tidak memiliki *Toleration* yang cocok.
  3. **NodeSelector / NodeAffinity Mismatch:** Manifes Pod meminta label node tertentu (misalnya `disktype: ssd` atau `topology.kubernetes.io/zone: zone-a`), namun tidak ada satupun node yang memiliki label tersebut.
  4. **PVC Pending:** Pod membutuhkan volume penyimpanan persisten, tetapi StorageClass belum menyediakan PV atau volume backend sedang offline.

```bash
# Cara mendeteksi penyebab status Pending
$ kubectl describe pod report-worker-341a-pl09a -n sre-lab
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  12m   default-scheduler  0/3 nodes are available: 3 Insufficient memory. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.
```

### 4. `OOMKilled` (Exit Code 137)
* **Definisi:** *Out Of Memory Killed*. Aplikasi di dalam kontainer mencoba mengalokasikan RAM melebihi batas nilai `resources.limits.memory` yang telah didefinisikan pada manifes Pod.
* **Mekanisme di Balik Layar:** Kernel Linux mengawasi penggunaan memori kontainer melalui subsistem **cgroups (Control Groups)**. Ketika memori mencapai angka batas limit, kernel memanggil fungsi `oom-killer` untuk langsung mengeksekusi sinyal `SIGKILL (sinyal 9)` ke proses aplikasi.
* **Rumus Matematika Exit Code 137:**
  $$\text{Exit Code} = 128 + \text{Nomor Sinyal Unix} = 128 + 9 (\text{SIGKILL}) = 137$$
* **Pola Perbaikan yang Benar:**
  1. Jangan langsung menaikkan limit tanpa analisa! Periksa apakah ada kebocoran memori (*memory leak*) pada kode aplikasi.
  2. Untuk aplikasi Java/JVM, pastikan flag heap memory `-Xmx` disetel sekitar 70% dari limit kontainer agar menyisakan ruang untuk *metaspace* dan *thread stack*.
  3. Jika aplikasi memang secara wajar membutuhkan RAM lebih besar saat memproses beban puncak, naikkan `limits.memory` dan `requests.memory` secara terukur sesuai kapasitas node.

---

## 4. Probe Health Checks: Kawan atau Lawan?

Kubernetes menyediakan 3 jenis health probe untuk memantau kesehatan aplikasi secara otomatis. Namun jika salah dikonfigurasi, probe ini justru menjadi penyebab utama robohnya seluruh kluster (*Cascading Failure*):

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TIGA JENIS HEALTH PROBE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Startup Probe   ──► Menahan liveness/readiness sampai aplikasi siap start│
│ 2. Liveness Probe  ──► "Apakah proses macet?" ──► Jika GAGAL: Pod DIBUNUH   │
│ 3. Readiness Probe ──► "Apakah siap melayani?"──► Jika GAGAL: HAPUS DARI LB │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bahaya Fatal: Salah Merancang Liveness Probe
Banyak developer pemula membuat liveness probe dengan mengecek koneksi database:

```yaml
# CONTOH SALAH (ANTI-PATTERN FATAL!):
livenessProbe:
  httpGet:
    path: /healthz-check-db  # Endpoint ini melakukan query ke Database eksternal!
    port: 8080
```

**Skenario Bencana Kasus Nyata (*Cascading Disaster*):**
1. Server database utama sedang mengalami lonjakan beban sesaat dan merespons lambat selama 10 detik.
2. Endpoint `/healthz-check-db` pada 50 Pod aplikasi mengalami timeout.
3. Kubelet menganggap ke-50 Pod aplikasi tersebut sudah mati!
4. Kubelet **membunuh seluruh 50 Pod secara bersamaan** dan merestart semuanya.
5. Saat ke-50 Pod baru hidup secara bersamaan, mereka melakukan koneksi ulang serentak (*connection storm*) ke database.
6. Database langsung tumbang total (*crash*), dan seluruh sistem aplikasi mati berkepanjangan!

> [!TIP]
> **Aturan Emas Health Probe SRE:**
> - **Liveness Probe** HANYA boleh memeriksa kondisi internal proses kontainer itu sendiri (misal: apakah thread pool macet / deadlock?). JANGAN PERNAH mengecek dependensi eksternal!
> - **Readiness Probe** digunakan untuk memeriksa apakah aplikasi siap menerima traffic. Jika database down, biarkan readiness probe yang gagal sehingga Pod dikeluarkan dari Service Endpoints tanpa harus dibunuh.

---

## 5. Investigasi Jaringan: Pod Running Tapi Service Tidak Bisa Diakses

Sering kali terjadi kasus aneh: semua Pod berstatus `Running 1/1`, tetapi klien atau Ingress Controller selalu menerima error `HTTP 503 Service Unavailable` atau koneksi terputus.

Gunakan urutan investigasi berikut:

```bash
# 1. Periksa apakah Service memiliki Endpoints aktif
$ kubectl get endpoints web-service -n sre-lab
NAME          ENDPOINTS   AGE
web-service   <none>      15m

# 2. Mengapa Endpoints bernilai <none>? Bandingkan Label Selector!
$ kubectl get svc web-service -n sre-lab -o jsonpath='{.spec.selector}'
{"app":"web-app","tier":"frontend"}

# 3. Periksa label asli yang menempel pada Pod
$ kubectl get pods -n sre-lab --show-labels
NAME                       READY   STATUS    LABELS
web-app-85bf9485b5-4b2kd   1/1     Running   app=web-app,tier=ui  # PERHATIKAN: tier=ui BUKAN tier=frontend!
```

Akar masalah ditemukan: Terjadi ketidakcocokan (*label mismatch*) antara `selector` pada Service dengan `labels` pada Pod. Kube-proxy tidak dapat mendaftarkan IP Pod ke dalam Endpoints Service, sehingga seluruh traffic ke Service dibuang!

---

## 6. Template Laporan Insiden SRE (Blameless Postmortem)

Sebagai calon engineer SRE profesional, Anda wajib mendokumentasikan setiap insiden produksi ke dalam dokumen **Blameless Postmortem**. Dokumen ini berfokus pada kelemahan sistem dan otomasi, bukan menyalahkan individu:

```markdown
# [INCIDENT REPORT] — Kegagalan Pembayaran Payment API (CrashLoopBackOff)

## 1. Ringkasan Eksekutif
- **Tanggal/Waktu Insiden:** 17 Oktober 2026, 14:02 s.d. 14:25 WIB (Durasi: 23 menit)
- **Dampak Bisnis:** Pengguna tidak dapat menyelesaikan transaksi pembayaran. Sebanyak ~120 transaksi gagal dengan estimasi kerugian Rp 18.000.000,-.
- **Tingkat Keparahan (Severity):** P1 (Critical)

## 2. Garis Waktu Insiden (Timeline)
- `14:02 WIB` — CI/CD men-deploy rilis `v1.2` dari payment-api ke kluster produksi.
- `14:04 WIB` — Alertmanager membunyikan peringatan: `PaymentAPIDown` (Availability < 99%).
- `14:07 WIB` — SRE On-call menerima tiket dan memulai investigasi.
- `14:10 WIB` — Melalui `kubectl logs --previous`, ditemukan pesan fatal: `dial tcp: connection refused` ke port database 5432.
- `14:15 WIB` — Ditemukan bahwa ConfigMap rilis baru salah mencantumkan IP database staging alih-alih produksi.
- `14:18 WIB` — Dilakukan rollback ke revisi Deployment sebelumnya menggunakan `kubectl rollout undo`.
- `14:22 WIB` — Seluruh Pod kembali Ready (1/1) dan transaksi pembayaran berhasil diproses.
- `14:25 WIB` — Status insiden ditutup (Resolved).

## 3. Akar Masalah (Root Cause)
Pipeline deployment tidak memiliki tahap validasi integrasi variabel lingkungan (*schema validation*) antara environment staging dan produksi, sehingga nilai ConfigMap staging ter-commit ke branch release produksi.

## 4. Tindakan Pencegahan (Action Items)
1. [ ] Menambahkan validasi schema pada CI linting agar ConfigMap diverifikasi sebelum apply. (Owner: DevOps Team, Target: 24 Okt 2026)
2. [ ] Menambahkan automated smoke test pada pipeline staging sebelum promosi rilis. (Owner: QA Team, Target: 28 Okt 2026)
```

---

## 7. Ringkasan & Cheatsheet Investigasi Workload

```bash
# 1. Mendapatkan status Pod dengan ringkasan custom
kubectl get pods -n <ns> -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[*].restartCount

# 2. Melihat seluruh event urut dari waktu terbaru
kubectl get events -n <ns> --sort-by=.lastTimestamp

# 3. Membaca log kontainer spesifik sebelum crash
kubectl logs <pod> -n <ns> -c <container> --previous --tail=100 --timestamps

# 4. Memeriksa detail kenapa Pod gagal dijadwalkan
kubectl describe pod <pod> -n <ns> | sed -n '/Events:/,$p'

# 5. Memeriksa apakah Service memiliki sasaran Pod aktif
kubectl get endpoints <svc> -n <ns>

# 6. Membatalkan deployment yang bermasalah secara instan
kubectl rollout undo deployment/<deployment-name> -n <ns>
```

---

## 8. Cek Pemahaman Mandiri

1. Mengapa tindakan menghapus Pod yang sedang mengalami error (*kubectl delete pod*) dianggap sebagai tindakan buruk dalam kultur operasional SRE?
2. Jelaskan perbedaan mendasar antara informasi yang disajikan oleh `kubectl logs <pod>` biasa dibandingkan dengan `kubectl logs <pod> --previous`!
3. Jika sebuah Pod menampilkan status `Exit Code: 137`, apa yang sebenarnya terjadi pada proses aplikasi di tingkat kernel Linux?
4. Mengapa kita tidak boleh memasukkan kueri ke database eksternal di dalam endpoint *liveness probe* aplikasi web? Apa bahayanya jika database tersebut mengalami lonjakan beban sesaat?
5. Sebuah Pod berstatus `Pending` selama 30 menit. Ketika Anda menjalankan `kubectl top nodes`, penggunaan CPU dan RAM node hanya berada di angka 20%. Mengapa Pod tersebut tetap tidak dapat berjalan di node tersebut?

---

*Langkah Selanjutnya:* Masuk ke [02 — Resource, QoS & HPA](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.4-operasi-troubleshooting/02-resource-hpa.md) untuk mempelajari bagaimana mengatur alokasi CPU/Memory secara terukur dan mengonfigurasi autoscaling otomatis berdasarkan beban nyata.
