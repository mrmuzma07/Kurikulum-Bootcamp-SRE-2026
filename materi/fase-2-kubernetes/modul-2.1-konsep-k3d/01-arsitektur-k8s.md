# 01 — Arsitektur Kubernetes: Control Plane, Worker Node & Alur Rekonsiliasi

> **Ringkasan:** Membedah anatomi mesin internal Kubernetes dari balik layar: memahami pembagian peran antara **Control Plane (Otak Pengendali)** dan **Worker Nodes (Otot Pelaksana)**, fungsi krusial database **`etcd`** sebagai sumber kebenaran tunggal (*Single Source of Truth*), peran mandor **`kubelet`**, serta menelusuri siklus hidup sebuah permintaan (*Request Lifecycle*) dari ketikan `kubectl apply` hingga container benar-benar berjalan di server.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal MacBook, kamu akan mampu:
- Menjelaskan mengapa sistem orkestrasi container seperti Kubernetes dibutuhkan di lingkungan modern menggunakan analogi orkestra / bandara.
- Membedakan peran dan tanggung jawab antara **Control Plane** (komponen master) dan **Worker Node** (komponen pelaksana beban kerja).
- Menjelaskan fungsi masing-masing komponen inti: **`kube-apiserver`**, **`etcd`**, **`kube-scheduler`**, **`kube-controller-manager`**, **`kubelet`**, **`kube-proxy`**, dan **`container runtime`**.
- Menjelaskan mengapa **`etcd`** adalah jantung dari seluruh cluster dan mengapa prosedur pencadangan (*backup*) etcd wajib dikuasai oleh seorang SRE.
- Memetakan alur kerja rekonsiliasi (*Loop Control / Desired vs Actual State*) saat menjalankan perintah `kubectl apply`.
- Memahami bagaimana distribusi ringan **k3s** dan **k3d** menyederhanakan arsitektur Kubernetes standar tanpa kehilangan kepatuhan standar CNCF.

---

## 1. Konsep Dasar: Mengapa Kita Membutuhkan Kubernetes?

### Masalah Nyata: Mengelola Ratusan Container di Puluhan Server
Di Fase 1, kita belajar mengoperasikan container menggunakan Docker dan Docker Compose. Di satu laptop atau satu server tunggal, perintah `docker run` dan `docker compose up` bekerja sangat baik.

Namun, bayangkan skenario nyata di sebuah perusahaan e-commerce atau perbankan:
1. Kamu memiliki **200 aplikasi mikro (*microservices*)** yang berjalan di atas **15 server bare-metal** yang tersebar di data center.
2. Jam 02.00 dini hari, **Server fisik nomor 03 tiba-tiba mati lampu atau mengalami kerusakan motherboard**.
3. Di dalam server 03 tersebut terdapat 30 container penting (seperti layanan pembayaran, autentikasi, dan katalog produk).

**Siapa yang akan memindahkan ke-30 container tersebut ke server nomor 04 dan 05 yang masih sehat?**
Jika mengandalkan manusia, sistem akan mati (*downtime*) selama berjam-jam sampai tim operasional bangun dan mengetik perintah manual via SSH.
Di sinilah kita membutuhkan sebuah **Sistem Orkestrasi Container Otomatis (Container Orchestrator)**. Dan standar industri dunia saat ini adalah **Kubernetes** (sering disingkat **k8s**).

```text
       KONDUKTOR / OTAK                         PARA PEMUSIK / OTOT
   ┌───────────────────────┐                 ┌───────────────────────┐
   │                       │  Perintah Partitur│  Node Worker 01       │
   │     CONTROL PLANE     │ ──────────────► │  (Menjalankan Pod A)  │
   │  (Otak Pengendali k8s)│                 └───────────────────────┘
   │                       │                 ┌───────────────────────┐
   │ • Memantau kesehatan  │                 │  Node Worker 02       │
   │ • Menjaga jumlah pod  │ ──────────────► │  (Menjalankan Pod B)  │
   │ • Membagi beban kerja │                 └───────────────────────┘
   └───────────────────────┘                 ┌───────────────────────┐
                                             │  Node Worker 03       │
                                             │  (💥 Mati Lampu!)     │
                                             └───────────────────────┘
                                                         │
               Kubernetes mendeteksi kematian 03 ────────┘
               dan otomatis memindahkan Pod ke Node 01 & 02!
```

