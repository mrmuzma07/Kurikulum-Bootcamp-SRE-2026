# 02 — Objek Inti Kubernetes: Pod, Deployment, Service, Ingress, ConfigMap, Secret & Storage

> **Ringkasan:** Memahami blok-blok bangunan dasar yang dikelola oleh seorang SRE setiap hari: menguasai filosofi **Manifes Deklaratif (YAML)**, memahami mengapa **Pod** adalah unit komputasi terkecil, mengelola siklus hidup aplikasi tanpa *downtime* dengan **Deployment & ReplicaSet**, menyediakan jaringan stabil menggunakan **Service** dan **Ingress**, memisahkan konfigurasi dan rahasia lewat **ConfigMap & Secret**, serta mengelola data permanen menggunakan **PersistentVolume (PV) & Claim (PVC)**.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan menyelesaikan latihan praktiknya di terminal, kamu akan mampu:
- Menjelaskan perbedaan mendasar antara pendekatan **Deklaratif (YAML)** dan **Imperatif (CLI)** serta anatomi baku file manifes Kubernetes.
- Menjelaskan mengapa Kubernetes menggunakan abstraksi **Pod** (bukan container telanjang) serta konsep *Multi-Container / Sidecar Pattern*.
- Mengelola replika dan pembaruan versi aplikasi tanpa *downtime* (**RollingUpdate**) serta memulihkan kegagalan menggunakan fitur **Rollback**.
- Membedakan 3 tipe **Service** (`ClusterIP`, `NodePort`, `LoadBalancer`) dan memahami bagaimana **Label Selector** menghubungkan Service ke Pod melalui **Endpoints**.
- Menjelaskan peran **Ingress Controller** sebagai perute lalu lintas HTTP/HTTPS berbasis nama domain dan *path*.
- Mengamankan dan menginjeksi konfigurasi aplikasi menggunakan **ConfigMap** dan **Secret** (baik via *Environment Variables* maupun *File Mount*).
- Menjelaskan mengapa Secret berbasis Base64 **bukan** enkripsi dan memahami bahayanya jika di-commit ke Git.
- Menghubungkan penyimpanan permanen ke dalam Pod menggunakan **PersistentVolumeClaim (PVC)** dan **StorageClass**.

---

## 1. Filosofi Inti: Deklaratif vs Imperatif

Di dunia komputasi tradisional (dan saat menggunakan `docker run`), kita terbiasa dengan gaya **Imperatif**:
*"Komputer, tolong jalankan perintah A, lalu jalankan perintah B, lalu hapus folder C!"*
Pendekatan ini sangat rentan gagal jika di tengah jalan terjadi gangguan jaringan atau server mati separuh jalan.

### Pendekatan Deklaratif Kubernetes
Kubernetes dirancang sejak awal menggunakan filosofi **Deklaratif**:
Kita tidak memberi tahu Kubernetes *bagaimana cara* bekerja. Kita hanya menyerahkan selembar dokumen kontrak tertulis (file manifes **YAML**) yang menyatakan: **"Ini kondisi akhir yang saya inginkan (*Desired State*). Tolong wujudkan dan pertahankan kondisi ini selamanya!"**

```text
 SRE Menulis Manifes YAML (Desired State):
 "Saya ingin ada 3 Pod web dengan image v1.0.0"
                    │
                    ▼ (kubectl apply -f app.yaml)
         ┌─────────────────────┐
         │ Kubernetes Control  │ ◄─── Loop Rekonsiliasi Otomatis
         │      Plane          │      Memeriksa setiap detik:
         └──────────┬──────────┘      "Apakah jumlah Pod aktual == 3?"
                    │
                    ▼ (Jika aktual baru ada 2 Pod...)
         K8s otomatis menyalakan 1 Pod tambahan di Node yang sehat!
```

---

### Anatomi Baku File Manifes YAML Kubernetes
Setiap dokumen YAML di Kubernetes—mulai dari Pod paling sederhana hingga aturan keamanan paling rumit—selalu memiliki **5 bagian kerangka baku**:

```yaml
apiVersion: apps/v1        # 1. Versi API skema objek yang digunakan
kind: Deployment           # 2. Jenis objek apa yang ingin dibuat
metadata:                  # 3. KTP identitas objek (nama, namespace, label)
  name: web-aplikasi
  namespace: default
  labels:
    app: web
    tier: frontend
spec:                      # 4. Spesifikasi teknis (Desired State yang kita inginkan)
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:                # Cetakan Pod yang akan diproduksi
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
# status:                  # 5. Status aktual saat ini (Otomatis diisi oleh k8s, jangan ditulis manual!)
```

---

## 2. Objek 1: Pod — Unit Komputasi Terkecil

### Mengapa Unit Terkecilnya Pod, Bukan Container Langsung?
Banyak pemula mengira bahwa objek terkecil di Kubernetes adalah Container. **Itu keliru!**
Unit terkecil yang bisa dibuat dan dijadwalkan oleh Kubernetes disebut **Pod**.

> 🥜 **Analogi Polong Kacang (Pea Pod):**
> Nama "Pod" diambil dari polong kacang polong (*pea pod*). Di dalam satu kulit polong kacang, bisa terdapat satu, dua, atau tiga butir biji kacang (**Container**).
> - Seluruh container di dalam satu Pod yang sama **tinggal di kamar yang sama**.
> - Mereka **berbagi Alamat IP yang sama** dan ruang port yang sama (Namespace `net`).
> - Mereka bisa saling memanggil dengan kecepatan kilat cukup menggunakan alamat **`localhost`**!
> - Mereka bisa berbagi media penyimpanan disk yang sama (*Shared Volumes*).

```text
 ┌─ POD: Alamat IP Tunggal: 10.42.1.15 ──────────────────────────────────────┐
 │                                                                           │
 │  ┌─────────────────────────────┐         ┌─────────────────────────────┐  │
 │  │ Container Utama (Aplikasi)  │         │ Container Pembantu (Sidecar)│  │
 │  │ Menjalankan Web Go / Python │ ◄─────► │ Pengirim Log / Proxy Envoy  │  │
 │  │ Port: 8080                  │localhost│ Port: 9090                  │  │
 │  └──────────────┬──────────────┘         └──────────────┬──────────────┘  │
 │                 │                                       │                 │
 │                 └───────────────┬───────────────────────┘                 │
 │                                 ▼                                         │
 │                 📁 Shared Volume: /var/log/app                            │
 └───────────────────────────────────────────────────────────────────────────┘
```

### Konsep Multi-Container: Pola Sidecar (Sidecar Pattern)
Kapan kita menaruh lebih dari 1 container di dalam satu Pod?
Ketika ada container pembantu (*helper*) yang tugasnya melayani container utama, misalnya:
- Container A menjalankan aplikasi web yang menulis file log ke `/var/log`.
- Container B (**Sidecar**) bertugas membaca file log tersebut dan mengirimkannya ke Grafana Loki di data center.
Keduanya harus selalu hidup berdampingan di server fisik yang sama!

---

### Manifes Pod Tunggal & Pembatasan Resource

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  labels:
    app: demo
spec:
  containers:
  - name: web
    image: nginx:1.27-alpine
    ports:
    - containerPort: 80
    resources:
      requests:          # Jatah MINIMAL yang wajib disediakan node agar Pod mau masuk
        cpu: "100m"      # 100 millicores (0.1 CPU Core)
        memory: "128Mi"  # 128 Megabyte RAM
      limits:            # Plafon MAKSIMAL yang tidak boleh dilewati
        cpu: "500m"      # 0.5 CPU Core (jika lewat, CPU direm/throttled)
        memory: "256Mi"  # 256 Megabyte RAM (jika lewat, dibunuh! OOMKilled Exit 137)
