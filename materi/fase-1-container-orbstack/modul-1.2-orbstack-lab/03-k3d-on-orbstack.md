# 03 — k3d di Atas OrbStack: Jembatan Cepat Menuju Kubernetes

> **Ringkasan:** Memahami perbedaan antara **`k8s`**, **`k3s`**, dan **`k3d`**, mempelajari arsitektur bagaimana sebuah cluster Kubernetes multi-node dapat berjalan di dalam container Docker hanya dalam hitungan detik, menguasai penggunaan **`k3d`** dan **`kubectl`**, memahami mekanisme perutean port **`@loadbalancer`**, mempelajari trik **`k3d image import`**, serta mengetahui kapan harus memilih `k3d` (untuk eksperimen harian cepat) vs `k3s` di atas VM (untuk simulasi production on-premises).

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal MacBook, kamu akan mampu:
- Menjelaskan perbedaan definisi dan fungsi antara **Kubernetes (`k8s`)**, **Rancher k3s (`k3s`)**, dan **k3s-in-Docker (`k3d`)**.
- Membedah komponen container yang menyusun sebuah cluster `k3d` (*Server/Control-Plane*, *Agent/Worker*, dan *LoadBalancer Proxy*).
- Membedakan peran alat baris perintah **`k3d`** (pengelola infrastruktur cluster) dan **`kubectl`** (pengendali objek di dalam cluster) beserta konsep **Kubeconfig Context**.
- Membangun cluster Kubernetes multi-node (1 Server + 2 Agent) dengan pemetaan port ke laptop Mac.
- Men-deploy aplikasi container, memperbanyak jumlah replika (*scaling*), dan menyuntikkan image lokal langsung ke dalam cluster menggunakan `k3d image import`.
- Menentukan secara tepat kapan harus menggunakan `k3d` dan kapan harus menggunakan `k3s` di atas OrbStack Linux Machine pada fase-fase bootcamp berikutnya.

---

## 1. Konsep & Arsitektur: Mengurai Singkatan `k8s`, `k3s`, dan `k3d`

Saat memasuki dunia infrastruktur modern, pemula sering kali pusing melihat banyaknya istilah yang diawali huruf **"k"** dan diikuti angka. Mari kita jernihkan ketiganya sekali untuk selamanya!

### 1. Apa itu `k8s` (Kubernetes)?
Jika kamu baru punya 2 atau 3 container di satu server, kamu masih bisa mengelolanya secara manual dengan `docker run` atau `docker compose`.
Tetapi bayangkan jika perusahaanmu memiliki **500 container** yang tersebar di **20 server fisik**! Bagaimana jika server nomor 5 tiba-tiba mati lampu? Siapa yang bertugas memindahkan container dari server nomor 5 ke server nomor 6 secara otomatis jam 3 pagi?
Tugas itulah yang dikerjakan oleh **Kubernetes** (disingkat **`k8s`**, karena ada 8 huruf di antara huruf `'k'` dan `'s'`). Kubernetes adalah sistem pengatur (*Container Orchestrator*) standar dunia. Namun, instalasi Kubernetes murni (*vanilla k8s*) terkenal sangat berat dan rumit.

### 2. Apa itu `k3s` (Lightweight Kubernetes)?
Untuk mengatasi kerumitan instalasi Kubernetes murni, diciptakanlah **`k3s`** (proyek resmi di bawah naungan CNCF).
- Mengapa dinamai **`k3s`**? Para pembuatnya ingin membuat Kubernetes yang ukurannya **"setengah kali lebih ringan"** dari `k8s` (setengah dari angka 8 adalah 4, bahkan dikurangi lagi menjadi **3**!).
- Seluruh komponen Kubernetes yang rumit dikemas menjadi **satu file biner tunggal berukuran di bawah 100 MB**.
- Meskipun ringan, `k3s` adalah **100% Kubernetes asli yang tersertifikasi resmi**. Di Bootcamp SRE ini, `k3s` adalah mesin utama yang akan kita instal di atas server Linux untuk simulasi **Production On-Premises** (Fase 2.2 & Fase 4).

