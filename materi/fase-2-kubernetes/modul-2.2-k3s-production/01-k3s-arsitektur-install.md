# 01 — k3s: Arsitektur & Instalasi di VM OrbStack

> **Ringkasan:** Memasuki jalur simulasi data center nyata (*Production Lane*): memahami keunggulan arsitektural **k3s** untuk lingkungan on-premises bare-metal, menyiapkan mesin virtual **Ubuntu 24.04** di OrbStack dengan `systemd` penuh, menginstal k3s single-node via skrip resmi, mengekstraksi dan menyetel file **Kubeconfig** agar cluster dapat dikendalikan dari laptop Mac, serta membandingkan perbedaannya dengan `k3d`.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan menyelesaikan praktiknya di terminal, kamu akan mampu:
- Menjelaskan mengapa k3s sangat ideal untuk lingkungan server on-premises dan edge computing dibandingkan distribusi Kubernetes hulu (*Vanilla k8s*).
- Menjelaskan perbedaan mendasar antara menjalankan k3s di dalam container Docker (**k3d**) vs menjalankan k3s langsung di atas sistem operasi Linux dengan supervisi **`systemd`**.
- Menyiapkan mesin virtual Linux OrbStack dengan konfigurasi kunci SSH dan Alamat IP statis.
- Menginstal k3s single-node menggunakan perintah resmi dan memvalidasi unit layanannya (`k3s.service`).
- Menyalin, membersihkan, dan mengonfigurasi file `kubeconfig` (`k3s.yaml`) ke laptop Mac agar dapat diakses oleh `kubectl`.
- Menjalankan aplikasi uji coba dan melakukan pencopotan instalasi (*uninstall*) k3s secara bersih tanpa meninggalkan berkas sampah.

---

## 1. Konsep & Arsitektur: Mengapa k3s untuk Lingkungan On-Premises?

### Dari "Fast Lane" Menuju "Production Lane"
Pada Modul 2.1 sebelumnya, kita menggunakan **k3d** untuk mempelajari objek-objek inti Kubernetes. k3d sangat cepat dan praktis karena berjalan di dalam container Docker.
Namun, di dunia nyata seorang **Site Reliability Engineer (SRE)** yang mengelola private data center atau server fisik kantor (*bare-metal*), **Kubernetes tidak dijalankan di dalam container Docker laptop**!

Kita membutuhkan lingkungan yang mencerminkan realitas server produksi:
- Memiliki sistem operasi Linux utuh dengan kernel mandiri.
- Seluruh komponen Kubernetes berjalan di bawah supervisi manajer sistem **`systemd`** (`k3s.service`), sehingga otomatis menyala kembali jika server di-reboot.
- Memiliki antarmuka jaringan fisik dengan **Alamat IP Statis**, yang memungkinkan kita memasang load balancer jaringan fisik seperti **MetalLB (Modul 2.3)** dan mengotomasinya menggunakan **Ansible (Fase 4)**.

Inilah yang kita sebut sebagai **Production Lane**: Kita memasang k3s langsung di dalam **OrbStack Linux Machine (VM Ubuntu)**!

```text
 ┌─ K3D (MODUL 2.1: FAST LANE) ───────────┐   ┌─ K3S DI VM (MODUL 2.2: PRODUCTION LANE) ┐
 │ • Berjalan sebagai CONTAINER Docker    │   │ • Berjalan di atas VM LINUX SUNGGUHAN   │
 │ • Tanpa systemd di dalam node          │   │ • Dikelola oleh unit SYSTEMD (PID 1)    │
 │ • IP internal Docker bridge (berubah)  │   │ • Alamat IP Statis stabil               │
 │ • Cocok untuk: Coret-coret YAML harian │   │ • Cocok untuk: Simulasi On-Prem & MetalLB│
 └────────────────────────────────────────┘   └─────────────────────────────────────────┘
```

---

### Keunggulan k3s untuk Infrastruktur Mandiri (*On-Premises*)
Kubernetes versi hulu (*Vanilla Kubernetes*) terkenal sangat rakus sumber daya dan rumit untuk diinstal dari nol. Satu server master membutuhkan RAM minimal 4 GB dan puluhan sertifikat TLS yang harus dibuat manual.