```

> 🚨 **Aturan Emas SRE: Dilarang Membuat "Pod Telanjang" (*Naked Pod*) di Production!**
> Jika kamu membuat Pod langsung menggunakan `kind: Pod`, Pod tersebut berstatus "telanjang" tanpa pelindung. Jika server fisik tempat Pod itu hidup mati terbakar, Pod tersebut **akan mati selamanya dan tidak akan pernah dibangkitkan ulang**! Di production, kita selalu membungkus Pod di dalam **Deployment**.

---

## 3. Objek 2: Deployment & ReplicaSet — Manajemen Siklus Hidup Tanpa Downtime

### Hubungan Segitiga: Deployment ➔ ReplicaSet ➔ Pod

```text
 ┌─────────────────────────────────────────────────────────┐
 │               DEPLOYMENT (web-deployment)               │
 │  - Mengatur strategi rilis (RollingUpdate / Recreate)   │
 │  - Menyimpan riwayat versi revisi (Revisions 1, 2, 3)   │
 └────────────────────────────┬────────────────────────────┘
                              │ Mengendalikan
                              ▼
 ┌─────────────────────────────────────────────────────────┐
 │              REPLICASET (web-deployment-7b4f)           │
 │  - Tugas tunggal: Memastikan jumlah Pod SELALU pas (3)  │
 └────────────────────────────┬────────────────────────────┘
                              │ Mencetak & Memantau
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
   ┌───────────┐        ┌───────────┐        ┌───────────┐
   │   Pod 1   │        │   Pod 2   │        │   Pod 3   │
   └───────────┘        └───────────┘        └───────────┘
```

1. **Deployment:** Bertanggung jawab atas strategi evolusi aplikasi (pembaruan versi image, rollback saat bug, dan pencatatan audit log perubahan).
2. **ReplicaSet:** Mesin penghitung otomatis yang bertugas memastikan jumlah Pod di lapangan selalu persis sama dengan angka `replicas: N`.
3. **Pod:** Wujud nyata dari proses yang sedang melayani pengunjung.

---

### Strategi Pembaruan Tanpa Downtime: `RollingUpdate`
Secara default, Deployment menggunakan strategi **`RollingUpdate`**.
Ketika kamu mengganti versi aplikasi dari `v1.0.0` ke `v2.0.0`, Deployment **tidak mematikan seluruh aplikasi secara serentak**!
- Deployment membuat ReplicaSet baru untuk versi `v2.0.0`.
- 1 Pod `v2.0.0` baru dinyalakan. Begitu Pod baru tersebut lulus uji kesehatan (*Readiness Probe*), barulah 1 Pod `v1.0.0` lama dimatikan.
- Proses bergulir bertahap ini berlanjut satu per satu hingga seluruh Pod beralih ke versi baru tanpa ada pelanggan yang mengalami halaman error (nol detik *downtime*)!

### Contoh Manifes Deployment Lengkap Production

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  labels:
    app: payment
spec:
  replicas: 3
  revisionHistoryLimit: 5   # Simpan 5 riwayat revisi terakhir untuk rollback
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Boleh membuat maksimal 1 Pod ekstra di atas kuota saat update
      maxUnavailable: 0     # DILARANG ada Pod yang mati sebelum Pod baru benar-benar siap!
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: api
        image: registry.gitlab.com/budi/sre-bootcamp/payment:v1.0.0
        ports:
        - containerPort: 8080
        readinessProbe:     # Penjaga gerbang: Pod baru belum boleh menerima trafik sebelum ini lulus!
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
```

### Perintah Kendali Rollout & Rollback SRE
```bash
# Terapkan perubahan
kubectl apply -f deployment.yaml

# Pantau proses pembaruan bergulir secara live:
kubectl rollout status deployment/payment-api

# Lihat riwayat revisi pembaruan masa lalu:
kubectl rollout history deployment/payment-api

# Bencana jam 2 pagi: Kembalikan aplikasi ke versi 1 langkah sebelumnya seketika!
kubectl rollout undo deployment/payment-api
```

---

## 4. Objek 3: Service — Alamat IP & DNS Stabil di Atas Pod yang Fana