### 3. Apa itu `k3d` (k3s-in-Docker)?
Huruf **"d"** pada **`k3d`** adalah singkatan dari **Docker**.
Normalnya, untuk membuat cluster Kubernetes yang terdiri dari 3 server (1 Master + 2 Worker), kamu harus membuat 3 buah Virtual Machine (VM) Linux yang berat.
**`k3d`** mengambil pendekatan jenius: daripada membuat 3 VM berat, **`k3d` membungkus program `k3s` dan menjalankannya sebagai container Docker biasa!**

```text
 ┌─ Di Dalam Mesin Docker OrbStack ──────────────────────────────────────────┐
 │                                                                           │
 │  ┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐ │
 │  │ Container Docker 1 │   │ Container Docker 2 │   │ Container Docker 3 │ │
 │  │                    │   │                    │   │                    │ │
 │  │  k3d-lab-server-0  │   │  k3d-lab-agent-0   │   │  k3d-lab-agent-1   │ │
 │  │  (Otak Pengendali  │   │  (Pekerja / Node   │   │  (Pekerja / Node   │ │
 │  │   Control Plane)   │   │   Tempat Pod Jalan)│   │   Tempat Pod Jalan)│ │
 │  └────────────────────┘   └────────────────────┘   └────────────────────┘ │
 │                                     ▲                                     │
 │                                     │                                     │
 │                        ┌────────────┴───────────┐                         │
 │                        │   Container Docker 4   │                         │
 │                        │    k3d-lab-serverlb    │ ◄── Pintu Gerbang Proxy │
 │                        └────────────────────────┘                         │
 └───────────────────────────────────────────────────────────────────────────┘
```

> ✈️ **Analogi Simulator Penerbangan (`k3d`) vs Pesawat Terbang Sungguhan (`k3s` di VM):**
> - **`k3d` (Simulator Penerbangan di Ruang Latihan):** Menyala hanya dalam **5 detik**, sangat hemat RAM, dan jika kamu salah mengutak-atik konfigurasi hingga cluster-nya hancur lebur, kamu cukup mengetik satu perintah hapus lalu membuat cluster baru yang segar dalam 5 detik! Sangat ideal untuk latihan harian mempelajari manifes Kubernetes dan Helm.
> - **`k3s` di atas OrbStack Linux Machine (Pesawat Sungguhan di Landasan Pacu):** Diinstal di atas sistem operasi Ubuntu lengkap dengan `systemd`, aturan firewall, IP statis, dan jaringan bare-metal MetalLB. Inilah yang kita gunakan ketika berlatih menjadi SRE yang sesungguhnya.

---

## 2. Mengenal Dua Alat Komando: `k3d` vs `kubectl`

Saat bekerja dengan cluster ini, kamu akan menggunakan dua perintah CLI yang berbeda. Jangan sampai tertukar fungsinya:

| Nama Alat | Peran & Analogi | Contoh Perintah |
|---|---|---|
| **`k3d`** | **Tukang Bangunan Gedung Cluster.**<br>Tugasnya hanya membuat, menyalakan, mematikan, atau merobohkan cluster dari luar. | `k3d cluster create ...`<br>`k3d cluster list`<br>`k3d cluster delete ...` |
| **`kubectl`** | **Remote Control Pengatur Isi Gedung.**<br>Begitu gedung cluster berdiri, semua perintah untuk menjalankan aplikasi, melihat status Pod, dan mengecek log dilakukan lewat `kubectl`! | `kubectl get nodes`<br>`kubectl get pods -A`<br>`kubectl logs ...` |

### Instalasi `k3d` dan `kubectl` di macOS
Jalankan perintah berikut di terminal Mac milikmu:

```bash
# Instal k3d dan kubectl melalui Homebrew
brew install k3d kubectl

# Verifikasi instalasi keduanya
k3d version
kubectl version --client
```

---

###Bagaimana `kubectl` Tahu Harus Menghubungi Cluster yang Mana? (`~/.kube/config`)
Mungkin kamu bertanya: *"Setelah `k3d` selesai membangun cluster, kok `kubectl` bisa langsung tahu alamat IP dan kunci rahasia untuk masuk ke cluster tersebut?"*

Jawabannya ada pada sebuah file buku alamat khusus di komputer Mac milikmu yang terletak di:
**`~/.kube/config`** (sering disebut sebagai file **Kubeconfig**).
- Setiap kali kamu menjalankan `k3d cluster create <nama>`, program `k3d` secara otomatis menuliskan alamat IP, sertifikat keamanan TLS, dan nama **Context** (`k3d-<nama>`) ke dalam file `~/.kube/config` tersebut!
- Kamu bisa berpindah-pindah kendali antar-cluster kapan saja menggunakan perintah:
  ```bash
  kubectl config get-contexts
  kubectl config use-context k3d-<nama-cluster>
  ```

