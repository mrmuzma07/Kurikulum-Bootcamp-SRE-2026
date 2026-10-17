# 03 — k3d: Jalur Cepat (Fast Lane) Latihan Harian Kubernetes

> **Ringkasan:** Memanfaatkan **k3d** sebagai arena laboratorium harian yang cepat, ringan, dan dapat dihancurkan berulang kali (*ephemeral*): menguasai pembuatan cluster multi-node dalam hitungan detik, memahami aliran port mapping **`@loadbalancer`**, mengelola banyak konteks cluster di dalam **Kubeconfig** tanpa salah sasaran, menyuntikkan image lokal dengan **`k3d image import`**, serta menerapkan aturan tegas kapan harus memakai `k3d` vs `k3s` di atas VM.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan menyelesaikan simulasinya di terminal MacBook, kamu akan mampu:
- Menjelaskan filosofi *Fast Lane* dan mengapa ketangkasan membuat dan menghancurkan cluster (*cluster lifecycle*) sangat penting bagi seorang SRE.
- Mengoperasikan perintah `k3d` untuk membuat (*create*), menghentikan (*stop*), menyalakan kembali (*start*), dan menghapus (*delete*) cluster dengan bersih.
- Menjelaskan arsitektur aliran lalu lintas port mapping **`@loadbalancer`** dari peramban host Mac menuju Ingress Controller Traefik.
- Mengelola banyak konteks cluster di dalam file **`~/.kube/config`** serta berpindah antar-cluster dengan aman tanpa risiko salah tembak.
- Menghindari jebakan `ImagePullBackOff` pada image buatan sendiri menggunakan perintah **`k3d image import`**.
- Menyiapkan kredensial **`imagePullSecrets`** untuk menarik image dari repositori privat GitLab Container Registry.
- Membedakan secara tegas kapan saatnya menggunakan `k3d` dan kapan saatnya beralih ke `k3s` di atas VM OrbStack.

---

## 1. Konsep & Filosofi: Mengapa k3d Menjadi "Fast Lane" SRE?

### Menghilangkan Sindrom "Takut Mencoba" pada Pemula
Banyak pemula yang belajar Kubernetes merasa cemas dan ragu saat ingin mencoba perintah baru atau bereksperimen dengan file YAML:
*"Bagaimana kalau saya salah ketik lalu cluster-nya rusak dan tidak bisa kembali lagi?"*
Kekhawatiran ini sangat wajar jika kamu membangun cluster di server fisik data center atau di cloud publik (yang mahal dan butuh waktu lama untuk dibuat ulang).

Namun, seorang **Site Reliability Engineer (SRE)** yang handal dibangun dari **keberanian untuk merusak sistem (*break things*) dan kemahiran memulihkannya kembali**.
Di sinilah **`k3d`** berperan sebagai **Jalur Cepat (Fast Lane)** latihan harian kita:
1. **Kecepatan Ekstrem:** Membangun cluster 3-node (1 Server + 2 Agent) hanya butuh waktu **~10 detik**.
2. **Hemat Memori:** Node berjalan sebagai container Docker di OrbStack, memakan RAM sangat sedikit.
3. **Bersih Total Tanpa Sisa:** Begitu selesai latihan, ketik `k3d cluster delete` dan seluruh jejak cluster musnah dari harddisk dalam 3 detik tanpa mengotori laptop Mac milikmu!

```text
  SIKLUS BELAJAR SRE DI K3D (FAST LANE):
 ┌────────────────────────────────────────────────────────┐
 │ 1. Buat cluster baru dalam 10 detik (k3d cluster create)│
 └──────────────────────────┬─────────────────────────────┘
                            ▼
 ┌────────────────────────────────────────────────────────┐
 │ 2. Uji coba manifes YAML, pasang Helm, rusak sistem!   │
 └──────────────────────────┬─────────────────────────────┘
                            ▼
 ┌────────────────────────────────────────────────────────┐
 │ 3. Hapus sampai bersih total! (k3d cluster delete)     │
 └────────────────────────────────────────────────────────┘
```

---

## 2. Siklus Hidup Cluster `k3d`: Create, Stop, Start, dan Delete

### 1. Membuat Cluster Multi-Node yang Ideal untuk Latihan
Di modul-modul mendatang, kita akan sering menguji apakah aplikasi kita terdistribusi secara seimbang ke beberapa server. Oleh karena itu, kita selalu membuat cluster dengan **1 Node Server (Control Plane)** dan **2 Node Agent (Worker)**:

```bash
k3d cluster create lab \
  --servers 1 \
  --agents 2 \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer"
```