### Masalah Besar: Pod Itu Bersifat Fana (*Ephemeral*)
Di Kubernetes, Pod dirancang untuk bisa mati dan lahir kembali kapan saja.
- Setiap kali Pod mati lalu dibuat ulang oleh ReplicaSet, Pod baru tersebut akan mendapatkan **Alamat IP yang benar-benar baru**!
- Bayangkan jika aplikasi frontend harus menghubungkan database menggunakan IP Pod langsung. Setiap kali database di-restart, frontend akan langsung putus koneksi karena IP-nya berganti!

### Solusi: Kubernetes Service
**Service** adalah sebuah abstraksi yang menyediakan **Satu Alamat IP Virtual (ClusterIP) dan Satu Nama DNS yang Permanen** di depan sekelompok Pod.
Aplikasi pemanggil cukup menghubungi alamat Service. Service yang bertindak sebagai *Load Balancer* internal untuk membagi lalu lintas ke Pod-Pod di belakangnya.

```text
                           ┌───────────────────────────────┐
                           │    SERVICE: backend-svc       │
                           │    IP Stabil: 10.43.50.100    │
                           │    DNS: backend-svc.default   │
                           └───────────────┬───────────────┘
                                           │
                           Label Selector: │ app=backend
                                           ▼
             ┌─────────────────────────────┼─────────────────────────────┐
             ▼                             ▼                             ▼
       ┌───────────┐                 ┌───────────┐                 ┌───────────┐
       │   Pod 1   │                 │   Pod 2   │                 │   Pod 3   │
       │IP: 10.42.1│                 │IP: 10.42.2│                 │IP: 10.42.3│
       └───────────┘                 └───────────┘                 └───────────┘
```

---

### Bagaimana Service Menemukan Pod Miliknya? (Label Selector & Endpoints)
Perhatikan bagian kuncinya: Service tidak melacak Pod berdasarkan nama Pod, melainkan mencarinya menggunakan radar stiker **`selector: matchLabels`**.
Setiap kali ada Pod berlabel cocok yang lulus uji *Readiness Probe*, IP Pod tersebut otomatis dicatat oleh Kubernetes ke dalam buku daftar penerima tamu bernama objek **Endpoints (atau EndpointSlice)**.

```bash
# Memeriksa daftar IP Pod yang saat ini sedang dilayani oleh Service:
kubectl get endpoints <nama-service>
```

---

### Tiga Tipe Service yang Wajib Dikuasai

| Tipe Service | Karakteristik Akses Jaringan | Kapan Menggunakannya? |
|---|---|---|
| **`ClusterIP`** *(Default)* | Hanya memiliki alamat IP virtual internal cluster. **Sama sekali tidak bisa diakses dari luar cluster.** | Komunikasi internal antar-layanan (Database, Redis Cache, API Backend privat). |
| **`NodePort`** | Membuka satu nomor port statis di rentang **30000–32767** pada **seluruh IP komputer node fisik/VM**. | Keperluan pengujian langsung atau integrasi jaringan lawas. Jarang dipakai langsung di production modern. |
| **`LoadBalancer`** | Meminta alokasi Alamat IP eksternal mandiri ke infrastruktur jaringan. | Mengekspos layanan langsung ke publik.<br>• Di Cloud: Membuat AWS NLB / GCP LB.<br>• **Di On-Premises Bare-Metal (Bootcamp Kita): Dilayani oleh MetalLB (Modul 2.3)!** |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  type: ClusterIP          # Ganti menjadi LoadBalancer jika ingin IP eksternal
  selector:
    app: payment           # Sambungkan ke Pod yang berlabel 'app: payment'
  ports:
  - port: 80               # Port yang dibuka oleh Service
    targetPort: 8080       # Port asli yang didengarkan di dalam Container Pod
```

> 🌐 **Format Nama DNS Internal Kubernetes:**
> Setiap Service otomatis memiliki nama domain internal cluster dengan struktur:
> **`<nama-service>.<nama-namespace>.svc.cluster.local`**
> Jika Pod aplikasi frontend berada di namespace yang sama dengan Service `payment-service`, aplikasi frontend cukup memanggil URL: `http://payment-service:80`!

---

## 5. Objek 4: Ingress — Pintu Gerbang HTTP/HTTPS Layer 7