---

## 3. Membangun Cluster Multi-Node & Memahami Port Mapping

Mari kita bedah perintah standar untuk membangun sebuah cluster Kubernetes mini yang memiliki **1 Node Pengendali (Server/Control-Plane)** dan **2 Node Pekerja (Agents/Workers)**:

```bash
k3d cluster create lab \
  --servers 1 \
  --agents 2 \
  --port "8080:80@loadbalancer"
```

### Membedah Arti Setiap Argumen:
1. **`lab`**: Nama cluster yang kita buat (nanti context-nya bernama `k3d-lab`).
2. **`--servers 1`**: Membuat 1 container node *Control Plane* (tempat berjalannya otak Kubernetes: *API Server*, database *SQLite/etcd*, dan *Scheduler*).
3. **`--agents 2`**: Membuat 2 container node *Worker* (tempat aplikasi-aplikasi kita ditempatkan).
4. **`--port "8080:80@loadbalancer"`**: Bagian ini sangat penting untuk dipahami!

#### Bagaimana Aliran Trafik `--port "8080:80@loadbalancer"` Bekerja?
Karena node-node `k3d` berjalan di dalam container Docker, bagaimana caranya agar browser di laptop Mac kita bisa mengakses web aplikasi yang berjalan di dalam cluster?

```text
 [ Laptop Mac ]           [ Container Proxy k3d ]         [ Dalam Cluster Kubernetes ]
 localhost:8080  ──────►  k3d-lab-serverlb (Port 80) ───► Traefik Ingress ───► Pod Aplikasi
```
Ketika kamu menulis `--port "8080:80@loadbalancer"`, kamu memerintahkan Docker untuk menghubungkan **Port `8080` di laptop Mac** menuju **Port `80` pada container pintu gerbang `k3d-lab-serverlb`**, yang kemudian otomatis diteruskan ke penerima tamu (**Traefik Ingress Controller**) di dalam cluster Kubernetes!

---

### Memeriksa Wujud Fisik Cluster `k3d`

Setelah perintah `k3d cluster create lab ...` selesai (biasanya hanya butuh ~10 detik), mari kita periksa dari dua sudut pandang:

#### 1. Sudut Pandang Kubernetes (`kubectl`):
```bash
kubectl get nodes -o wide
```
*Output:*
```text
NAME               STATUS   ROLES                  AGE   VERSION        INTERNAL-IP   OS-IMAGE
k3d-lab-server-0   Ready    control-plane,master   30s   v1.30.4+k3s1   172.18.0.3    K3s v1.30.4+k3s1
k3d-lab-agent-0    Ready    <none>                 25s   v1.30.4+k3s1   172.18.0.4    K3s v1.30.4+k3s1
k3d-lab-agent-1    Ready    <none>                 25s   v1.30.4+k3s1   172.18.0.5    K3s v1.30.4+k3s1
```
Di mata Kubernetes, ia merasa sedang berjalan di atas 3 buah komputer server yang terpisah!

#### 2. Sudut Pandang Docker (`docker ps`):
Sekarang coba ketik `docker ps` biasa:
```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```
*Output:*
```text
NAMES              IMAGE                            STATUS          PORTS
k3d-lab-serverlb   ghcr.io/k3d-io/k3d-proxy:5.7.4   Up 45 seconds   0.0.0.0:8080->80/tcp, ...
k3d-lab-agent-1    rancher/k3s:v1.30.4-k3s1         Up 45 seconds   
k3d-lab-agent-0    rancher/k3s:v1.30.4-k3s1         Up 45 seconds   
k3d-lab-server-0   rancher/k3s:v1.30.4-k3s1         Up 45 seconds   
```
Terbukti tanpa sihir: ke-3 komputer server Kubernetes tersebut ditambah 1 LoadBalancer-nya ternyata hanyalah **4 buah container Docker yang sedang berjalan di atas OrbStack**!

---

## 4. Menjalankan Aplikasi & Trik Emas `k3d image import`

Meskipun kita baru akan mendalami Kubernetes secara menyeluruh di **Fase 2**, mari kita cicipi betapa mudahnya menyebarkan aplikasi ke atas 3 node cluster yang baru saja kita bangun.

