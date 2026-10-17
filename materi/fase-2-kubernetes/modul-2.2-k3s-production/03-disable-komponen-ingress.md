# 03 — Menonaktifkan Komponen Bawaan k3s & Arsitektur Ingress Controller

> **Ringkasan:** Memahami filosofi *Batteries Included, But Removable* pada k3s: menonaktifkan komponen bawaan yang tidak dibutuhkan (**`--disable servicelb`** dan **`--disable traefik`**), mencegah konflik benturan port dengan Load Balancer bare-metal **MetalLB**, membandingkan kelebihan dan kekurangan antara **Traefik**, **Ingress-Nginx**, dan **HAProxy**, memahami objek **`IngressClass`**, serta memasang Ingress Controller alternatif di lingkungan on-premises.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan menyelesaikan praktiknya di terminal, kamu akan mampu:
- Mengidentifikasi seluruh komponen sistem bawaan yang otomatis dipasang oleh instalasi standar k3s di namespace `kube-system`.
- Menjelaskan cara kerja **ServiceLB (Klipper-LB)** bawaan k3s dan alasan teknis mengapa komponen ini **WAJIB dinonaktifkan** sebelum memasang **MetalLB** di Modul 2.3.
- Menggunakan parameter instalasi **`--disable`** untuk mematikan komponen tertentu secara modular (*declarative provisioning*).
- Memverifikasi keberhasilan penonaktifan komponen menggunakan pemeriksaan status Service berstatus `<pending>`.
- Membedakan karakteristik teknis antara **Traefik Ingress** (bawaan k3s) dan **Ingress-Nginx Controller** (standar industri enterprise).
- Mengonfigurasi objek **`IngressClass`** untuk mengarahkan aturan Ingress ke controller yang tepat saat terdapat lebih dari satu penyedia di cluster.

---

## 1. Konsep & Arsitektur: Filosofi *Batteries Included, But Removable*

Secara bawaan, tim pengembang Rancher k3s ingin agar pengguna yang baru menginstal k3s bisa langsung menjalankan aplikasi dan membukanya di browser tanpa perlu menginstal plugin tambahan apa pun. Oleh karena itu, k3s datang dengan konsep **"Sudah Termasuk Baterai" (*Batteries Included*)**.

Mari kita periksa komponen apa saja yang otomatis dinyalakan oleh k3s:

```bash
kubectl get pods,svc,deploy -n kube-system
```

| Komponen Bawaan | Apa Tugas Sebenarnya? | Status Bawaan | Rekomendasi di Bootcamp SRE On-Prem |
|---|---|---|---|
| **CoreDNS** | Menyediakan resolusi nama domain internal cluster (`<svc>.<ns>.svc.cluster.local`). | Aktif | **Wajib Tetap Aktif** (Jantung komunikasi internal Pod). |
| **metrics-server** | Mengumpulkan metrik penggunaan CPU dan RAM Pod/Node untuk `kubectl top`. | Aktif | **Tetap Aktif** (Pondasi autoscaling HPA). |
| **local-path-provisioner** | Otomatis membuat PersistentVolume (PV) di harddisk lokal node saat ada PVC. | Aktif | **Tetap Aktif** (Penyimpanan dinamis bawaan). |
| **Traefik** | Ingress Controller Layer 7 HTTP/HTTPS bawaan k3s. | Aktif | **Bisa Dinonaktifkan** jika tim ingin menggunakan Ingress-Nginx. |
| **ServiceLB (Klipper-LB)** | Memberikan Alamat IP untuk Service bertipe `LoadBalancer` di on-prem. | Aktif | 🚨 **WAJIB DINONAKTIFKAN!** (Akan digantikan oleh **MetalLB** di Modul 2.3). |

---

## 2. Mengapa ServiceLB (Klipper) Wajib Dimatikan Sebelum MetalLB?