### Mengapa Kita Butuh Ingress Jika Sudah Ada Service LoadBalancer?
Bayangkan kamu memiliki **30 aplikasi web mikro**.
Jika setiap aplikasi web meminta `type: LoadBalancer`, kamu harus menyediakan **30 Alamat IP Publik** yang sangat mahal dan boros!

Di sinilah **Ingress** hadir sebagai solusinya:
Ingress adalah perute lalu lintas cerdas di tingkat aplikasi (**Layer 7 HTTP/HTTPS**) yang memungkinkan puluhan aplikasi berbeda berbagi **SATU Alamat IP LoadBalancer yang sama** menggunakan pemilahan nama domain (*Virtual Host*) atau awalan jalur URL (*Path Routing*).

```text
                              Satu IP Publik: 198.19.249.50
                                            │
                                            ▼
                           ┌─────────────────────────────────┐
                           │    INGRESS CONTROLLER (Traefik) │
                           └───────────────┬─────────────────┘
                                           │
                   ┌───────────────────────┴───────────────────────┐
                   │ Membaca Host Header                           │ Membaca Host Header
                   ▼                                               ▼
         Host: toko.sre.local                            Host: api.sre.local
  ┌───────────────────────────────┐               ┌───────────────────────────────┐
  │   Service: frontend-service   │               │    Service: backend-service   │
  └───────────────────────────────┘               └───────────────────────────────┘
```

### Manifes Ingress Standar
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gerbang-utama
spec:
  rules:
  - host: toko.k3d.local             # Rute berdasarkan nama domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
```

> 🚦 **Ingress Resource vs Ingress Controller:**
> - Manifes di atas hanyalah selembar **Kertas Aturan Rute (Ingress Resource)**.
> - Kertas aturan ini tidak akan berguna jika di dalam cluster tidak ada petugas polisi yang menjalankannya (**Ingress Controller**).
> - Di **k3s dan k3d**, polisi lalu lintas ini sudah terpasang otomatis secara bawaan berupa **Traefik Ingress Controller**! Di server Kubernetes lain, orang sering memasang **Nginx Ingress Controller**.

---

## 6. Objek 5 & 6: ConfigMap & Secret — Memisahkan Konfigurasi dari Kode

Sesuai metodologi standar industri modern (**The Twelve-Factor App**), kode program aplikasi tidak boleh memuat alamat database atau password secara tertulis mati (*hardcoded*). Kode aplikasi harus dibuat seragam, sedangkan konfigurasinya disuntikkan dari luar.

### 1. ConfigMap (Untuk Teks Konfigurasi Biasa)
Digunakan untuk menyimpan konfigurasi teks bebas non-sensitif (seperti nama database, tingkat verbositas log `LOG_LEVEL=info`, atau file `nginx.conf`).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: konfigurasi-aplikasi
data:
  APP_ENV: "production"
  DATABASE_HOST: "postgres-service.database.svc.cluster.local"
  app-settings.json: |
    {
      "timeoutSeconds": 30,
      "maxRetries": 3
    }
```

---

### 2. Secret (Untuk Kredensial & Berkas Rahasia)
Digunakan untuk menyimpan informasi sensitif seperti kata sandi database, API Token GitLab, atau sertifikat TLS HTTPS.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rahasia-database
type: Opaque
stringData:                  # Tips Praktis: Gunakan stringData agar tidak repot encode manual!
  DB_PASSWORD: "PasswordSuperRahasia123!"