### 1. Men-deploy dan Melakukan *Scaling* Aplikasi
```bash
# 1. Perintahkan Kubernetes menjalankan aplikasi web nginx
kubectl create deployment web-pertama --image=nginx:1.27-alpine

# 2. Gandakan jumlah aplikasinya menjadi 4 buah (Replicas = 4)
kubectl scale deployment web-pertama --replicas=4

# 3. Lihat bagaimana Kubernetes menyebarkan ke-4 Pod tersebut secara merata ke node agent-0 dan agent-1!
kubectl get pods -o wide
```

### 2. Mengakses Aplikasi Secara Cepat dengan `kubectl port-forward`
Untuk mengetes apakah aplikasi di dalam cluster berjalan sehat tanpa perlu membuat konfigurasi Ingress yang panjang, gunakan jalan tol sementara bernama **`port-forward`**:

```bash
# Buat jembatan langsung dari Port 8081 di Mac menuju Port 80 milik Deployment web-pertama:
kubectl port-forward deployment/web-pertama 8081:80
```
Selama perintah tersebut berjalan, buka tab terminal baru dan ketik `curl -I http://localhost:8081` (atau buka di browser). Tekan `Ctrl + C` untuk menutup jembatan tersebut.

---

### 3. Trik Rahasia SRE: `k3d image import` (Uji Image Lokal Tanpa Push ke Internet!)

Ini adalah masalah yang **pasti** akan kamu alami saat latihan:
1. Kamu membuat image sendiri di laptop: `docker build -t aplikasiku:v1 .`.
2. Image `aplikasiku:v1` ada di daftar `docker images` laptopmu.
3. Lalu kamu mengetik: `kubectl create deployment tes --image=aplikasiku:v1`.
4. **Tiba-tiba Pod gagal menyala dengan status `ErrImagePull` / `ImagePullBackOff`!**

**Mengapa bisa gagal padahal image-nya ada di `docker images` laptop kita?**
Karena setiap node di dalam `k3d` (`k3d-lab-server-0`, `agent-0`) memiliki **ruang penyimpanan container internal (`containerd`) sendiri yang terpisah dari penyimpanan Docker laptopmu**! Saat kamu menyuruh Kubernetes menjalankan `aplikasiku:v1`, node `k3d` mencarinya ke internet (Docker Hub), dan tentu saja tidak ditemukan!

**Solusi Cepat Tanpa Perlu `docker push` ke Registry:**
Gunakan perintah **`k3d image import`** untuk menyuntikkan image dari Docker lokal langsung ke dalam perut cluster `k3d`:

```bash
# Suntikkan image lokal 'aplikasiku:v1' ke dalam cluster bernama 'lab':
k3d image import aplikasiku:v1 -c lab
```
Dalam 2 detik, image lokalmu langsung tersedia di seluruh node cluster `lab` tanpa memakan kuota internet!

---

## 5. Matriks Keputusan SRE: Kapan Pakai `k3d` vs `k3s` di VM OrbStack?

Di sepanjang Bootcamp SRE 2026 ini, kita menggunakan dua jenis lingkungan Kubernetes. Memahami kapan harus menggunakan yang mana adalah keterampilan arsitektural yang sangat penting:

| Kebutuhan / Skenario | Gunakan **`k3d` (Container)** | Gunakan **`k3s` di OrbStack Machine (VM)** |
|---|---|---|
| **Kecepatan Pembuatan Cluster** | ⚡ **Sangat Cepat (~10 detik)** | 🐢 Sedang (~1–2 menit via skrip/Ansible) |
| **Konsumsi RAM & Baterai** | 🍃 **Sangat Hemat** | 🔋 Lebih besar (karena menjalankan OS penuh) |
| **Belajar Dasar Pod, Deployment, Service, ConfigMap, & Helm** | ✅ **Pilihan Terbaik!** (Modul 2.1 & Fase 5) | Bisa, tapi terlalu berat untuk sekadar latihan sintaks harian |
| **Latihan Otomasi Instalasi Cluster via SSH & Ansible** | ❌ Tidak bisa (karena bukan VM ber-systemd) | ✅ **Wajib!** (Modul 2.2 & Fase 4) |
| **Simulasi Bare-Metal LoadBalancer dengan MetalLB (L2 ARP/BGP)** | ⚠️ Sulit & tidak stabil (terhalang jaringan NAT container Docker) | ✅ **Wajib & 100% Akurat!** (Modul 2.3 — menggunakan IP stabil VM OrbStack) |
| **Simulasi Insiden Node Mati / Kernel Tuning (`sysctl`) / Etcd Backup** | ❌ Terbatas | ✅ **Wajib!** (Fase 8 & Fase 9 Capstone) |