Ini adalah salah satu pertanyaan paling penting dalam arsitektur Kubernetes On-Premises:
*"Kubernetes sudah punya ServiceLB bawaan untuk melayani `type: LoadBalancer`. Kenapa kita repot-repot mematikannya dan menggantinya dengan MetalLB?"*

### Bagaimana Cara Kerja ServiceLB (Klipper-LB) Bawaan?
Di lingkungan cloud (seperti AWS atau GCP), saat kamu membuat Service `type: LoadBalancer`, penyedia cloud akan membuatkan perangkat fisik Load Balancer nyata di luar cluster dan memberikan Alamat IP Publik independen.

Di lingkungan bare-metal/on-prem biasa, tidak ada cloud provider yang menyediakan hardware tersebut. Maka k3s membuat trik cerdas bernama **ServiceLB (Klipper)**:
1. ServiceLB membuat sebuah Pod pembantu (*DaemonSet*) di setiap node worker.
2. Pod pembantu ini menyamar sebagai Load Balancer dengan cara **membajak port node fisik secara langsung (`hostPort`)** dan meminjam Alamat IP milik node fisik tersebut!
3. Akibatnya: Service `type: LoadBalancer` milikmu sebenarnya **hanya menumpang pada IP fisik server host**, bukan memiliki IP Virtual (VIP) mandiri.

```text
 CARA KERJA SERVICELB (KLIPPER) BAWAAN:
 Service type: LoadBalancer ──► Membajak Port 80 di IP Node k3s-cp1 (198.19.249.10)
                               (Tidak ada Alamat IP Virtual baru yang tercipta!)

 CARA KERJA METALLB (STANDAR PRODUCTION ENTERPRISE - MODUL 2.3):
 Service type: LoadBalancer ──► Menerima Alamat IP Virtual Mandiri dari Pool Khusus:
                                Contoh: 198.19.249.200 (Bukan IP Node mana pun!)
```

### Bahaya Bentrokan ServiceLB vs MetalLB:
Jika ServiceLB k3s tidak dimatikan, maka saat kamu memasang **MetalLB** di Modul 2.3:
- ServiceLB akan langsung menyambar setiap pembuatan Service `type: LoadBalancer` dan menempelkan IP node fisik.
- MetalLB juga akan mencoba mengalokasikan IP dari pool khususnya.
- **Keduanya akan berebut kendali jaringan (*race condition*)**, menyebabkan tabel perutean ARP bentrok, dan trafik jaringan dari luar cluster menjadi putus-nyambung (*flapping*)!
- **Aturan Tegas SRE:** Matikan ServiceLB sejak hari pertama sebelum MetalLB dipasang!

---

## 3. Cara Menonaktifkan Komponen Bawaan

Ada dua metode untuk menonaktifkan komponen di k3s:

### Metode 1: Melalui Flag Perintah Instalasi (Paling Bersih & Direkomendasikan)
Saat mengeksekusi skrip instalasi k3s, sertakan flag **`--disable <nama-komponen>`** (bendera ini bisa ditulis berulang-ulang):

```bash
# Contoh: Menginstal k3s Master dengan mematikan ServiceLB dan Traefik sekaligus:
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --disable servicelb \
  --disable traefik
```

---

### Metode 2: Menonaktifkan pada Cluster yang Sudah Terlanjur Terinstal
Bagaimana jika cluster k3s kamu saat ini sudah terlanjur berjalan dengan Traefik dan ServiceLB aktif, dan kamu tidak ingin menghapus database cluster?

Kamu bisa membuat file konfigurasi deklaratif k3s di dalam VM:

```bash
# 1. Masuk ke dalam VM:
ssh k3s-cp1

# 2. Buat direktori konfigurasi jika belum ada:
sudo mkdir -p /etc/rancher/k3s

# 3. Tulis daftar komponen yang ingin dimatikan di /etc/rancher/k3s/config.yaml:
sudo tee /etc/rancher/k3s/config.yaml << 'EOF'
disable:
  - servicelb
  - traefik
EOF

# 4. Restart daemon k3s agar konfigurasi baru dimuat:
sudo systemctl restart k3s
```