> 🎼 **Analogi Orkestra Musik Simfoni:**
> - Di atas panggung ada puluhan musisi yang memegang biola, terompet, dan drum (**Worker Nodes & Container**).
> - Di depan mereka berdiri seorang **Konduktor Musik (Control Plane)** yang memegang partitur nada lagu (**Spesifikasi Manifes YAML**).
> - Musisi tidak boleh bermain sendiri-sendiri sesuka hati. Konduktor yang memberi aba-aba: kapan biola harus masuk, seberapa kencang temponya, dan jika ada pemain terompet yang pingsan kelelahan, konduktor segera memanggil pemain cadangan untuk menggantikannya secara mulus tanpa menghentikan pertunjukan!

---

## 2. Arsitektur Dua Pilar: Control Plane vs Worker Node

Sebuah cluster Kubernetes selalu dibagi menjadi dua kubu utama:

```text
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                       KUBERNETES CONTROL PLANE                          │
 │                                                                         │
 │                     ┌───────────────────────┐                           │
 │                     │     kube-apiserver    │ ◄── Satu-satunya pintu!   │
 │                     └───────┬───────┬───────┘     (kubectl, controller) │
 │                             │       │                                   │
 │              ┌──────────────┘       └──────────────┐                    │
 │              ▼                                     ▼                    │
 │    ┌──────────────────┐                  ┌──────────────────┐           │
 │    │       etcd       │                  │  kube-scheduler  │           │
 │    │ (Database State) │                  │ (Penentu Lokasi) │           │
 │    └──────────────────┘                  └──────────────────┘           │
 │                                                    │                    │
 │                                          ┌─────────┴────────┐           │
 │                                          │ kube-controller- │           │
 │                                          │     manager      │           │
 │                                          └──────────────────┘           │
 └───────────────────────────────────┬─────────────────────────────────────┘
                                     │
                 Jaringan Komunikasi │ (Instruksi & Laporan Status)
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                          WORKER NODES (DATA PLANE)                      │
 │                                                                         │
 │   NODE WORKER 01                          NODE WORKER 02                │
 │  ┌─────────────────────────────┐         ┌─────────────────────────────┐│
 │  │ 👮 kubelet (Mandor Node)    │         │ 👮 kubelet (Mandor Node)    ││
 │  ├─────────────────────────────┤         ├─────────────────────────────┤│
 │  │ 🚦 kube-proxy (Jaringan)    │         │ 🚦 kube-proxy (Jaringan)    ││
 │  ├─────────────────────────────┤         ├─────────────────────────────┤│
 │  │ 📦 containerd (Runtime CRI) │         │ 📦 containerd (Runtime CRI) ││
 │  │   ┌───────┐ ┌───────┐       │         │   ┌───────┐ ┌───────┐       ││
 │  │   │ Pod A │ │ Pod B │       │         │   │ Pod C │ │ Pod D │       ││
 │  │   └───────┘ └───────┘       │         │   └───────┘ └───────┘       ││
 │  └─────────────────────────────┘         └─────────────────────────────┘│
 └─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Komponen Inti Control Plane (Otak Pengendali)

Control Plane bertanggung jawab atas segala keputusan global di dalam cluster (seperti menjadwalkan aplikasi, mendeteksi node yang mati, dan merespons perubahan konfigurasi).

### 1. `kube-apiserver` (Pintu Gerbang & Meja Resepsionis)
- **Peran:** Adalah komponen paling sibuk di Control Plane. `kube-apiserver` menyediakan antarmuka REST API berbasis HTTPS yang menjadi **satu-satunya pintu masuk** bagi seluruh komunikasi di dalam cluster.
- **Tugas Utama:**
  - Mengautentikasi dan mengotorisasi siapa yang memanggil (apakah pengguna manusia via `kubectl`, atau agen internal cluster via ServiceAccount).
  - Memvalidasi isi sintaks manifes YAML yang dikirim.
  - **Aturan Emas:** `kube-apiserver` adalah **SATU-SATUNYA komponen yang diizinkan berbicara dan menulis data ke dalam database `etcd`**. Tidak ada komponen lain yang boleh menyentuh `etcd` secara langsung!

### 2. `etcd` (Buku Catatan Sejarah & Sumber Kebenaran Tunggal)
- **Peran:** Adalah database terdistribusi berbasis *Key-Value Store* yang super konsisten (menggunakan algoritma konsensus Raft).
- **Apa yang Disimpan di etcd?** Seluruh informasi cluster tanpa terkecuali: daftar seluruh node yang aktif, status setiap Pod, file konfigurasi ConfigMap, password Secret, hingga aturan lalu lintas Service.
- **Mengapa SRE Wajib Menjaga etcd?**
  Jika harddisk server etcd terbakar dan kamu tidak memiliki salinan cadangan (*backup snapshot*), cluster-mu akan mengalami **amnesia total**! Pod yang sedang berjalan di node mungkin masih hidup sementara waktu, tetapi Control Plane tidak lagi mengenali siapa mereka, tidak bisa melakukan scaling, dan tidak bisa memulihkan sistem jika terjadi gangguan.

### 3. `kube-scheduler` (Manajer Penempatan & Penjaga Pintu Masuk)
- **Peran:** Bertugas mengamati Pod yang baru dibuat tetapi belum memiliki tempat tinggal (*unassigned node*), lalu memilihkan **Node Worker mana yang paling tepat dan sehat** untuk menampung Pod tersebut.
- **Faktor Pertimbangan Scheduler:**
  - Ketersediaan kapasitas perangkat keras (Apakah Node A masih punya cukup sisa RAM 512 MB sesuai permintaan Pod?).
  - Aturan *Affinity* dan *Anti-Affinity* (misal: "Jangan letakkan Pod Database primer dan Pod Database sekunder di server fisik yang sama demi keselamatan redundansi!").
  - *Taints dan Tolerations* (penolakan node tertentu khusus untuk beban kerja khusus).

### 4. `kube-controller-manager` (Pengawas Lingkaran Setan / Rekonsiliasi)
- **Peran:** Menjalankan puluhan proses pengawas (*controller*) di latar belakang yang terus-menerus membandingkan: **Apa yang kamu inginkan (*Desired State*)** vs **Apa yang sedang terjadi di lapangan (*Actual State*)**.
- Contoh Controller Bawaan:
  - **Node Controller:** Memeriksa detak jantung (*heartbeat*) setiap node. Jika sebuah node tidak memberi kabar selama 40 detik, node tersebut ditandai `NotReady`. Jika tembus 5 menit, controller akan memerintahkan evakuasi seluruh Pod di node tersebut.
  - **Deployment / ReplicaSet Controller:** Memastikan jumlah salinan aplikasi selalu sesuai target (misal: kamu minta 3 Pod, jika 1 Pod mati, controller ini yang memerintahkan pembuatan 1 Pod baru pengganti).
  - **EndpointSlice Controller:** Memperbarui daftar IP Pod yang sedang sehat untuk diteruskan ke Service.

---

## 4. Komponen Inti Worker Node (Otot Pelaksana)

Worker Node adalah komputer server (bisa berupa mesin fisik bare-metal, VM Ubuntu di OrbStack, atau container di k3d) tempat aplikasi sesungguhnya bekerja keras melayani trafik pengguna.

### 1. `kubelet` (Mandor Setia di Setiap Node)
- **Peran:** Adalah agen resmi Kubernetes yang berjalan sebagai *daemon* sistem di setiap node worker.
- **Tugas Utama:**
  - Menerima pesanan spesifikasi Pod (*PodSpec*) dari `kube-apiserver`.
  - Berbicara ke Container Runtime untuk menyalakan atau mematikan container sesuai perintah.
  - Menjalankan pemeriksaan kesehatan aplikasi (*Liveness* dan *Readiness Probes*). Jika aplikasi hang atau gagal merespons, `kubelet` yang bertindak merestart container tersebut.
  - Melaporkan kondisi kesehatan node (pemakaian CPU, RAM, status disk) kembali ke `kube-apiserver`.

### 2. `container runtime` (Mesin Pemutar Container / CRI)
- **Peran:** Perangkat lunak tingkat rendah yang benar-benar bertugas mengunduh image dari registry, membuat cgroups, menyetel namespaces, dan menyalakan container Linux.
- Kubernetes menggunakan antarmuka standar bernama **CRI (Container Runtime Interface)**.
- Di lingkungan modern (termasuk k3s dan k3d), runtime default yang digunakan adalah **`containerd`** (bukan Docker Engine utuh yang berat).

### 3. `kube-proxy` (Polisi Lalu Lintas Jaringan Node)
- **Peran:** Mengelola aturan jaringan (*network routing rules*) di tingkat kernel sistem operasi pada setiap node menggunakan **`iptables`** atau **`IPVS`**.
- `kube-proxy` memastikan bahwa ketika aplikasi memanggil alamat IP virtual milik sebuah **Kubernetes Service**, paket data tersebut diteruskan dan dibagi secara seimbang (*load balanced*) ke alamat IP Pod yang tepat di mana pun Pod tersebut berada di dalam cluster.

---

## 5. Melacak Jalur Permintaan: Dari `kubectl apply` Hingga Pod `Running`

Pernahkah kamu memperhatikan bahwa ketika kamu mengetik perintah `kubectl apply -f deployment.yaml`, terminalmu langsung selesai dalam 0,5 detik dengan pesan `deployment.apps/web created`, tetapi ketika kamu mengetik `kubectl get pods`, status Pod-nya masih `ContainerCreating` selama beberapa detik sebelum akhirnya berubah menjadi `Running`?

Mengapa ada jeda waktu tersebut? Mari kita bedah 8 langkah rekonsiliasi yang terjadi di balik layar:

```text
 1. SRE mengetik: kubectl apply -f deployment.yaml (minta 3 replika)
    │
    ▼ (Kirim via HTTPS REST API)
 2. kube-apiserver memvalidasi izin (RBAC) & menyimpan deklarasi ➔ etcd
    │
    ▼ (Mendeteksi perubahan via Watch Event)
 3. Deployment Controller melihat ada Deployment baru, lalu membuat objek ReplicaSet.
    ReplicaSet Controller melihat butuh 3 Pod, lalu membuat 3 spesifikasi Pod tanpa nama node.
    │
    ▼ (Mendeteksi ada Pod yang belum punya rumah)
 4. kube-scheduler menganalisis beban node, memilih Node-02 untuk Pod tersebut,
    lalu menuliskan keputusan 'nodeName: node-02' ke ➔ kube-apiserver ➔ etcd.
    │
    ▼ (Watch Event di Node-02)
 5. kubelet di Node-02 melihat ada tugas baru yang ditugaskan ke dirinya!
    │
    ▼ (Perintah via gRPC CRI)
 6. kubelet menyuruh containerd: "Unduh image nginx:alpine dan nyalakan container!"
    │
    ▼
 7. containerd menyiapkan namespace, cgroups, dan menyalakan container.
    │
    ▼
 8. kubelet memastikan container sudah hidup dan sehat, lalu mengirim laporan:
    "Status Pod: Running!" ke ➔ kube-apiserver ➔ etcd.