> 📌 **Aturan Praktis Bootcamp:**
> - Gunakan **`k3d`** sebagai **"Papan Tulis Coret-Coretan Harian"** saat kamu sedang merancang manifes YAML, mencoba chart Helm, atau mengetes pipeline CI. Jangan ragu menghapusnya setiap selesai latihan (`k3d cluster delete`).
> - Gunakan **`k3s` di atas OrbStack Linux Machine** sebagai **"Lingkungan Simulasi Production"** saat kita mempraktikkan otomasi infrastruktur, jaringan bare-metal MetalLB, dan Game Day insiden.

---

## 6. Hands-On Mandiri Step-by-Step (20 Menit)

Mari kita jalankan siklus hidup lengkap pembuatan cluster `k3d`, penyuntikan image lokal, hingga mengekspos layanan melalui Ingress Traefik:

```bash
# --- LANGKAH 1: Bersihkan Cluster Lama (Jika Ada) & Buat Cluster Multi-Node Baru ---
k3d cluster delete sre-lab 2>/dev/null || true

k3d cluster create sre-lab \
  --servers 1 \
  --agents 2 \
  --port "8080:80@loadbalancer"

# Verifikasi ke-3 node berstatus Ready:
kubectl get nodes

# --- LANGKAH 2: Buat Image Aplikasi Lokal & Suntikkan dengan 'k3d image import' ---
cd ~
mkdir -p uji-k3d-lokal && cd uji-k3d-lokal

cat << 'EOF' > index.html
<h1>Selamat! Web ini berjalan di atas Cluster Multi-Node k3d pada OrbStack!</h1>
EOF

cat << 'EOF' > Dockerfile
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
EOF

# Build image di Docker lokal (tanpa push ke registry mana pun!):
docker build -t web-lokal-sre:v1 .

# Suntikkan image lokal tersebut langsung ke dalam cluster 'sre-lab':
k3d image import web-lokal-sre:v1 -c sre-lab

# --- LANGKAH 3: Deploy Aplikasi & Buat Service + Ingress ---
kubectl create deployment sapa-k3d --image=web-lokal-sre:v1 --replicas=3
kubectl expose deployment sapa-k3d --port=80 --target-port=80

# Buat aturan Ingress agar Traefik meneruskan trafik dari port 8080 ke service sapa-k3d:
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-sapa
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sapa-k3d
            port:
              number: 80
EOF

# Tunggu hingga semua Pod berstatus Running:
kubectl rollout status deployment/sapa-k3d
kubectl get pods -o wide

# --- LANGKAH 4: Uji Akses dari Terminal Mac melalui Port 8080 LoadBalancer ---
sleep 3
curl http://localhost:8080/

# --- LANGKAH 5: Bersihkan Folder & Hentikan/Hapus Cluster ---
cd ~ && rm -rf uji-k3d-lokal
# (Jika sudah selesai mengerjakan LAB-01 modul ini, kamu bisa menghapusnya dengan: k3d cluster delete sre-lab)
```

---

## 7. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 masalah yang paling sering dialami saat mengoperasikan `k3d` beserta langkah perbaikannya:

### Kasus 1: Gagal Membuat Cluster karena `Bind for 0.0.0.0:8080 failed: port is already allocated`
- **Gejala:** Proses `k3d cluster create` gagal di tengah jalan dengan keluhan bahwa port `8080` sudah dipakai.
- **Penyebab:** Ada container Docker lain, atau cluster `k3d` sebelumnya yang masih menyala dan sedang menduduki port `8080` di laptop Mac milikmu.
- **Diagnosa & Solusi:**
  ```bash
  # 1. Cek container apa yang sedang memakai port 8080:
  docker ps --filter "publish=8080"

  # 2. Hentikan container tersebut atau gunakan nomor port lain di sisi Mac (misal 8088:80@loadbalancer):
  k3d cluster create lab --port "8088:80@loadbalancer"
  ```