Begitu k3s menyala kembali, k3s akan membaca file tersebut, menghapus manifest Traefik dan ServiceLB dari internal cluster, serta mencopot seluruh Pod terkait secara otomatis!

---

## 4. Memverifikasi Keberhasilan Penonaktifan Komponen

Bagaimana seorang SRE membuktikan bahwa komponen yang dimatikan benar-benar telah lenyap dari muka bumi cluster?

### Uji 1: Periksa Daftar Pod di Namespace `kube-system`
```bash
kubectl get pods -n kube-system
```
Pastikan **TIDAK ADA LAGI** Pod yang bernama:
- `svclb-*` (DaemonSet ServiceLB klipper).
- `traefik-*` atau `helm-install-traefik-*`.

### Uji 2: Tes Pembuktian Status `<pending>` pada Service LoadBalancer
Ini adalah pembuktian paling otentik bahwa ServiceLB telah mati dan cluster siap menyambut MetalLB:

```bash
# 1. Buat deployment uji coba:
kubectl create deployment tes-lb --image=nginx:1.27-alpine

# 2. Buat Service bertipe LoadBalancer:
kubectl expose deployment tes-lb --port=80 --type=LoadBalancer

# 3. Periksa status Service:
kubectl get svc tes-lb
```

*Output yang HARUS muncul:*
```text
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
tes-lb    LoadBalancer   10.43.150.200   <pending>     80:31234/TCP   10s
```

> 🎯 **Mengapa Status `<pending>` Justru Adalah Kabar Gembira?**
> Perhatikan kolom **`EXTERNAL-IP: <pending>`**!
> - Jika ServiceLB masih aktif, kolom tersebut akan langsung terisi IP node fisik VM secara lancang.
> - Status **`<pending>`** membuktikan bahwa k3s kini patuh dan sedang berkata: *"Saya tidak punya Load Balancer bawaan lagi, saya sedang menunggu pengendali eksternal (MetalLB di Modul 2.3) untuk memberi saya Alamat IP Virtual!"*

```bash
# Bersihkan service dan deployment tes:
kubectl delete svc tes-lb
kubectl delete deployment tes-lb
```

---

## 5. Menimbang Pilihan Ingress Controller: Traefik vs Ingress-Nginx vs HAProxy

Ingress Controller adalah pintu gerbang lalu lintas aplikasi web kita. Mengapa banyak perusahaan mengganti Traefik bawaan dengan Ingress-Nginx?

| Parameter Penilaian | **Traefik Ingress** (Bawaan k3s) | **Ingress-Nginx Controller** (Kubernetes Community) | **HAProxy Ingress** |
|---|---|---|---|
| **Penyedia & Status** | Dibuat oleh Traefik Labs; bawaan default di k3s. | Dibuat resmi oleh komunitas Kubernetes; **Standar Industri De-Facto**. | Dibuat di atas mesin HAProxy; standar enterprise perbankan. |
| **Format Konfigurasi** | Ingress standar + Custom Resource Definition (**`IngressRoute`**). | **Ingress Spesifikasi Standar K8s** murni + Anotasi Nginx luas. | Ingress standar + CRD HAProxy. |
| **Popularitas di Industri** | Populer di kalangan startup, edge, dan Docker Compose. | **Paling banyak digunakan di dunia (> 70% perusahaan)**. Ribuan dokumentasi & tutorial tersedia. | Sangat populer untuk beban kerja konkurensi ekstrem dan TCP/TLS mentah. |
| **Otomasi Sertifikat HTTPS** | Otomatis mendukung Let's Encrypt bawaan tanpa modul ekstra. | Membutuhkan pemasangan alat pendamping bernama **`cert-manager`**. | Membutuhkan `cert-manager`. |
| **Konsumsi Resource** | Sangat ringan dan cepat. | Sedang (sedikit memakan RAM untuk proses worker Nginx). | Sangat efisien dalam pemakaian CPU. |

---