**Rancher k3s** hadir menyederhanakan segalanya tanpa membuang standar resmi CNCF:
1. **Satu File Biner Tunggal (< 70 MB):** Seluruh komponen Control Plane (`kube-apiserver`, `scheduler`, `controller-manager`), Worker (`kubelet`, `kube-proxy`), dan runtime container (`containerd`) dikemas rapi menjadi satu file biner `/usr/local/bin/k3s`.
2. **Kebutuhan Memori Sangat Kecil:** k3s hanya membutuhkan sekitar **512 MB RAM** untuk berjalan. Ini memungkinkan kita menyalakan cluster 3 hingga 5 node di laptop MacBook tanpa membuat Mac kepanasan!
3. **Instalasi Satu Baris Perintah:** Cukup mengeksekusi `curl -sfL https://get.k3s.io | sh -`, k3s akan otomatis mengunduh biner, membuat user sistem, menyetel sertifikat TLS internal, dan mengaktifkan service `systemd`.
4. **Komponen Modular yang Fleksibel:** Komponen bawaan k3s (seperti Traefik atau ServiceLB) dapat dimatikan dengan mudah lewat satu flag parameter agar bisa digantikan oleh komponen standar enterprise seperti MetalLB dan Nginx Ingress.

---

## 2. Menyiapkan Mesin Virtual Ubuntu di OrbStack

Sebelum menginstal k3s, kita siapkan terlebih dahulu "server virtual" yang akan menjadi rumah bagi node Control Plane pertama kita.

```bash
# 1. Buat VM baru bernama 'k3s-cp1' berbasis Ubuntu 24.04 LTS (native ARM64 di Mac M5):
orb create ubuntu:24.04 k3s-cp1

# 2. Catat Alamat IP stabil dari mesin tersebut:
orb ip k3s-cp1
# Contoh output: 198.19.249.10 (Alamat IP ini statis dan tidak akan berubah!)
```

### Konfigurasi Akses SSH Berbasis Kunci (Standar SRE)
Agar kelak di Fase 4 kita bisa mengotomasi server ini menggunakan **Ansible**, kita siapkan konfigurasi SSH standar:

```bash
# Salin public key Mac (~/.ssh/id_ed25519.pub) ke dalam VM k3s-cp1:
orb ssh k3s-cp1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
cat ~/.ssh/id_ed25519.pub | orb ssh k3s-cp1 "cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Daftarkan alias host di file ~/.ssh/config pada Mac:
cat << EOF >> ~/.ssh/config

Host k3s-cp1
    HostName $(orb ip k3s-cp1)
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
EOF

# Uji koneksi SSH standar:
ssh k3s-cp1 "hostname && uname -m"
```

---

## 3. Instalasi k3s Single-Node

Sekarang kita masuk ke dalam mesin `k3s-cp1` dan mengeksekusi skrip instalasi resmi k3s. Pada mode *Single-Node*, server ini akan bertindak ganda: sebagai **Control Plane (Otak)** sekaligus sebagai **Worker (Tempat Pod Berjalan)**.

```bash
# Masuk ke dalam VM via SSH:
ssh k3s-cp1

# Jalankan skrip instalasi otomatis k3s:
curl -sfL https://get.k3s.io | sh -
```

*Contoh proses output di layar terminal:*
```text
[INFO]  Finding release for channel stable
[INFO]  Using v1.30.4+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/...
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/...
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating systemd service /etc/systemd/system/k3s.service
[INFO]  Enabling k3s unit
[INFO]  Starting k3s
```

---

### Memeriksa Status Layanan `systemd` di Dalam VM
Perhatikan bahwa k3s kini telah berjalan sebagai daemon sistem Linux sejati:

```bash
# Periksa status service systemd:
sudo systemctl status k3s --no-pager
```
*Output yang diharapkan:*
```text
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-10-17 12:55:00 WIB; 25s ago
```

### Menggunakan `kubectl` Bawaan di Dalam VM
Di dalam VM, k3s secara otomatis menyediakan jalan pintas perintah `kubectl`. Jalankan perintah verifikasi node:

```bash
sudo k3s kubectl get nodes -o wide
```
*Output:*
```text
NAME      STATUS   ROLES                  AGE   VERSION        INTERNAL-IP      OS-IMAGE
k3s-cp1   Ready    control-plane,master   45s   v1.30.4+k3s1   198.19.249.10    Ubuntu 24.04 LTS
```

Periksa seluruh Pod sistem bawaan yang otomatis dinyalakan oleh k3s:
```bash
sudo k3s kubectl get pods -A
```
Kamu akan melihat Pod **CoreDNS**, **Traefik Ingress**, **ServiceLB (klipper)**, dan **Local Path Storage Provisioner** sudah berjalan sehat.

---

## 4. Mengendalikan Cluster dari Mac: Ekstraksi Kubeconfig

Sebagai seorang SRE, kita tidak ingin terus-menerus melakukan SSH ke dalam server setiap kali ingin menjalankan perintah `kubectl`. Kita ingin mengendalikan cluster tersebut langsung dari kenyamanan terminal laptop Mac kita!

Di sinilah letak perbedaan teknis terbesar dibanding k3d:
- k3d secara otomatis menyuntikkan konfigurasi ke laptop Mac.
- Pada k3s di server VM, file **Kubeconfig berada di `/etc/rancher/k3s/k3s.yaml` di dalam VM milik user `root`**.
- Kita harus **menyalin file tersebut ke Mac**, lalu **mengganti alamat IP lokal `127.0.0.1` dengan alamat IP asli VM**!

```text
 Di Dalam VM (k3s-cp1):
 /etc/rancher/k3s/k3s.yaml ───► server: https://127.0.0.1:6443
                                            │
                                            ▼ (Jika disalin mentah-mentah ke Mac...)
 Di Laptop Mac:
 kubectl get nodes ───────────► Menghubungi 127.0.0.1:6443 milik Mac sendiri!
                                💥 ERROR: connection refused (Mac bukan server k3s!)
                                            │
                                            ▼ (Solusi SRE: Edit alamat IP)
 Ganti 127.0.0.1 ────────────► https://198.19.249.10:6443 (IP VM k3s-cp1)
 Hasilnya: kubectl dari Mac berhasil tersambung ke VM!
```

---

### Prosedur Ekstraksi & Penyetelan Kubeconfig di Mac

Buka jendela terminal laptop Mac milikmu, lalu jalankan langkah-langkah berikut:

```bash
# 1. Ambil IP VM k3s-cp1:
VM_IP=$(orb ip k3s-cp1)

# 2. Salin isi file k3s.yaml dari VM, lalu ganti 127.0.0.1 menjadi IP VM asli menggunakan 'sed':
ssh k3s-cp1 "sudo cat /etc/rancher/k3s/k3s.yaml" | sed "s/127.0.0.1/${VM_IP}/g" > /tmp/k3s-cp1.yaml

# 3. Gabungkan file konfigurasi baru tersebut ke dalam ~/.kube/config milik Mac:
KUBECONFIG=~/.kube/config:/tmp/k3s-cp1.yaml kubectl config view --flatten > ~/.kube/config.tmp
mv ~/.kube/config.tmp ~/.kube/config
rm -f /tmp/k3s-cp1.yaml

# 4. Beri nama Context yang jelas dan mudah diingat (agar tidak bentrok dengan k3d):
kubectl config rename-context default k3s-production 2>/dev/null || true

# 5. Beralih ke context k3s-production:
kubectl config use-context k3s-production

# 6. Uji kendali dari terminal Mac:
kubectl get nodes -o wide
```

Sekarang kamu memiliki dua jalur (*two lanes*) di terminal Mac yang sama:
```bash
# Pindah ke Jalur Cepat (k3d):
kubectl config use-context k3d-lab

# Pindah ke Jalur Produksi Server VM (k3s):
kubectl config use-context k3s-production
```

---

## 5. Membandingkan k3s di VM vs k3d di Container