---

### Kasus 2: Error `The connection to the server 127.0.0.1:xxxxx was refused` Saat Mengetik `kubectl`
- **Gejala:** Kemarin `kubectl get nodes` berjalan lancar, namun hari ini setelah laptop dinyalakan ulang, perintah `kubectl` gagal terhubung ke server.
- **Penyebab:**
  1. Container cluster `k3d` sedang dalam posisi berhenti (`stopped`), atau
  2. File `~/.kube/config` kamu sedang menunjuk ke *context* cluster lama yang sudah dihapus.
- **Diagnosa & Solusi:**
  ```bash
  # 1. Cek daftar cluster k3d dan nyalakan jika mati:
  k3d cluster list
  k3d cluster start <nama-cluster>

  # 2. Perbarui dan arahkan ulang kunci kubeconfig ke cluster tersebut:
  k3d kubeconfig merge <nama-cluster> --kubeconfig-switch-context
  ```

---

### Kasus 3: Pod Berstatus `ErrImagePull` atau `ImagePullBackOff` pada Image Buatan Sendiri
- **Gejala:** Kamu men-deploy image `aplikasiku:v1` yang baru di-build di lokal, tetapi Pod tidak kunjung `Running` dan malah berstatus `ImagePullBackOff`.
- **Penyebab:** Kamu lupa menjalankan `k3d image import`, atau kamu tidak mengatur `imagePullPolicy: IfNotPresent` ketika memakai tag `:latest` (sehingga Kubernetes selalu memaksa mencari ke internet).
- **Solusi SRE:**
  1. Selalu gunakan tag versi yang jelas (jangan `:latest`, gunakan `:v1`).
  2. Jalankan `k3d image import aplikasiku:v1 -c <nama-cluster>` setiap kali selesai melakukan `docker build` lokal.

---

## 8. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Perintah `k3d` & `kubectl` (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Buat cluster 1 server + 2 worker + LB** | `k3d cluster create <nama> --servers 1 --agents 2 -p "8080:80@loadbalancer"` |
| **Lihat daftar cluster k3d** | `k3d cluster list` |
| **Matikan / Nyalakan sementara cluster**| `k3d cluster stop <nama>` / `k3d cluster start <nama>` |
| **Hapus cluster sampai bersih** | `k3d cluster delete <nama>` |
| **Suntikkan image Docker lokal ke k3d** | `k3d image import <nama-image>:<tag> -c <nama-cluster>` |
| **Perbaiki konteks kubeconfig yang nyangkut**| `k3d kubeconfig merge <nama> --kubeconfig-switch-context` |
| **Cek daftar node di dalam cluster** | `kubectl get nodes -o wide` |
| **Cek seluruh Pod di semua namespace** | `kubectl get pods -A` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk memastikan kesiapanmu sebelum mengerjakan Lab Modul 1.2 dan melangkah ke Fase 2 (Kubernetes):

1. **Jelaskan dengan kalimatmu sendiri hubungan dan perbedaan antara `k8s`, `k3s`, dan `k3d`!**
2. **Ketika kamu membuat cluster dengan `--servers 1 --agents 2`, mengapa saat kamu mengetik `docker ps` justru terlihat ada 4 buah container yang berjalan? Apa tugas dari container ke-4 tersebut?**
3. **Mengapa sebuah image yang baru saja kamu buat dengan `docker build -t tes:v1 .` tidak bisa langsung ditemukan oleh cluster `k3d` tanpa di-push ke registry, dan perintah `k3d` apa yang menjadi jalan pintas solusinya?**
4. **Pada Fase 2 nanti kita akan mempraktikkan pemasangan LoadBalancer bare-metal *MetalLB* menggunakan protokol ARP Layer 2 dan mengotomasi instalasi server via *Ansible*. Mengapa untuk skenario tersebut kita harus beralih menggunakan `k3s` di atas *OrbStack Linux Machine* dan tidak menggunakan `k3d`?**

---
*Selamat! Kamu telah menguasai seluruh materi teori di Fase 1. Lanjutkan ke laboratorium praktik: [lab/LAB-01-k3d-cluster.md](lab/LAB-01-k3d-cluster.md) dan selesaikan evaluasi modul untuk bersiap memasuki dunia orkestrasi sesungguhnya di **Fase 2 — Kubernetes**!*