## 6. Mengenal Objek `IngressClass` (Kubernetes Modern)

Dahulu, sebelum Kubernetes versi 1.18, pemilihan Ingress Controller dilakukan menggunakan stiker anotasi usang:
`annotations: kubernetes.io/ingress.class: "nginx"`.
Cara lama ini rawan salah dan tidak memiliki validasi resmi.

Kubernetes modern menggunakan objek resmi bernama **`IngressClass`**:
Setiap Ingress Controller yang terpasang di dalam cluster wajib mendaftarkan dirinya ke dalam daftar kelas cluster.

```bash
# Melihat daftar Ingress Controller yang tersedia di cluster:
kubectl get ingressclass
```
*Contoh output:*
```text
NAME      CONTROLLER                      PARAMETERS   AGE
traefik   traefik.io/ingress-controller   <none>       1d
nginx     k8s.io/ingress-nginx            <none>       2h
```

Ketika kamu menulis manifes Ingress, kamu cukup menyebutkan kelas mana yang ingin kamu tugaskan di bagian **`spec.ingressClassName`**:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: toko-online-ingress
spec:
  ingressClassName: nginx        # Perintahkan Ingress-Nginx yang melayani, bukan Traefik!
  rules:
  - host: toko.sre.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: toko-service
            port:
              number: 80
```

---

## 7. Memasang Ingress Controller Alternatif (Ingress-Nginx)

Sebagai seorang SRE, kamu harus memiliki kemampuan mengganti komponen bawaan cluster dengan komponen pilihanmu sendiri.
Berikut adalah cara memasang **Ingress-Nginx Controller** resmi di lingkungan bare-metal k3s:

```bash
# 1. Pasang manifes resmi Ingress-Nginx untuk lingkungan bare-metal:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml

# 2. Pantau proses instalasi hingga Pod Controller berstatus Running:
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# 3. Periksa Service ingress-nginx di namespace-nya:
kubectl get svc -n ingress-nginx
```
Secara bawaan di bare-metal, Ingress-Nginx akan mengekspos dirinya menggunakan **`type: NodePort`** (membuka port acak 30000–32767).
Kelak di Modul 2.3, kita akan mengubah tipe Service Ingress-Nginx ini menjadi **`type: LoadBalancer`** agar mendapatkan Alamat IP Virtual (VIP) tunggal port 80/443 dari MetalLB!

---

## 8. Hands-On Mandiri Step-by-Step (20 Menit)

Mari kita latih proses penonaktifan ServiceLB dan Traefik pada mesin virtual `k3s-cp1`:

```bash
# --- LANGKAH 1: Masuk ke VM k3s-cp1 dan Pasang Konfigurasi Disable ---
ssh k3s-cp1 "sudo mkdir -p /etc/rancher/k3s"
ssh k3s-cp1 "sudo tee /etc/rancher/k3s/config.yaml << 'EOF'
disable:
  - servicelb
  - traefik
EOF"

# --- LANGKAH 2: Terapkan Konfigurasi Baru via Restart Systemd ---
ssh k3s-cp1 "sudo systemctl restart k3s"

# Tunggu 10 detik hingga daemon selesai membersihkan pod lama:
sleep 10

# --- LANGKAH 3: Verifikasi di Terminal Mac ---
# Pastikan tidak ada lagi Pod traefik atau svclb yang berjalan:
kubectl get pods -n kube-system | grep -E "traefik|svclb" || echo "Luar biasa! Traefik dan ServiceLB berhasil dibersihkan total."

# --- LANGKAH 4: Uji Pembuatan Service LoadBalancer (Harus Pending) ---
kubectl create deployment uji-bersih --image=nginx:1.27-alpine
kubectl expose deployment uji-bersih --port=80 --type=LoadBalancer

sleep 3
# Amati kolom EXTERNAL-IP (Wajib berstatus <pending>):
kubectl get svc uji-bersih