```

> ⚠️ **PERINGATAN KERAS KEAMANAN SRE: Base64 BUKAN Enkripsi!**
> Jika kamu membuat Secret menggunakan kolom `data:` (bukan `stringData:`), nilainya harus disandikan dalam format **Base64**.
> Ingat baik-baik: **Base64 HANYALAH PENYANDIAN KARAKTER, BUKAN ENKRIPSI!**
> Siapa pun yang memiliki izin membaca file Secret bisa mendekode password tersebut dalam 1 detik:
> ```bash
> echo "UGFzc3dvcmRTdXBlclJhaGFzaWExMjMh" | base64 -d
> ```
> Oleh karena itu, **DILARANG KERAS meng-commit file Secret YAML biasa ke dalam repositori Git!**
> Di modul-modul GitOps lanjutan (Fase 6), kita akan mempelajari cara mengenkripsi file Secret menggunakan alat seperti **HashiCorp Vault, Sealed Secrets, atau Mozilla SOPS** sebelum di-commit ke Git.

---

### Cara Menyuntikkan ConfigMap & Secret ke Dalam Pod

Ada dua metode utama menyuntikkan data ke dalam container Pod:

```text
                        ┌───────────────────────────────┐
                        │      ConfigMap / Secret       │
                        └───────────────┬───────────────┘
                                        │
                 ┌──────────────────────┴──────────────────────┐
                 ▼ (Metode A: Environment)                     ▼ (Metode B: Volume Mount)
   ┌───────────────────────────┐                 ┌───────────────────────────┐
   │ Variabel Lingkungan di OS │                 │ Menjadi File Teks Nyata   │
   │ printenv APP_ENV          │                 │ cat /etc/config/app.json  │
   └───────────────────────────┘                 └───────────────────────────┘
```

#### Contoh Sintaks YAML Injeksi:
```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    # METODE A: Disuntikkan sebagai Environment Variable
    env:
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: rahasia-database
          key: DB_PASSWORD
    # METODE B: Disuntikkan sebagai File Konfigurasi di dalam folder
    volumeMounts:
    - name: folder-config
      mountPath: /etc/app-config
      readOnly: true
  volumes:
  - name: folder-config
    configMap:
      name: konfigurasi-aplikasi
```

---

## 7. Objek 7: PV & PVC — Penyimpanan Data Persisten

Ingat kembali sifat dasar Pod: **Pod bersifat fana (*ephemeral*)**. Jika sebuah Pod database PostgreSQL mati dan digantikan oleh Pod baru, seluruh tabel database di dalam container lama akan musnah jika tidak disimpan di media persisten!

Kubernetes memisahkan urusan penyimpanan menjadi dua peran:

```text
 TIM INFRASTRUKTUR / SRE                           DEVELOPER / APLIKASI
 ┌────────────────────────────────┐               ┌────────────────────────────────┐
 │     PersistentVolume (PV)      │               │  PersistentVolumeClaim (PVC)   │
 │                                │ ◄── Terikat ──│                                │
 │ "Saya sediakan lahan harddisk  │    (Bound)    │ "Aplikasi database saya butuh  │
 │  sebesar 100 GB di NAS / SAN"  │               │  lahan penyimpanan 10 GB!"     │
 └────────────────────────────────┘               └────────────────────────────────┘
```

1. **PersistentVolume (PV):** Lahan penyimpanan fisik/virtual nyata yang ada di cluster (disediakan manual oleh admin, atau otomatis dibuat oleh penyedia storage).
2. **PersistentVolumeClaim (PVC):** Formulir permohonan sewa lahan yang diajukan oleh Pod aplikasi.

### Dynamic Volume Provisioning (`StorageClass`)
Di k3s dan k3d, kamu tidak perlu repot membuat file `PersistentVolume` secara manual satu per satu.
k3s menyertakan sistem otomatis bernama **Local Path Provisioner (`local-path`)**:
Cukup buat sebuah `PersistentVolumeClaim`, maka Kubernetes akan **langsung mencetak sebuah PV baru secara otomatis di harddisk host**!

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce            # RWO: Hanya boleh ditulis oleh 1 Node pada satu waktu
  resources:
    requests:
      storage: 1Gi           # Minta jatah ruang 1 Gigabyte
```

Untuk memasangnya ke Pod, cukup sambungkan klaim tersebut di bagian `spec.volumes`:
```yaml
      volumes:
      - name: data-storage
        persistentVolumeClaim:
          claimName: data-postgres-pvc
```

---

## 8. Objek 8: Label, Selector & Namespace — Tata Kelola Cluster