#### Membedah Bendera (*Flags*) Kunci:
- **`--servers 1`**: Menentukan jumlah node master. (Di k3d, jika kamu menulis `--servers 3`, k3d otomatis mengaktifkan mode High Availability dengan database etcd terdistribusi).
- **`--agents 2`**: Menentukan jumlah node pekerja yang akan menampung Pod aplikasi kita.
- **`--port "8080:80@loadbalancer"`**: Memetakan port laptop Mac ke LoadBalancer k3d (akan kita bedah detail di Bagian 3).
- **`--k3s-arg` (Opsional Lanjutan)**: Menambahkan argumen khusus ke mesin k3s, misalnya menonaktifkan Traefik bawaan jika ingin menguji Nginx Ingress: `--k3s-arg "--disable=traefik@server:0"`.

---

### 2. Membedakan `k3d cluster stop` vs `k3d cluster delete`
Banyak pemula tidak sengaja menghapus cluster padahal hanya ingin istirahat makan siang:

| Perintah | Apa yang Terjadi pada Container & Data? | Kapan Digunakan? |
|---|---|---|
| **`k3d cluster stop <nama>`** | Container k3s hanya **dihentikan sementara (*paused/stopped*)**. Seluruh file, database etcd, dan Pod di dalamnya **TETAP AMAN**. Memori RAM Mac langsung dilepas dan kembali lega! | Saat kamu ingin mengistirahatkan laptop atau mengerjakan hal lain, dan ingin melanjutkan lab besok pagi. |
| **`k3d cluster start <nama>`** | Menyalakan kembali container cluster yang sedang tidur. Dalam 5 detik seluruh Pod kembali berstatus `Running`. | Saat kamu ingin melanjutkan latihan dari kondisi terakhir. |
| **`k3d cluster delete <nama>`** | **MEMUSNAHKAN TOTAL** seluruh container, volume data, dan catatan context Kubeconfig. Data tidak bisa dipulihkan kembali! | Saat modul lab sudah tuntas, atau cluster rusak parah dan kamu ingin memulai dari lembaran bersih baru. |

```bash
# Menghentikan sementara cluster agar hemat baterai laptop:
k3d cluster stop lab

# Menyalakan kembali:
k3d cluster start lab

# Menghapus total cluster setelah selesai evaluasi:
k3d cluster delete lab
```

---

## 3. Membedah Port Mapping `@loadbalancer` & Akses dari Mac

Bagaimana alur komunikasi paket data ketika kamu membuka browser di laptop Mac dan mengakses aplikasi di dalam cluster `k3d`?

### Anatomi Rute Jaringan Tiga Tingkat

```text
 1. Browser Mac / curl ➔ Mengakses http://localhost:8080
    │
    ▼ (Tembus Port Mapping Docker)
 2. Container 'k3d-lab-serverlb' (Pintu Gerbang Nginx Proxy)
    Menerima di Port 80, lalu meneruskan ke alamat IP Node Worker di Port 80
    │
    ▼ (Tembus ke Jaringan Cluster)
 3. Pod Traefik Ingress Controller (Mendengarkan di Port 80)
    Membaca Host Header (misal: Host: app.k3d.local)
    │
    ▼ (Meneruskan ke Service)
 4. Kubernetes Service ➔ Pod Aplikasi (Port 8080)
```

Ketika kamu menyertakan parameter `--port "8080:80@loadbalancer"`, k3d secara otomatis membuat satu container khusus bernama **`k3d-<nama-cluster>-serverlb`**:
- Huruf **`8080`** di sebelah kiri adalah port yang dibuka di **laptop Mac milikmu**.
- Huruf **`80`** di sebelah kanan adalah port yang didengarkan oleh **Traefik Ingress** di dalam cluster.
- Tulisan **`@loadbalancer`** memberi tahu k3d agar port forwarding ini dipasang pada container proxy gerbang depan (`serverlb`), bukan langsung ke node server k3s!

---

### Mensimulasikan Nama Domain Perusahaan (`.k3d.local`)
Agar latihanmu mencerminkan production nyata yang menggunakan nama domain (bukan angka IP kaku), daftarkan nama domain lokalmu ke dalam file resolver `/etc/hosts` di macOS:

```bash
# Daftarkan nama domain lokal ke alamat loopback 127.0.0.1 (hanya butuh sekali):
echo "127.0.0.1 app.k3d.local api.k3d.local portal.k3d.local" | sudo tee -a /etc/hosts
```
Sekarang, jika kamu memasang Ingress dengan aturan `host: app.k3d.local`, kamu cukup membuka URL **`http://app.k3d.local:8080`** langsung dari browser Safari atau Chrome di Mac milikmu!

