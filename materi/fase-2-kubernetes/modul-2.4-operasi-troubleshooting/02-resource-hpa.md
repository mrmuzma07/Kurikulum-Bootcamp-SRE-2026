# 02 — Manajemen Resource, Kualitas Layanan (QoS), dan Horizontal Pod Autoscaler (HPA)

> **Ringkasan:** Mengoperasikan kluster Kubernetes tanpa mendefinisikan *resource requests* dan *limits* ibarat mengelola gedung apartemen tanpa meteran listrik dan air—satu penyewa yang boros dapat memadamkan listrik seluruh gedung. Dokumen ini membahas secara mendalam mekanisme alokasi CPU dan Memory, hierarki kelas *Quality of Service* (QoS), rumus perhitungan kapasitas node (*Allocatable*), pipeline pengumpulan metrik, hingga konfigurasi autoscaling otomatis menggunakan Horizontal Pod Autoscaler (HPA v2) dengan kontrol perilaku (*behavior*) yang aman dari fenomena *flapping*.

---

## 1. Analogi Pemula: Reservasi Meja Restoran & Meteran Listrik Apartemen

Untuk memahami perbedaan konseptual antara **Requests** dan **Limits**, mari gunakan dua analogi kehidupan nyata:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DUA ANALOGI KUNCI: REQUESTS VS LIMITS                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ ANALOGI 1: RESERVASI MEJA RESTORAN (REQUESTS) ]                          │
│  - Anda menelepon restoran bintang lima: "Saya pesan meja untuk 4 orang     │
│    nanti malam" (Ini adalah Resource Request).                              │
│  - Restoran (kube-scheduler) memeriksa denah lantai (Node). Jika ada meja  │
│    kosong berkapasitas 4 kursi, restoran mengonfirmasi reservasi Anda.      │
│  - Yang penting bagi restoran adalah: kursi tersebut di-booking untuk Anda. │
│    Apakah saat makan nanti Anda hanya datang berdua atau berempat, meja itu │
│    tetap tidak boleh diberikan ke orang lain!                               │
│                                                                             │
│  [ ANALOGI 2: SEKERING & METERAN LISTRIK (LIMITS) ]                         │
│  - Kamar apartemen Anda dipasangi sekering pembatas daya 1300 Watt          │
│    (Ini adalah Resource Limit).                                             │
│  - Anda bebas menyalakan lampu, laptop, dan kipas angin.                    │
│  - Namun, begitu Anda menyalakan microwave dan pemanas air bersamaan hingga │
│    daya menyentuh 1301 Watt, sekering otomatis JEPRET mati seketika!        │
│    (Ini adalah proses OOMKilled di memori!).                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Membedah Requests vs Limits Secara Teknis