### 1. Label & Selector: Perekat Ekosistem Kubernetes
Label adalah pasangan kata kunci (*Key-Value Pair*) yang ditempelkan di bagian `metadata.labels` pada setiap objek.
Kubernetes tidak mengelompokkan objek menggunakan folder, melainkan menggunakan **Label Selector**:
- Service mencari Pod mana yang harus dilayani menggunakan Label Selector.
- Deployment mencari Pod mana yang harus dijaga replikanya menggunakan Label Selector.
- Tim SRE mencari Pod mana yang sedang error menggunakan Label Selector:
  ```bash
  kubectl get pods -l app=payment,tier=backend,env=production
  ```

### 2. Namespace: Pembagian Wilayah Cluster Logis
Namespace bertindak seperti **dinding sekat ruangan** di dalam satu cluster fisik yang sama.
- Kita bisa memisahkan lingkungan: namespace `development`, `staging`, dan `production`.
- Seluruh komponen inti k8s sendiri ditaruh di dalam namespace khusus: **`kube-system`**.
- Objek dengan nama yang sama (misal Service bernama `database`) boleh ada di namespace `staging` dan namespace `prod` secara bersamaan tanpa saling bentrok!

```bash
# Membuat namespace baru:
kubectl create namespace staging

# Melihat objek di namespace tertentu:
kubectl get pods -n staging

# Melihat objek di SELURUH namespace yang ada:
kubectl get pods -A
```

---

## 9. Hands-On Mandiri Step-by-Step (30 Menit)

Mari kita gabungkan seluruh objek inti di atas menjadi satu kesatuan aplikasi web yang utuh di cluster `k3d` milikmu:

```bash
# --- LANGKAH 1: Persiapan Folder Latihan ---
cd ~
mkdir -p lab-k8s-objek && cd lab-k8s-objek

# --- LANGKAH 2: Tulis Manifes Gabungan Deklaratif (app-stack.yaml) ---
cat << 'EOF' > app-stack.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-cfg
data:
  PESAN_SAMBUTAN: "Halo SRE! Objek ConfigMap berhasil disuntikkan!"
---
apiVersion: v1
kind: Secret
metadata:
  name: web-secret
type: Opaque
stringData:
  API_KEY: "RahasiaTingkatTinggiSRE2026"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-dep
  labels:
    app: portal
spec:
  replicas: 2
  selector:
    matchLabels:
      app: portal
  template:
    metadata:
      labels:
        app: portal
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        env:
        - name: SAMBUTAN
          valueFrom:
            configMapKeyRef:
              name: web-cfg
              key: PESAN_SAMBUTAN
        - name: TOKEN
          valueFrom:
            secretKeyRef:
              name: web-secret
              key: API_KEY
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: portal
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: portal.k3d.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

# --- LANGKAH 3: Terapkan Seluruh Tumpukan Objek Sekaligus ---
kubectl apply -f app-stack.yaml

# --- LANGKAH 4: Verifikasi Keberhasilan Seluruh Objek ---
kubectl get deploy,rs,pod,svc,ingress,cm,secret -l app=portal

# Periksa apakah variabel ConfigMap & Secret sukses masuk ke dalam Pod:
POD_NAME=$(kubectl get pods -l app=portal -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it "$POD_NAME" -- env | grep -E "SAMBUTAN|TOKEN"

# --- LANGKAH 5: Uji Akses Melalui Ingress Traefik ---
# Akses dari Mac menggunakan Host Header:
curl -H "Host: portal.k3d.local" http://localhost:8080/

# --- LANGKAH 6: Bersihkan Seluruh Objek Latihan ---
kubectl delete -f app-stack.yaml
cd ~ && rm -rf lab-k8s-objek
```

---

## 10. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 masalah klasik saat mengoperasikan objek Kubernetes beserta cara diagnosanya:

### Kasus 1: Service Mengembalikan `502 Bad Gateway` atau `curl: Empty reply`
- **Gejala:** Service dan Ingress berhasil dibuat, namun saat diakses tidak ada respon sama sekali.
- **Penyebab:** Label Selector pada Service salah ketik (*typo*), sehingga Service tidak bisa menemukan Pod mana pun.
- **Diagnosa SRE:**
  Periksa objek **Endpoints** milik service tersebut:
  ```bash
  kubectl get endpoints <nama-service>
  ```
  Jika kolom `ENDPOINTS` tertulis `<none>`, berarti Service tidak tersambung ke satu Pod pun! Cocokkan kembali ejaan `spec.selector` di Service dengan `metadata.labels` di template Pod.

---

### Kasus 2: Pod Gagal Menyala dengan Status `CreateContainerConfigError`
- **Gejala:** Pod tidak mau jalan dan terus menampilkan status error konfigurasi.
- **Penyebab:** Pod mereferensikan nama ConfigMap atau Secret yang **belum dibuat**, atau nama `key` di dalam Secret salah ketik.
- **Diagnosa SRE:**
  Jalankan `kubectl describe pod <nama-pod>` dan periksa bagian `Events`:
  `Error: secret "rahasia-db" not found`. Buat Secret yang diminta, maka Pod akan otomatis menyala normal.

---

### Kasus 3: PVC Terjebak Selamanya di Status `Pending`
- **Gejala:** Objek `PersistentVolumeClaim` tidak kunjung berstatus `Bound`.
- **Penyebab:** Nama `storageClassName` tidak cocok dengan yang tersedia di cluster, atau kuota disk node sudah habis.
- **Diagnosa SRE:**
  Jalankan `kubectl describe pvc <nama-pvc>` dan periksa baris `Events` untuk melihat alasan penolakan provisioner.

---

## 11. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Objek Inti Kubernetes (Cheat Sheet)

| Objek | Fungsi Utama Sederhana | Perintah Cepat Diagnosa |
|---|---|---|
| **Pod** | Unit komputasi terkecil (1+ container) | `kubectl get pods -o wide` |
| **Deployment** | Menjaga jumlah replika & pembaruan versi | `kubectl rollout status deploy/<name>` |
| **Service** | IP & DNS stabil di depan Pod yang dinamis | `kubectl get endpoints <name>` |
| **Ingress** | Gerbang rute HTTP domain/path Layer 7 | `kubectl get ingress` |
| **ConfigMap** | Menyimpan teks konfigurasi non-sensitif | `kubectl get cm <name> -o yaml` |
| **Secret** | Menyimpan data sensitif (Base64) | `kubectl get secret <name> -o yaml` |
| **PVC** | Klaim sewa media penyimpanan persisten | `kubectl get pvc` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk mengevaluasi pemahamanmu tentang objek inti Kubernetes:

1. **Jelaskan perbedaan mendasar antara cara kerja `Deployment` dan `ReplicaSet`! Mengapa seorang SRE hampir selalu membuat Deployment dan tidak pernah membuat ReplicaSet secara langsung?**
2. **Sebuah Pod memiliki alamat IP `10.42.1.20`. Mengapa aplikasi backend lain sangat dilarang melakukan koneksi database langsung ke alamat IP tersebut? Objek apa yang harus dipasang di depannya?**
3. **Sebutkan perbedaan karakteristik antara Service bertipe `ClusterIP`, `NodePort`, dan `LoadBalancer`! Di lingkungan on-premises bare-metal, komponen apa yang bertugas memberikan IP untuk tipe `LoadBalancer`?**
4. **Mengapa menyimpan password database di dalam file `Secret` bawaan Kubernetes belum bisa dikategorikan aman 100% jika file manifesnya diunggah ke repositori GitLab publik?**
5. **Apa yang akan terjadi pada data di dalam folder database jika sebuah Pod terhapus, pada kasus (A) Pod menggunakan PVC persisten, vs kasus (B) Pod tidak menggunakan PVC?**

---
*Lanjutkan ke materi berikutnya: [03-k3d-latihan-harian.md](03-k3d-latihan-harian.md) untuk mendalami strategi pengelolaan cluster k3d sebagai arena latihan harian yang tangguh, aman, dan dapat dihancurkan berulang kali tanpa rasa takut!*