```

> 💡 **Pelajaran Fundamental SRE: Sistem Berbasis Peristiwa (*Event-Driven Watch*):**
> Komponen-komponen Kubernetes tidak bekerja dengan cara menanyai database secara kasar (*polling*) setiap detik yang memboroskan CPU. Setiap komponen membuka sambungan saluran siaga (*Watch Stream*) ke `kube-apiserver`. Begitu ada data baru di `etcd`, `kube-apiserver` langsung mengabari (*push notification*) komponen yang berkepentingan untuk segera bertindak!

---

## 6. Bagaimana k3s & k3d Menyederhanakan Arsitektur Raksasa Ini?

Di lingkungan Kubernetes kelas enterprise tradisional (seperti OpenShift atau kubespray), setiap komponen di atas diinstal sebagai program terpisah, memiliki file konfigurasi sertifikat TLS masing-masing, dan membutuhkan minimal 3 server master berukuran besar hanya untuk Control Plane.

**Rancher k3s** melakukan inovasi luar biasa untuk kebutuhan *Edge, IoT, dan On-Premises*:
1. **Semua Komponen Digabung Menjadi 1 Biner Ringan:** `kube-apiserver`, `scheduler`, `controller-manager`, `kubelet`, `kube-proxy`, dan `containerd` dikompilasi menjadi satu file biner tunggal `/usr/local/bin/k3s` yang ukurannya di bawah 100 MB!
2. **Penggantian Database Cerdas:** Untuk cluster single-node atau lab lokal, k3s secara default mengganti database `etcd` yang boros memori dengan database **SQLite** yang super cepat dan hemat RAM. Namun untuk cluster production multi-server, k3s tetap menyediakan mode `etcd` terdistribusi bawaan (*embedded etcd*).
3. **Membuang Kode Bawaan Cloud Kuno:** Driver cloud lawas (untuk AWS, GCP, Azure lama) yang menumpuk di Kubernetes hulu dibuang seluruhnya, sehingga k3s sangat murni dan ideal untuk server fisik bare-metal mandiri.

Dan **k3d** melangkah satu tahap lebih praktis lagi:
k3d mengambil biner k3s tersebut, lalu menjalankannya di dalam container Docker di laptop Mac milikmu, sehingga kamu bisa memiliki cluster 3-node yang siap pakai hanya dalam waktu **10 detik**!

---

## 7. Ketersediaan Tinggi: High Availability (HA) Control Plane

Dalam operasional SRE, kita selalu memikirkan: *"Bagaimana jika server Control Plane mati?"*

| Karakteristik | Single-Server Control Plane (Cocok untuk Lab / Dev) | High-Availability (HA) Clustered (Wajib di Production) |
|---|---|---|
| **Jumlah Node Master** | **1 Node Server** | **Minimal 3 Node Server** (Harus ganjil: 3, 5, 7) |
| **Penyimpanan State** | SQLite (k3s default) atau 1 etcd lokal | **Clustered etcd (Quorum Consensus)** |
| **Toleransi Kerusakan** | **Nol:** Jika server master mati, cluster tidak bisa menerima perintah baru. | **Toleran:** Jika 1 dari 3 server master terbakar, cluster tetap berjalan normal tanpa gangguan! |
| **Kapan Digunakan?** | • Laboratorium harian **k3d** (Modul 2.1)<br>• Lingkungan pengujian fitur | • Simulasi **k3s Production On-Premises** (Modul 2.2)<br>• Server produksi data center nyata |

> 🗳️ **Mengapa Node etcd Harus Berjumlah Ganjil (3 atau 5)?**
> `etcd` menggunakan sistem pemungutan suara mayoritas (*Quorum*) untuk menyetujui setiap penulisan data: **`Quorum = (N/2) + 1`**.
> - Pada cluster 3 server, suara mayoritas yang dibutuhkan adalah `(3/2) + 1 = 2`. Jika 1 server mati, masih ada 2 server yang hidup (`2 >= 2`), sehingga cluster tetap aman beroperasi!
> - Jika kamu memakai 2 server, kuorumnya tetap 2. Jika 1 server mati, sisa 1 server tidak mencapai kuorum (`1 < 2`) dan seluruh cluster akan mogok menulis data (*Split-Brain protection*). Maka 3 server jauh lebih aman daripada 2 server!

---

## 8. Hands-On Mandiri Step-by-Step (15 Menit)

Mari kita bedah dan buktikan keberadaan komponen-komponen arsitektur ini di cluster `k3d` milikmu:

```bash
# --- LANGKAH 1: Pastikan Cluster k3d Multi-Node Sedang Berjalan ---
k3d cluster delete lab 2>/dev/null || true
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer

# Periksa peran masing-masing node
kubectl get nodes -o wide

# --- LANGKAH 2: Mengintip Pod-Pod Milik Sistem Kubernetes ---
# Seluruh komponen pendukung k8s (DNS, Ingress, Network) ditaruh di namespace 'kube-system'
kubectl get pods -n kube-system -o wide

# --- LANGKAH 3: Membuktikan Kubelet & Containerd Berjalan di Dalam Node ---
# Masuk ke dalam container node k3d server dan intip proses Linux aslinya:
docker exec -it k3d-lab-server-0 ps aux | grep -E "k3s|containerd" | head -n 10

# --- LANGKAH 4: Menguji Kesiapan & Titik Kesehatan (Healthz/Readyz) API Server ---
# Jalankan query diagnostik langsung ke API Server k8s:
kubectl get --raw='/readyz?verbose'

# --- LANGKAH 5: Mengamati Perjalanan Pembuatan Pod Secara Real-Time ---
# Buka satu terminal untuk memantau event cluster:
kubectl get events --sort-by='.metadata.creationTimestamp' -w &
EVENT_PID=$!

# Buat sebuah pod uji coba:
kubectl run uji-alur --image=nginx:1.27-alpine

# Hentikan pemantauan event di latar belakang:
kill $EVENT_PID 2>/dev/null || true