# Bersihkan:
kubectl delete svc uji-bersih
kubectl delete deployment uji-bersih
```

---

## 9. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 kendala yang sering terjadi saat memodifikasi komponen bawaan k3s beserta solusinya:

### Kasus 1: Pod `svclb-*` Masih Muncul Meskipun Sudah Menulis `--disable servicelb`
- **Gejala:** Kamu sudah mengedit konfigurasi, namun Pod `svclb` masih tetap berstatus Running di `kube-system`.
- **Penyebab:** Pada cluster multi-server, penonaktifan komponen harus diterapkan secara seragam pada **SEMUA server Control Plane**, bukan hanya di server pertama! Jika server kedua masih menyalakan ServiceLB, server kedua akan membangkitkan DaemonSet tersebut kembali.
- **Solusi SRE:** Pastikan file `/etc/rancher/k3s/config.yaml` dipasang di `k3s-cp1`, `k3s-cp2`, dan `k3s-cp3`, lalu restart layanan k3s di seluruh server master.

---

### Kasus 2: Aturan Ingress Tidak Bekerja (Trafik Menggantung / Timeout)
- **Gejala:** File Ingress berhasil di-apply, tetapi saat diakses lewat browser/curl koneksi ditolak.
- **Penyebab:** Di cluster terdapat 2 Ingress Controller (Traefik dan Nginx), tetapi di manifes Ingress kamu lupa menyertakan `spec.ingressClassName`. Akibatnya, kedua controller saling mengabaikan manifes tersebut.
- **Solusi:** Selalu deklarasikan `spec.ingressClassName: <nama-controller>` secara eksplisit di setiap manifes Ingress.

---

### Kasus 3: Perintah `kubectl top` Gagal dengan Pesan `Metrics API not available`
- **Gejala:** Perintah `kubectl top nodes` atau `kubectl top pods` menolak berjalan.
- **Penyebab:** Kamu tidak sengaja menyertakan flag `--disable metrics-server` saat instalasi k3s.
- **Solusi:** Hapus baris `metrics-server` dari daftar disable di `/etc/rancher/k3s/config.yaml`, lalu restart k3s.

---

## 10. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Penonaktifan Komponen (Cheat Sheet)

| Kebutuhan Operasional | Solusi Konfigurasi |
|---|---|
| **Matikan ServiceLB (Wajib untuk MetalLB)** | `disable: ["servicelb"]` di `/etc/rancher/k3s/config.yaml` |
| **Matikan Traefik (Untuk Nginx Ingress)** | `disable: ["traefik"]` di `/etc/rancher/k3s/config.yaml` |
| **Cek status keberhasilan disable** | `kubectl get svc <nama> -o wide` (EXTERNAL-IP wajib `<pending>`) |
| **Daftar Ingress Controller resmi** | `kubectl get ingressclass` |
| **Tentukan controller pada manifes** | `spec.ingressClassName: <nama-class>` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk menguji ketajaman analisismu sebelum melanjutkan ke perancangan topologi:

1. **Jelaskan perbedaan mendasar cara kerja *ServiceLB (Klipper)* bawaan k3s dengan cara kerja *MetalLB* dalam mengalokasikan Alamat IP untuk Service bertipe `LoadBalancer`!**
2. **Mengapa membiarkan ServiceLB tetap menyala saat kita memasang MetalLB di cluster yang sama berpotensi fatal bagi stabilitas jaringan?**
3. **Mengapa status `EXTERNAL-IP: <pending>` pada sebuah Service bertipe LoadBalancer justru menjadi bukti keberhasilan penonaktifan ServiceLB?**
4. **Sebutkan minimal 2 keuntungan teknis dari mengganti Traefik Ingress dengan Ingress-Nginx Controller di lingkungan data center perusahaan!**

---
*Lanjutkan ke materi berikutnya: [04-topologi-onprem.md](04-topologi-onprem.md) untuk mempelajari seni merancang arsitektur jaringan data center on-premises (perencanaan subnet IP, pencegahan tabrakan IP, dan rute trafik end-to-end)!*