Di dalam manifes Pod atau Deployment, konfigurasi sumber daya didefinisikan di bawah blok kontainer:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi
```

Mari kita bedah perbedaan perilaku mekanis kedua parameter ini terhadap CPU dan Memori:

```text
┌─────────────────┬──────────────────────────────────┬─────────────────────────────────┐
│ Jenis Sumber    │ Efek dari REQUESTS               │ Efek dari LIMITS                │
│ Daya            │ (Kebutuhan Penjadwalan)          │ (Batas Maksimal Eksekusi)       │
├─────────────────┼──────────────────────────────────┼─────────────────────────────────┤
│ **CPU**         │ Menentukan node mana yang dipilih│ Dibatasi melalui CFS Quota.     │
│ (Kompresibel)   │ oleh kube-scheduler. Dijamin     │ Jika lewat batas: **THROTTLED** │
│                 │ selalu tersedia untuk kontainer. │ (Diperlambat, TIDAK DIBUNUH!).  │
├─────────────────┼──────────────────────────────────┼─────────────────────────────────┤
│ **Memory**      │ Menentukan node penempatan awal. │ Dibatasi melalui cgroup memory. │
│ (Non-Kompresibel│ Cadangan minimal yang dialokasi- │ Jika lewat batas: **OOMKILLED** │
│                 │ kan oleh sistem operasi.         │ (Proses langsung DIBUNUH paksa!)│
└─────────────────┴──────────────────────────────────┴─────────────────────────────────┘
```

### 1. Satuan Pengukuran CPU: Millicores (`m`)
- `1 CPU` di Kubernetes setara dengan **1 vCPU / 1 Core Fisik**.
- CPU dapat dipecah menjadi satuan milicerdas (*millicores*): `1000m = 1 CPU`.
- Nilai `250m` berarti kontainer meminta jaminan $250 / 1000 = 0.25$ core CPU (seperempat core).
- **Perilaku Throttling:** Jika kontainer Anda mencapai limit CPU (misal `1000m`), kernel Linux tidak akan membunuh aplikasi Anda. Kernel menggunakan algoritma *Completely Fair Scheduler (CFS) Bandwidth Control* untuk menahan jatah waktu siklus CPU (*clock cycles*). Akibatnya, aplikasi Anda akan berjalan sangat lambat, latensi meningkat, dan probe kesehatan mungkin mengalami timeout.

### 2. Satuan Pengukuran Memory: Mebibytes (`Mi`) vs Megabytes (`M`)
- **Desimal (`M`, `G`):** Berbasis kelipatan 1000 ($1\text{ M} = 1.000.000\text{ bytes}$).
- **Biner (`Mi`, `Gi`):** Berbasis kelipatan 1024 ($1\text{ Mi} = 1.048.576\text{ bytes}$).
- Di dunia sistem operasi Linux dan SRE, sistem memori komputer dihitung dalam format biner. **Selalu gunakan satuan `Mi` dan `Gi`** untuk menghindari selisih perhitungan matematis yang membingungkan.
- **Perilaku OOMKilled:** Memori adalah sumber daya *non-kompresibel*. Anda tidak bisa "memperlambat" pemakaian RAM. Begitu kontainer menggunakan 513 Mi (melebihi limit 512 Mi), kernel Linux langsung mengeksekusi `SIGKILL` (Exit Code 137).

---

## 3. Hierarki Kualitas Layanan (QoS - Quality of Service)

Ketika sebuah node server kehabisan memori (*Memory Pressure*), kernel Linux harus memilih kontainer mana yang harus "dikorbankan" dan dibunuh terlebih dahulu demi menyelamatkan stabilitas server fisik. Kubernetes mengelompokkan setiap Pod ke dalam **3 Kasta Kualitas Layanan (QoS Class)**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       URUTAN KASTA QOS KUBERNETES                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASTA 1: GUARANTEED (Anak Emas / Prioritas Tertinggi) ]                  │
│  Syarat: Setiap kontainer memiliki CPU & Memory Request == Limit.           │
│  Contoh: Request CPU 500m / Limit 500m, Request RAM 1Gi / Limit 1Gi.        │
│  Nasib: PALING TERAKHIR DIBUNUH! Sangat aman dan terlindungi.               │
│                                                                             │
│  [ KASTA 2: BURSTABLE (Golongan Menengah / Fleksibel) ]                     │
│  Syarat: Memiliki Request dan Limit, tetapi nilainya TIDAK SAMA             │
│          (misal: Request 256Mi, Limit 1Gi).                                 │
│  Nasib: Dibunuh jika memori node habis setelah seluruh BestEffort musnah.   │
│                                                                             │
│  [ KASTA 3: BESTEFFORT (Kasta Terendah / Warga Kelas Tiga) ]                │
│  Syarat: TIDAK MENYEBUTKAN Request maupun Limit sama sekali (Kosong!).      │
│  Nasib: KORBAN PERTAMA! Begitu node tertekan, Pod ini langsung DIBANTAI     │
│         oleh OOM Killer tanpa ampun!                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Cara memeriksa kasta QoS sebuah Pod di terminal:

```bash
$ kubectl get pod payment-api-6789b-4k2l1 -n sre-lab -o jsonpath='{.status.qosClass}{"\n"}'
Burstable
```

> [!WARNING]
> **Anti-Pattern BestEffort di Lingkungan Produksi:**
> Jangan pernah membiarkan aplikasi bisnis produksi berjalan dengan status QoS `BestEffort`. Pod tanpa request/limit dapat menghabiskan seluruh memori node dan menyebabkan proses sistem kritis kluster (*kubelet*, *containerd*) terganggu, memicu node berstatus `NotReady`.

---

## 4. Perhitungan Kapasitas Node (Allocatable vs Capacity)

Sering kali timbul pertanyaan membingungkan dari engineer pemula:
> *"Mengapa Pod saya berstatus `Pending` karena `0/3 nodes are available: Insufficient cpu`, padahal ketika saya cek dengan `kubectl top nodes`, pemakaian CPU riil server saat itu hanya 10%?"*

Jawabannya adalah: **Scheduler menjadwalkan Pod berdasarkan total matematis REQUESTS, bukan berdasarkan penggunaan CPU aktual saat itu!**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STRUKTUR PEMBAGIAN MEMORI PADA NODE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TOTAL KAPASITAS FISIK NODE (Node Capacity: 4 Core / 8 GiB RAM) ]         │
│  ├── 1. Kube-Reserved   : Jatah khusus Kubelet & Containerd (~500 MiB)      │
│  ├── 2. System-Reserved : Jatah OS Linux Host, SSH, systemd (~500 MiB)      │
│  ├── 3. Eviction-Thresh : Batas cadangan darurat (Hard Eviction ~100 MiB)   │
│  └── 4. NODE ALLOCATABLE: JATAH BERSIH UNTUK POD KUBERNETES (~6.9 GiB)      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Rumus alokasi penjadwalan:
$$\text{Allocatable} = \text{Capacity} - \text{Kube-Reserved} - \text{System-Reserved} - \text{Eviction-Threshold}$$

### Contoh Perhitungan Kasus:
- Sebuah node memiliki `Allocatable CPU = 2000m` (2 Core).
- Sudah berjalan 4 Pod di node tersebut, masing-masing memiliki `requests.cpu: 500m`.
- Total reservasi CPU yang terpakai = $4 \times 500\text{m} = 2000\text{m}$ (Kapasitas reservasi penuh 100%).
- Meskipun saat itu keempat aplikasi tersebut sedang santai (*idle*) dan hanya memakai 50m CPU riil di `kubectl top`, **Pod kelima yang meminta `requests.cpu: 200m` akan DITOLAK oleh scheduler dan tertahan di status `Pending`**.
- Ini adalah mekanisme perlindungan Kubernetes untuk menjamin bahwa jika sewaktu-waktu keempat aplikasi tersebut mengalami lonjakan trafik bersamaan, jatah CPU mereka tidak akan dicuri oleh Pod lain!

---

## 5. Pipeline Metrik: Metrics-Server & `kubectl top`

Agar perintah `kubectl top` dan Horizontal Pod Autoscaler (HPA) dapat berfungsi, kluster Anda wajib memiliki komponen **Metrics-Server**:

```text
┌─────────────────┐       Scrape Port 10250       ┌──────────────────┐
│ Kubelet cAdvisor│ ◄──────────────────────────── │  Metrics-Server  │
│ (Tiap Node)     │                               │  (Deployment)    │
└─────────────────┘                               └────────┬─────────┘
                                                           │ Daftarkan API
                                                           ▼
┌─────────────────┐       Kueri Metrik CPU/RAM    ┌──────────────────┐
│ kubectl top /   │ ◄──────────────────────────── │ APIService K8s:  │
│ HPA Controller  │                               │ v1beta1.metrics  │
└─────────────────┘                               └──────────────────┘
```

Perintah pengujian metrik operasional:

```bash
# 1. Melihat konsumsi resource aktual seluruh node
$ kubectl top nodes
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k3s-cp1   142m         7%     890Mi           46%
k3s-cp2   110m         5%     750Mi           39%
k3s-w1    85m          4%     620Mi           32%

# 2. Melihat konsumsi resource Pod diurutkan dari yang paling boros RAM
$ kubectl top pods -n sre-lab --sort-by=memory
NAME                        CPU(cores)   MEMORY(bytes)
analytics-db-88ff-m99x2     12m          412Mi
payment-api-6789b-4k2l1     85m          210Mi
nginx-web-7bc9b-11zxc       2m           18Mi

# 3. Melihat detail penggunaan per kontainer di dalam Pod
$ kubectl top pod payment-api-6789b-4k2l1 -n sre-lab --containers
POD                       NAME          CPU(cores)   MEMORY(bytes)
payment-api-6789b-4k2l1   payment-api   82m          195Mi
payment-api-6789b-4k2l1   log-shipper   3m           15Mi
```

> [!NOTE]
> **K3s Metrics-Server:**
> Di k3s, *metrics-server* sudah terpasang secara bawaan (*built-in*). Jika perintah `kubectl top` menghasilkan error `Metrics API not available`, periksa status Pod metrics server dengan perintah: `kubectl get pods -n kube-system | grep metrics-server`.

---

## 6. Horizontal Pod Autoscaler (HPA v2)

Horizontal Pod Autoscaler (HPA) bertugas menambah atau mengurangi jumlah replika Pod secara otomatis berdasarkan beban komputasi nyata.

### Rumus Matematis Perhitungan Jumlah Replika HPA
HPA menghitung jumlah replika yang diinginkan menggunakan rumus deterministik:

$$\text{desiredReplicas} = \left\lceil \text{currentReplicas} \times \left( \frac{\text{currentMetricValue}}{\text{targetMetricValue}} \right) \right\rceil$$

*Contoh Perhitungan:*
- Jumlah Pod saat ini (`currentReplicas`) = **2**
- Target utilisasi CPU (`targetMetricValue`) = **50%**
- Beban CPU aktual saat terjadi lonjakan (`currentMetricValue`) = **120%**
- Perhitungan:
  $$\text{desiredReplicas} = \left\lceil 2 \times \left( \frac{120}{50} \right) \right\rceil = \lceil 2 \times 2.4 \rceil = \lceil 4.8 \rceil = \mathbf{5\text{ Pod}}$$
- HPA akan seketika menaikkan jumlah replika Deployment dari 2 menjadi 5!

---

### Manifes Deklaratif HPA Modern (API `autoscaling/v2`)
Berikut adalah contoh konfigurasi HPA produksi yang dilengkapi dengan pengatur perilaku (*behavior control*) untuk mencegah fenomena *flapping / thrashing* (Pod naik-turun terlalu cepat):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-api-hpa
  namespace: sre-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60   # Target 60% dari CPU REQUEST!
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0   # Naik instan jika ada lonjakan!
      policies:
      - type: Percent
        value: 100                    # Boleh menambah hingga 100% replika per 15 detik
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300 # Tunggu 5 menit stabil sebelum menurunkan Pod!
      policies:
      - type: Percent
        value: 20                     # Turunkan maksimal 20% Pod per menit (mencegah drop tiba-tiba)
        periodSeconds: 60
```

> [!IMPORTANT]
> **Kunci Sukses HPA: `averageUtilization` Dihitung Terhadap Request!**
> Angka `averageUtilization: 60` berarti rata-rata penggunaan CPU adalah **60% dari nilai `resources.requests.cpu`**, BUKAN 60% dari total CPU mesin fisik node! Jika kontainer Anda tidak memiliki konfigurasi `requests.cpu`, **HPA AKAN GAGAL TOTAL** dan menampilkan status `<unknown>`!

---

## 7. Simulasi Beban Terukur (Hands-On Load Testing)

Mari kita buktikan cara kerja autoscaling secara langsung di lab.

```bash
# 1. Terapkan konfigurasi HPA ke Deployment
$ kubectl apply -f payment-hpa.yaml
horizontalpodautoscaler.autoscaling/payment-api-hpa created

# 2. Pantau status awal HPA
$ kubectl get hpa payment-api-hpa -n sre-lab
NAME              REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
payment-api-hpa   Deployment/payment-api   4%/60%    2         8         2          30s

# 3. Jalankan Pod generator beban tiruan di terminal terpisah
$ kubectl run load-generator -n sre-lab --rm -it --restart=Never \
    --image=busybox:1.36 -- /bin/sh -c \
    "while true; do wget -q -O- http://payment-api.sre-lab.svc.cluster.local:8080/heavy-work; done"

# 4. Amati reaksi HPA dalam menaikkan jumlah replika
$ kubectl get hpa payment-api-hpa -n sre-lab -w
NAME              REFERENCE                TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
payment-api-hpa   Deployment/payment-api   4%/60%     2         8         2          1m
payment-api-hpa   Deployment/payment-api   98%/60%    2         8         2          2m
payment-api-hpa   Deployment/payment-api   115%/60%   2         8         4          2m15s
payment-api-hpa   Deployment/payment-api   58%/60%    2         8         5          3m
```

Perhatikan bagaimana ketika utilisasi mencapai 115%, HPA secara otomatis menambah jumlah Pod menjadi 4, lalu menjadi 5 hingga utilisasi rata-rata per Pod kembali turun di bawah target batas 60%.

---

## 8. Kapan HPA Bukan Solusi? (Peringatan & Anti-Pattern SRE)

Banyak organisasi menganggap HPA adalah "obat ajaib" untuk segala masalah performa. Ini adalah pemikiran yang keliru. HPA **TIDAK AKAN** menyelesaikan masalah berikut:

1. **Bottleneck pada Database / External Backend:** Jika database PostgreSQL Anda mengalami *CPU 100%* karena query SQL tanpa index, menambah replika Pod dari 2 menjadi 20 justru akan membanjiri database dengan ratusan koneksi baru dan membuatnya roboh total (*Thundering Herd Problem*).
2. **Kebocoran Memori (Memory Leak):** Jika aplikasi Anda memiliki bug *memory leak*, HPA berbasis memori hanya akan memperbanyak kontainer yang bocor dan menghabiskan sisa RAM node kluster.
3. **Aplikasi dengan Waktu Inisialisasi Lambat (*Cold Start Latency*):** Jika aplikasi Java/Spring Boot membutuhkan waktu 2 menit hanya untuk start, HPA tidak dapat menolong lonjakan traffic kilat (*flash sale*) karena saat Pod baru siap melayani, lonjakan traffic mungkin sudah selesai dan pengguna sudah terlanjur mengalami error.
4. **Kapasitas Fisik Node Telah Habis:** HPA hanya bisa membuat Pod baru jika node masih memiliki sisa *Allocatable Capacity*. Jika seluruh node sudah penuh, penambahan Pod dari HPA hanya akan tertahan di status `Pending`!

---

## 9. Ringkasan & Cheatsheet Manajemen Resource

```bash
# Melihat sisa jatah kapasitas Allocatable CPU & Memory seluruh node
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU_ALLOC:.status.allocatable.cpu,MEM_ALLOC:.status.allocatable.memory

# Memeriksa total akumulasi alokasi request pada sebuah node
kubectl describe node <node-name> | sed -n '/Allocated resources:/,/Events:/p'

# Memeriksa status dan kesehatan HPA
kubectl describe hpa <hpa-name> -n <ns>

# Melihat penggunaan CPU Pod secara real-time
kubectl top pods -n <ns> --sort-by=cpu
```

---

## 10. Cek Pemahaman Mandiri

1. Mengapa nilai `resources.requests` menjadi satu-satunya dasar bagi *kube-scheduler* untuk menempatkan Pod, dan mengapa scheduler mengabaikan angka penggunaan CPU aktual di `kubectl top`?
2. Jelaskan perbedaan nasib antara kontainer yang melewati batas `limits.cpu` dibandingkan dengan kontainer yang melewati batas `limits.memory`!
3. Sebutkan urutan eksekusi pembunuhan kontainer oleh kernel Linux OOM Killer berdasarkan kelas QoS (`Guaranteed`, `Burstable`, `BestEffort`) saat node mengalami *Memory Pressure*!
4. Jika sebuah Deployment memiliki `requests.cpu: 200m` dan HPA disetel dengan `averageUtilization: 50%`, pada angka penggunaan CPU riil berapakah (dalam satuan milicerdas) HPA akan mulai mempertimbangkan untuk menambah replika?
5. Mengapa penggunaan Horizontal Pod Autoscaler (HPA) berbasis memori (*memory-based scaling*) sering kali tidak direkomendasikan untuk aplikasi berbasis Java atau Go?

---

*Langkah Selanjutnya:* Masuk ke [03 — Snapshot & Restore Embedded etcd k3s](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.4-operasi-troubleshooting/03-etcd-backup-restore-k3s.md) untuk mempelajari bagaimana melindungi data jantung kluster dan melakukan pemulihan bencana (*Disaster Recovery*) jika seluruh control plane mengalami kerusakan.