---

## 4. Manajemen Banyak Kubeconfig (Multi-Cluster Workflow)

Di dunia kerja SRE, kamu tidak hanya mengelola satu cluster. Kamu mungkin terhubung ke cluster *Development*, cluster *Staging*, dan cluster *Production*.

Seluruh kunci akses, alamat server, dan sertifikat otentikasi disimpan di dalam satu file pusat: **`~/.kube/config`**.

```text
 File ~/.kube/config:
 ├── Clusters:    [ k3d-lab, k3d-sandbox, k3s-production-dc1 ]
 ├── Users:       [ admin-lab, admin-sandbox, sre-oncall ]
 └── Contexts:    [ k3d-lab, k3d-sandbox, k3s-prod ] ◄── Pasangan Cluster + User
                               ▲
               (Current Context yang sedang aktif!)
```

### Memahami Konsep "Context"
**Context** adalah tombol saklar penunjuk yang menentukan: *"Ketika saya mengetik perintah `kubectl`, ke cluster mana perintah ini akan dikirimkan?"*

```bash
# 1. Melihat daftar seluruh context cluster yang terdaftar di komputermu:
kubectl config get-contexts

# 2. Melihat context mana yang saat ini SEDANG AKTIF:
kubectl config current-context

# 3. Berpindah kendali ke cluster lain:
kubectl config use-context k3d-sandbox

# 4. Memperbarui atau memulihkan context k3d jika terhapus:
k3d kubeconfig merge lab --kubeconfig-switch-context
```

> 🚨 **Peringatan Fatal SRE: Selalu Periksa Context Sebelum Mengetik `kubectl delete`!**
> Skenario mimpi buruk SRE: Kamu mengira sedang berada di cluster latihan `k3d-lab`, lalu dengan santai mengetik `kubectl delete -f manifests/`. Ternyata terminalmu sedang menunjuk ke context cluster **`k3s-production`** data center perusahaan! Seluruh database dan aplikasi production musnah seketika.
> **Kebiasaan Baik SRE:** Pasang indikator nama context di prompt terminalmu (menggunakan tema shell seperti Starship, Oh-My-Zsh, atau alat seperti `kubectx` dan `kubens`).

---

## 5. Manajemen Image di k3d: Trik Cepat `image import` & Private Registry

### Mengapa Node k3d Tidak Mengenali `docker images` Lokal?
Sering kali peserta bootcamp mengalami kebingungan seperti ini:
1. Mengetik `docker build -t frontend:v1 .` di terminal Mac.
2. Mengetik `docker images` di Mac ➔ image `frontend:v1` jelas-jelas terlihat.
3. Menjalankan `kubectl create deployment web --image=frontend:v1`.
4. **Hasilnya: Pod berstatus `ErrImagePull` atau `ImagePullBackOff`!**

**Mengapa ini terjadi?**
Ingat bahwa node-node k3d (`k3d-lab-server-0`, `k3d-lab-agent-0`) adalah container yang memiliki **ruang penyimpanan runtime `containerd` sendiri di dalamnya**.
Node k3d **tidak bisa melihat** daftar image yang ada di Docker desktop Mac milikmu! Saat diminta menjalankan `frontend:v1`, node k3d mencarinya ke internet (Docker Hub) dan gagal menemukannya.

---

### Solusi 1: Trik Cepat Lokal — `k3d image import` (Tanpa Kuota Internet!)
Jika kamu hanya ingin menguji kode aplikasi baru secara cepat di laptop tanpa perlu mengunggahnya (*push*) ke internet:

```bash
# Suntikkan image dari Docker Mac langsung ke dalam perut cluster k3d:
k3d image import frontend:v1 -c lab
```
Perintah ini menyalin lapisan image langsung ke dalam *storage* `containerd` milik seluruh node k3d. Dalam 2 detik, Pod akan langsung berubah status menjadi `Running`!

---

### Solusi 2: Standar Production — Menggunakan GitLab Private Registry (`imagePullSecrets`)
Untuk alur kerja resmi (dan kelak saat menggunakan GitOps di Fase 6), image harus diambil dari repositori privat **GitLab Container Registry**.

Karena repositori GitLab bersifat privat, cluster Kubernetes harus diberi izin otentikasi menggunakan objek **`Secret` bertipe `docker-registry`**:

```bash
# Buat kunci akses penarik image di Kubernetes menggunakan Personal Access Token (PAT):
kubectl create secret docker-registry gitlab-pull-secret \
  --docker-server=registry.gitlab.com \
  --docker-username=<GITLAB_USERNAME> \
  --docker-password=<GITLAB_PAT_READ_REGISTRY> \
  --namespace=default
```

Lalu, di dalam manifes Deployment, panggil kunci rahasia tersebut di bagian `imagePullSecrets`:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
      - name: gitlab-pull-secret      # Kunci otentikasi untuk membuka gudang privat!
      containers:
      - name: app
        image: registry.gitlab.com/budi/sre-bootcamp/app:v1.0.0
```

---

## 6. Matriks Keputusan SRE: k3d (Fast Lane) vs k3s di VM (Production Lane)

Agar kamu tidak bingung di sepanjang 18 minggu kurikulum Bootcamp SRE 2026, berikut adalah panduan arsitektural kapan harus memakai k3d dan kapan wajib memakai k3s di atas VM OrbStack:

```text
                       KAPAN MEMILIH JALUR LAB?
                                  │
         ┌────────────────────────┴────────────────────────┐
         ▼                                                 ▼
   [ Fast Lane: k3d ]                           [ Production Lane: k3s di VM ]
 • Belajar menulis YAML k8s                    • Konfigurasi sistem Linux & sysctl
 • Uji coba Pod, Deployment, Service           • Setup cluster via Ansible Playbook
 • Belajar templating Helm Chart               • Bare-Metal LoadBalancer (MetalLB L2)
 • Uji cepat pipeline CI lokal                 • Simulasi etcd failure & DR snapshot
 • Fase 2.1 & Sebagian Fase 2.4                • Fase 2.2, 2.3, Fase 4, & Capstone
```

| Parameter Perbandingan | **`k3d` (Container di OrbStack)** | **`k3s` di Linux Machine (VM OrbStack)** |
|---|---|---|
| **Waktu Menyalakan Cluster** | ⚡ **~10 Detik** | 🐢 ~1–2 Menit |
| **Kebutuhan Memori RAM** | Sangat Ringan (~1 GB total untuk 3 node) | Lebih besar (~2 GB per VM, butuh 3 VM) |
| **Inisiator Sistem Operasi** | Tidak ada `systemd` di node | **Memiliki `systemd` penuh** layaknya server fisik |
| **Kesesuaian untuk MetalLB L2**| Sulit (karena terhalang bridge Docker) | **100% Sempurna** (menggunakan IP jaringan VM nyata) |
| **Dukungan Otomasi Ansible** | Tidak bisa di-SSH sebagai server biasa | **Bisa di-SSH** dan di-bootstrap dengan Ansible |

---

## 7. Hands-On Mandiri Step-by-Step (20 Menit)

Mari kita latih *muscle memory* mengelola siklus hidup cluster, menyuntikkan image lokal, dan menguji akses multi-node:

```bash
# --- LANGKAH 1: Buat Cluster Multi-Node Bersih ---
k3d cluster delete lab 2>/dev/null || true

k3d cluster create lab \
  --servers 1 \
  --agents 2 \
  --port "8080:80@loadbalancer"

# Periksa kesiapan node:
kubectl get nodes

# --- LANGKAH 2: Menguji Siklus Tidur Cluster (Stop & Start) ---
echo "Menghentikan cluster sejenak..."
k3d cluster stop lab

# Pastikan perintah kubectl gagal terhubung saat cluster tidur:
kubectl get nodes || echo "Bagus! Cluster sedang berhenti dan RAM Mac telah dilepas."

echo "Menyalakan kembali cluster..."
k3d cluster start lab
kubectl get nodes

# --- LANGKAH 3: Eksperimen Image Import Lokal ---
cd ~
mkdir -p demo-import && cd demo-import

cat << 'EOF' > index.html
<h3>Halo! Image ini disuntikkan langsung via k3d image import tanpa lewat internet!</h3>
EOF

cat << 'EOF' > Dockerfile
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
EOF

# Build image lokal di Mac:
docker build -t app-lokal:v1 .

# Suntikkan langsung ke dalam cluster 'lab':
k3d image import app-lokal:v1 -c lab

# Deploy ke cluster:
kubectl create deployment app-lokal --image=app-lokal:v1 --replicas=3
kubectl rollout status deployment/app-lokal
kubectl get pods -o wide

# --- LANGKAH 4: Ekspor via Ingress dan Akses dari Mac ---
kubectl expose deployment app-lokal --port=80 --target-port=80

cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-lokal
spec:
  rules:
  - host: app.k3d.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-lokal
            port:
              number: 80
EOF

# Uji akses HTTP:
sleep 2
curl -H "Host: app.k3d.local" http://localhost:8080/

# --- LANGKAH 5: Bersihkan File Latihan ---
cd ~ && rm -rf demo-import
# (Biarkan cluster 'lab' tetap menyala untuk menyelesaikan latihan materi 04)
```

---

## 8. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 kendala yang paling sering terjadi saat mengoperasikan `k3d` beserta solusinya:

### Kasus 1: Gagal Membuat Cluster karena Port Bentrok (`port is already allocated`)
- **Gejala:** Muncul error `Failed to create cluster ... port 0.0.0.0:8080 is already allocated`.
- **Penyebab:** Ada container Docker lain yang sedang berjalan di Mac dan memakai port 8080.
- **Diagnosa & Solusi:**
  1. Cari tahu container apa yang memakai port tersebut:
     ```bash
     docker ps --filter "publish=8080"
     ```
  2. Hentikan container tersebut (`docker stop <id>`), atau alihkan port k3d ke port lain di sisi Mac:
     ```bash
     k3d cluster create lab --port "8888:80@loadbalancer"
     ```

---

### Kasus 2: Perintah `kubectl` Nyasar ke Cluster yang Salah
- **Gejala:** Kamu baru saja membuat cluster `k3d-lab`, tetapi ketika mengetik `kubectl get pods`, yang muncul adalah Pod dari cluster pengujian kemarin.
- **Penyebab:** Context aktif di file `~/.kube/config` belum berpindah ke cluster baru.
- **Solusi:**
  ```bash
  # Paksa beralih ke context cluster lab:
  k3d kubeconfig merge lab --kubeconfig-switch-context
  # Verifikasi context yang aktif:
  kubectl config current-context
  ```

---

### Kasus 3: Node Agent Tidak Kunjung Masuk Status `Ready`
- **Gejala:** Saat `kubectl get nodes`, node `k3d-lab-agent-0` berstatus `NotReady` selama bermenit-menit.
- **Penyebab:** Mesin OrbStack kehabisan alokasi RAM atau beban CPU Mac sedang mencapai 100%.
- **Diagnosa & Solusi:**
  Periksa log container agent tersebut dari sisi Docker:
  ```bash
  docker logs k3d-lab-agent-0 --tail 50
  ```
  Jika ditemukan pesan *out of memory*, naikkan memori OrbStack di `orb config set memory_mib` atau kurangi jumlah agen.

---

## 9. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Perintah `k3d` (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Buat cluster multi-node lengkap** | `k3d cluster create <nama> --servers 1 --agents 2 -p "8080:80@loadbalancer"` |
| **Daftar seluruh cluster k3d** | `k3d cluster list` |
| **Tidurkan cluster (hemat baterai/RAM)**| `k3d cluster stop <nama>` |
| **Bangunkan cluster kembali** | `k3d cluster start <nama>` |
| **Hancurkan cluster total** | `k3d cluster delete <nama>` |
| **Suntikkan image lokal ke cluster** | `k3d image import <image>:<tag> -c <nama>` |
| **Arahkan context kubectl ke cluster** | `k3d kubeconfig merge <nama> --kubeconfig-switch-context` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk mengevaluasi kesiapan operasional k3d milikmu:

1. **Jelaskan perbedaan mendasar antara `k3d cluster stop` dan `k3d cluster delete`! Kapan kamu harus memilih stop dan kapan harus memilih delete?**
2. **Apa fungsi teknis dari container `k3d-<nama>-serverlb` di dalam cluster multi-node? Mengapa kita mengarahkan port mapping Mac ke container tersebut dan bukan langsung ke container `k3d-server-0`?**
3. **Kamu baru saja mengetik `docker build -t backend:v2 .` di terminal MacBook. Mengapa Pod di dalam cluster k3d gagal menarik image tersebut jika kamu tidak menjalankan `k3d image import` atau tidak mengunggahnya ke GitLab Registry?**
4. **Apa risiko operasional terbesar dari memiliki banyak context cluster di dalam file `~/.kube/config`, dan kebiasaan baik apa yang harus dilakukan sebelum menjalankan perintah perubahan (`apply` / `delete`)?**

---
*Lanjutkan ke materi berikutnya: [04-kubectl-survival-kit.md](04-kubectl-survival-kit.md) untuk mempelajari senjata harian SRE di terminal: perintah investigasi cepat, alur diagnosa insiden pod, serta trik container debugging!*