| Aspek Operasional | **k3d** (Modul 2.1) | **k3s di VM OrbStack** (Modul Ini) |
|---|---|---|
| **Rumah Eksekusi** | Container Docker biasa di Mac | Mesin Virtual Linux Ubuntu lengkap |
| **Pengawas Proses (Init)** | Tidak ada `systemd` di node | **Diawasi oleh `systemd` (`k3s.service`)** |
| **Alamat IP Node** | IP virtual bridge Docker (mudah berubah) | **Alamat IP Statis VM nyata (`orb ip`)** |
| **Penyetelan Kubeconfig** | Otomatis via `k3d kubeconfig merge` | **Disalin manual dan disanitasi IP-nya** |
| **Dukungan MetalLB Layer 2**| Sangat sulit (terhalang jembatan Docker) | **100% Sempurna** (jaringan bridge fisik VM) |
| **Target Otomasi Ansible** | Tidak bisa diperlakukan sebagai server SSH | **Bisa di-SSH** layaknya server data center |
| **Perilaku Saat Reboot** | Container harus di-start ulang manual | **Otomatis boot & start k3s via systemd** |

---

## 6. Prosedur Uninstall k3s Secara Bersih

Salah satu keunggulan besar k3s adalah ia menyediakan skrip pencopotan instalasi resmi yang sangat bersih.
Jika sewaktu-waktu kamu ingin mereset server dan memulai instalasi dari awal, kamu tidak perlu menghapus mesin VM-nya:

```bash
# Masuk ke dalam VM:
ssh k3s-cp1

# Jalankan skrip uninstall bawaan:
sudo /usr/local/bin/k3s-uninstall.sh
```

Skrip ini akan secara otomatis:
- Menghentikan proses `k3s.service`.
- Menghapus unit file systemd dan symlink `kubectl`, `crictl`, serta `ctr`.
- Menghapus seluruh direktori data database di `/var/lib/rancher/k3s`.
- Menghapus konfigurasi jaringan virtual cni (`/etc/cni/net.d`).
Server VM kembali bersih 100% dan siap diinstal ulang!

---

## 7. Hands-On Mandiri Step-by-Step (20 Menit)

Mari kita latih seluruh siklus hidup instalasi k3s, menghubungkannya ke Mac, dan mendeploy aplikasi pertama:

```bash
# --- LANGKAH 1: Siapkan VM k3s-cp1 ---
orb create ubuntu:24.04 k3s-cp1 2>/dev/null || orb start k3s-cp1
VM_IP=$(orb ip k3s-cp1)

# Pastikan SSH key terpasang:
orb ssh k3s-cp1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
cat ~/.ssh/id_ed25519.pub | orb ssh k3s-cp1 "cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# --- LANGKAH 2: Instal k3s di Dalam VM ---
ssh k3s-cp1 "curl -sfL https://get.k3s.io | sh -"

# Verifikasi service aktif:
ssh k3s-cp1 "sudo systemctl is-active k3s"

# --- LANGKAH 3: Ekstraksi Kubeconfig ke Laptop Mac ---
ssh k3s-cp1 "sudo cat /etc/rancher/k3s/k3s.yaml" | sed "s/127.0.0.1/${VM_IP}/g" > /tmp/k3s-test.yaml
KUBECONFIG=~/.kube/config:/tmp/k3s-test.yaml kubectl config view --flatten > ~/.kube/config.tmp
mv ~/.kube/config.tmp ~/.kube/config
rm -f /tmp/k3s-test.yaml

kubectl config rename-context default k3s-vm 2>/dev/null || true
kubectl config use-context k3s-vm

# --- LANGKAH 4: Deploy Aplikasi dari Terminal Mac ---
kubectl create namespace latihan
kubectl create deployment halo-k3s --image=nginx:1.27-alpine -n latihan --replicas=2
kubectl rollout status deployment/halo-k3s -n latihan
kubectl get pods -n latihan -o wide

# Uji akses dengan port-forward langsung ke VM:
kubectl port-forward -n latihan deployment/halo-k3s 9095:80 &
PF_PID=$!
sleep 2
curl -I http://localhost:9095/
kill $PF_PID 2>/dev/null || true

# --- LANGKAH 5: Bersihkan Namespace Latihan ---
kubectl delete namespace latihan
```

---

## 8. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 kendala yang paling sering terjadi saat menginstal k3s di VM beserta solusinya:

### Kasus 1: Gagal Menyalin Kubeconfig karena `Permission Denied`
- **Gejala:** Saat mengetik `scp ubuntu@k3s-cp1:/etc/rancher/k3s/k3s.yaml .`, terminal mengeluarkan error *Permission denied*.
- **Penyebab:** File `/etc/rancher/k3s/k3s.yaml` dibuat dengan hak akses ketat (`chmod 600`) dan dimiliki eksklusif oleh user `root`. User biasa `ubuntu` tidak punya izin baca.
- **Solusi SRE:** Gunakan `sudo` di dalam sesi SSH untuk membaca isinya dan alirkan (*pipe*) ke laptop Mac:
  ```bash
  ssh k3s-cp1 "sudo cat /etc/rancher/k3s/k3s.yaml" > /tmp/k3s.yaml
  ```

---

### Kasus 2: `kubectl` dari Mac Mengeluarkan Error `connection refused` ke `127.0.0.1:6443`
- **Gejala:** Perintah `kubectl get nodes` dari laptop Mac gagal terhubung ke port 6443.
- **Penyebab:** Kamu lupa menjalankan perintah `sed` untuk mengganti alamat `127.0.0.1` di dalam file kubeconfig menjadi alamat IP VM asli (`198.19.249.x`).
- **Solusi:** Buka file `~/.kube/config` di Mac, cari blok `cluster.server` milik k3s, dan ganti `https://127.0.0.1:6443` menjadi `https://<IP_VM_ORBSTACK>:6443`.

---

### Kasus 3: k3s Service Gagal Start (`CrashLoop` di systemd)
- **Gejala:** Perintah `systemctl status k3s` menampilkan status `failed` atau terus me-restart.
- **Penyebab:** Memori RAM VM terlalu kecil (< 512 MB), atau ada proses web server lain yang sudah menduduki port 80/443 sebelum k3s diinstal.
- **Diagnosa SRE:**
  Periksa catatan log journal systemd secara langsung:
  ```bash
  sudo journalctl -u k3s.service -e --no-pager
  ```
  Baca pesan error baris paling bawah untuk melihat penyebab kegagalan daemon k3s.

---

## 9. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Perintah k3s di VM (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Instal k3s di dalam VM** | `curl -sfL https://get.k3s.io \| sh -` |
| **Cek status service systemd** | `sudo systemctl status k3s` |
| **Restart k3s daemon** | `sudo systemctl restart k3s` |
| **Baca log internal k3s** | `sudo journalctl -u k3s -f` |
| **Ekstraksi kubeconfig ke Mac** | `ssh <vm> "sudo cat /etc/rancher/k3s/k3s.yaml" \| sed "s/127.0.0.1/<IP>/"` |
| **Uninstall bersih k3s dari VM** | `sudo /usr/local/bin/k3s-uninstall.sh` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk mengevaluasi pemahaman dasarmu sebelum melangkah ke cluster Multi-Node:

1. **Sebutkan minimal 2 alasan mengapa kita beralih menggunakan k3s di dalam VM OrbStack untuk simulasi data center on-premises dan tidak cukup hanya mengandalkan k3d di dalam container!**
2. **Apa yang dilakukan oleh skrip instalasi `curl -sfL https://get.k3s.io | sh -` di balik layar sistem operasi Linux Ubuntu? Komponen apa yang otomatis didaftarkan ke `systemd`?**
3. **Mengapa file `/etc/rancher/k3s/k3s.yaml` yang baru saja disalin dari VM tidak bisa langsung digunakan oleh `kubectl` di laptop Mac tanpa mengubah alamat IP-nya terlebih dahulu?**
4. **Jika laptop MacBook milikmu di-restart, apakah cluster k3s di dalam VM akan otomatis menyala kembali saat VM dihidupkan? Jelaskan mekanismenya!**

---
*Lanjutkan ke materi berikutnya: [02-k3s-multi-node-ha.md](02-k3s-multi-node-ha.md) untuk mempelajari cara merangkai beberapa mesin virtual menjadi cluster berkemampuan tinggi (**High Availability**) menggunakan database **Embedded etcd Quorum**!*