# Bersihkan pod uji coba:
kubectl delete pod uji-alur
```

---

## 9. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 masalah arsitektural yang paling sering ditemui di lingkungan produksi beserta langkah diagnosanya:

### Kasus 1: Perintah `kubectl` Menolak Terhubung (`The connection to the server ... was refused`)
- **Gejala:** Setiap kali mengetik perintah `kubectl get nodes`, terminal mengeluarkan error penolakan koneksi ke port API server (biasanya port 6443).
- **Penyebab:** Proses `kube-apiserver` sedang mati, atau container server `k3d` sedang dalam posisi berhenti (*stopped*).
- **Diagnosa & Solusi:**
  1. Cek apakah container node server k3d hidup di Docker:
     ```bash
     docker ps -a --filter "name=k3d-lab-server-0"
     ```
  2. Jika statusnya `Exited`, periksa log kematian k3s:
     ```bash
     docker logs k3d-lab-server-0 --tail 50
     ```
  3. Nyalakan kembali cluster dengan `k3d cluster start lab`.

---

### Kasus 2: Node Berstatus `NotReady` pada Output `kubectl get nodes`
- **Gejala:** Salah satu node worker menampilkan status `NotReady` di bawah kolom `STATUS`.
- **Penyebab:** Agen `kubelet` di node tersebut kehilangan koneksi ke `kube-apiserver`, proses `kubelet` mengalami crash, atau disk pada node tersebut kehabisan ruang (*DiskPressure*).
- **Diagnosa SRE:**
  Gunakan perintah describe pada node yang bermasalah dan periksa bagian `Conditions`:
  ```bash
  kubectl describe node <nama-node-yang-bermasalah>
  ```
  Perhatikan baris `MemoryPressure`, `DiskPressure`, dan `PIDPressure`. Jika salah satunya bernilai `True`, bersihkan ruang disk atau naikkan resource node tersebut.

---

### Kasus 3: Pod Terjebak Selamanya di Status `Pending`
- **Gejala:** Kamu membuat deployment baru, tetapi Pod tidak kunjung menyala dan tertahan di status `Pending` tanpa nomor IP node.
- **Penyebab:** Komponen **`kube-scheduler`** tidak berhasil menemukan node worker yang memenuhi syarat (biasanya karena semua node sudah kehabisan sisa alokasi CPU atau RAM).
- **Diagnosa SRE:**
  Selalu lakukan describe pada Pod yang pending dan baca baris **`Events`** paling bawah:
  ```bash
  kubectl describe pod <nama-pod-pending>
  ```
  Kamu akan melihat pesan peringatan dari scheduler, misalnya: `0/3 nodes are available: 3 Insufficient memory`. Solusinya adalah menurunkan angka `resources.requests` pada Pod atau menambah node worker baru ke dalam cluster!

---

## 10. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Pemahaman Arsitektur (Cheat Sheet)

| Komponen Cluster | Lokasi Berada | Tugas Utama Sederhana |
|---|---|---|
| **`kube-apiserver`** | Control Plane | Satu-satunya pintu gerbang REST API & penjaga gerbang `etcd` |
| **`etcd`** | Control Plane | Database penyimpan seluruh konfigurasi & status cluster |
| **`kube-scheduler`** | Control Plane | Menentukan node mana yang paling cocok ditempati oleh Pod baru |
| **`kube-controller-manager`** | Control Plane | Menjaga agar kondisi nyata (*actual*) selalu sama dengan target (*desired*) |
| **`kubelet`** | Worker Node | Mandor di node yang menerima perintah dan memantau container |
| **`containerd`** | Worker Node | Mesin runtime yang mengunduh image dan memutar container Linux |
| **`kube-proxy`** | Worker Node | Mengatur aturan firewall (*iptables*) untuk merutekan trafik Service |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan reflektif berikut untuk menguji ketajaman pemahamanmu tentang arsitektur Kubernetes:

1. **Mengapa komponen `kube-scheduler` dan `kube-controller-manager` tidak pernah diizinkan membaca atau menulis data ke dalam database `etcd` secara langsung? Lewat komponen mana mereka harus berkomunikasi?**
2. **Jika sebuah server Control Plane mati total dan seluruh file database `etcd` di dalamnya terhapus tanpa ada backup, apa yang akan terjadi pada Pod aplikasi yang saat itu sedang berjalan di Worker Node? Apakah aplikasi tersebut langsung mati? Dan apa kendala terbesar yang dihadapi tim SRE setelahnya?**
3. **Jelaskan perbedaan peran antara agen `kubelet` dengan runtime `containerd` di dalam sebuah Worker Node! Siapa yang memerintah dan siapa yang mengeksekusi?**
4. **Mengapa arsitektur cluster Kubernetes production yang menggunakan etcd terdistribusi selalu membutuhkan jumlah server master yang ganjil (3 atau 5 server) dan tidak disarankan menggunakan 2 server?**

---
*Lanjutkan ke materi berikutnya: [02-objek-inti.md](02-objek-inti.md) untuk mempelajari unit-unit bangunan dasar yang akan kamu operasikan setiap hari di Kubernetes: Pod, Deployment, Service, Ingress, ConfigMap, Secret, dan PV/PVC!*