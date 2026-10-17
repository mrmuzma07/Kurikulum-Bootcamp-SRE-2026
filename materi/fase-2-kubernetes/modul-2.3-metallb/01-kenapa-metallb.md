# 01 — Kenapa MetalLB? Membedah Misteri `<pending>` LoadBalancer di Lingkungan Bare-Metal

> **Ringkasan:** Di cloud publik, Service `type=LoadBalancer` otomatis memperoleh IP publik dari infrastruktur penyedia cloud. Di lingkungan on-premises atau bare-metal, Kubernetes tidak memiliki integrasi penyedia cloud bawaan sehingga status IP eksternal tertahan selamanya pada status `<pending>`. MetalLB hadir untuk mengisi kekosongan arsitektural ini dengan menyediakan implementasi LoadBalancer berbasis software menggunakan protokol jaringan standar (ARP/NDP dan BGP).

---

## 1. Masalah Utama: Fenomena Abadi `<pending>` di Bare-Metal

Bagi seorang pemula atau engineer yang terbiasa bekerja di cloud publik (seperti AWS EKS, Google Cloud GKE, atau Azure AKS), mengekspos aplikasi ke dunia luar terasa sangat mudah dan otomatis:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

Ketika manifes di atas di-*apply* di AWS EKS:
1. Komponen internal Kubernetes yang disebut **Cloud Controller Manager (CCM)** mendeteksi Service baru bertipe `LoadBalancer`.
2. CCM memanggil AWS API untuk membuat AWS Network Load Balancer (NLB) atau Application Load Balancer (ALB).
3. Setelah AWS selesai menyiapkan hardware/software load balancer fisiknya, AWS mengembalikan IP publik atau domain DNS ke Kubernetes.
4. Kolom `EXTERNAL-IP` pada `kubectl get svc` langsung terisi dalam beberapa puluh detik:

```text
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP                                                              PORT(S)
web-service   LoadBalancer   10.43.120.45   k8s-elb-1234567890.ap-southeast-1.elb.amazonaws.com                      80:31234/TCP
```

### Apa yang Terjadi di Data Center On-Premises / Bare-Metal?
Ketika Anda mengeksekusi manifes yang persis sama di kluster k3s bare-metal (atau VM Linux di lingkungan on-prem):

```bash
$ kubectl get svc -w
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
web-service   LoadBalancer   10.43.120.45   <pending>     80:31234/TCP   10s
web-service   LoadBalancer   10.43.120.45   <pending>     80:31234/TCP   5m
web-service   LoadBalancer   10.43.120.45   <pending>     80:31234/TCP   3d
```

Kolom `EXTERNAL-IP` akan membeku dalam status `<pending>` untuk selamanya!

> [!IMPORTANT]
> **Status `<pending>` Bukan Menandakan Error atau Kerusakan Kluster!**
> Kubernetes sengaja dirancang secara modular dan agnostik terhadap infrastruktur fisik. Komponen inti Kubernetes (*kube-apiserver*, *kube-controller-manager*, *kubelet*) **sengaja tidak menyertakan kode untuk mengonfigurasi router, switch, atau alokasi IP fisik jaringan kantor/data center Anda**. Kubernetes hanya menyediakan "antarmuka kosong" (*plugin boundary*). Jika tidak ada software yang mengimplementasikan antarmuka tersebut, Kubernetes akan menunggu dengan sabar (*pending*).

---

## 2. Analogi Pemula: Resepsionis Hotel Tanpa Alamat Jalan Fisik

Untuk memudahkan pemahaman mental model pemula:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ANALOGI HOTEL & LOKASI GEDUNG                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ DI LINGKUNGAN CLOUD PUBLIK (EKS / GKE) ]                                 │
│  Gedung hotel dibangun di dalam kawasan superblok modern terintegrasi.       │
│  Ketika pengelola hotel mendirikan meja resepsionis (Service LoadBalancer), │
│  manajemen kawasan superblok langsung menyambungkan pintu gerbang utama ke  │
│  jalan protokol kota, memasang plang neon besar berisikan nomor gedung resmi,│
│  dan menugaskan petugas valet di depan pintu. Tamu dari mana pun bisa       │
│  langsung tiba tanpa kebingungan.                                           │
│                                                                             │
│  [ DI LINGKUNGAN ON-PREMISES TANPA METALLB ]                                │
│  Anda membangun gedung hotel megah di atas tanah milik sendiri di pinggir   │
│  kota. Gedung berdiri kokoh, kamar tidur (Pod) sangat nyaman, meja          │
│  resepsionis (Service) sudah siap melayani. Namun, tanah Anda belum punya   │
│  nomor rumah resmi dari dinas tata kota, dan tidak ada plang penunjuk arah  │
│  dari jalan raya utama.                                                     │
│                                                                             │
│  Hasilnya: Resepsionis menunggu tamu selamanya (<pending>), dan tamu di     │
│  luar pagar tidak tahu ke pintu mana mobil mereka harus diarahkan.          │
│                                                                             │
│  [ DENGAN METALLB ]                                                         │
│  MetalLB bertindak sebagai Dinas Tata Ruang sekaligus Kontraktor Gerbang:   │
│  1. Controller MetalLB mengambil nomor rumah resmi dari buku jatah IP       │
│     (IPAddressPool) dan menempelkannya di plang resepsionis.                │
│  2. Speaker MetalLB berdiri di gerbang depan, memegang megafon (ARP/BGP),   │
│     lalu mengumumkan ke seluruh mobil yang lewat di jalan raya:             │
│     "Perhatian! Rumah Nomor 192.168.97.200 berada di gerbang ini!".         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Spektrum 4 Metode Mengekspos Aplikasi di Kubernetes

Sebelum memahami MetalLB lebih dalam, seorang SRE harus menguasai spektrum 4 cara mengekspos aplikasi di Kubernetes beserta konsekuensi arsitekturalnya:

```text
  Client Luar
      │
      ├─────────────────► [ 1. NodePort ] ──► NodeIP:30080 ────────┐
      │                                                           │
      ├─────────────────► [ 2. MetalLB VIP ] ──► 192.168.97.200:80 ┴─► [ Service ClusterIP ]
      │                                                                       │
      │                                                                       ▼
      └─────────────────► [ 3. Ingress L7 ] ──► app.domain.com:443 ──► [ Pod / Kontainer ]
```

### Tabel Perbandingan Arsitektur Eksternal Access

| Fitur / Parameter | ClusterIP | NodePort | LoadBalancer (MetalLB) | Ingress (L7) |
|---|---|---|---|---|
| **Layer OSI** | Layer 4 (TCP/UDP) | Layer 4 (TCP/UDP) | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| **Aksesibilitas** | Hanya di dalam kluster | Eksternal & Internal | Eksternal & Internal | Eksternal & Internal |
| **Format Alamat** | `10.43.x.x` (IP Virtual) | `<IP-Node>:<Port>` | `<VIP-Pool>:<Port>` | `http(s)://domain.com` |
| **Rentang Port** | Bebas (80, 443, 3306) | Terbatas `30000–32767` | Bebas (80, 443, 53, dll.) | Port standar (80/443) |
| **Kelebihan** | Aman, terisolasi, default | Selalu ada tanpa addon | IP stabil, port standar, VIP | Host/Path routing, SSL TLS |
| **Kekurangan** | Tidak bisa diakses dari luar | Port aneh, membocorkan IP node, tidak HA | Butuh provider (MetalLB) | Butuh LB di depannya |
| **Skenario Ideal** | Database internal, API privat | Debug darurat internal | Pintu gerbang utama on-prem | Web app multi-domain |

### Mengapa NodePort Kurang Layak untuk Standar Production On-Premises?
Sering kali pemula bertanya: *"Jika NodePort selalu bisa dipakai tanpa install apapun, kenapa kita harus repot-repot memasang MetalLB?"*

1. **Port yang Tidak Standar:** Pengguna aplikasi web umum terbiasa mengetik `http://sistem.kantor.id` (port 80). Dengan NodePort, pengguna harus dipaksa mengetik `http://192.168.97.10:31254`. Ini sangat tidak ramah pengguna dan melanggar standar enterprise.
2. **Keterikatan pada Node Tertentu:** Jika Anda mengarahkan DNS atau klien ke IP Node Worker 1 (`192.168.97.10`), ketika Node Worker 1 mengalami *kernel panic* atau mati listrik, klien akan gagal tersambung, meskipun Pod aplikasi Anda sebenarnya masih hidup dan sehat di Node Worker 2.
3. **Keterbatasan Jumlah Port:** Rentang port `30000–32767` hanya menyediakan 2.768 port unik. Port ini menjadi rebutan di seluruh kluster dan rentan konflik (*port collision*).

---

## 4. Arsitektur Inti MetalLB: Controller vs Speaker

MetalLB dirancang dengan pemisahan tugas yang sangat elegan dan modular (*separation of concerns*). MetalLB terdiri dari dua komponen utama:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ARSITEKTUR METALLB                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KUBERNETES CONTROL PLANE ]                                               │
│                                                                             │
│         Kubectl Apply                                                       │
│               │                                                             │
│               ▼                                                             │
│       ┌───────────────┐                                                     │
│       │ kube-apiserver│ ◄─── Menampung Service type=LoadBalancer            │
│       └───────┬───────┘                                                     │
│               │ Watch Event                                                 │
│               ▼                                                             │
│  ┌─────────────────────────┐                                                │
│  │   MetalLB Controller    │  (Berjalan sebagai Deployment: 1 Pod aktif)    │
│  │   (Tukang Alokasi IP)   │                                                │
│  │                         │  Tugas:                                        │
│  │                         │  1. Mendeteksi Service type=LoadBalancer baru  │
│  │                         │  2. Membaca CRD IPAddressPool                  │
│  │                         │  3. Menandai 1 IP kosong (misal 192.168.97.200)│
│  │                         │  4. Menulis IP ke status.loadBalancer.ingress  │
│  │                         │  Catatan: TIDAK MENYENTUH TRAFFIC FISIK!       │
│  └─────────────────────────┘                                                │
│                                                                             │
│ ─────────────────────────────────────────────────────────────────────────── │
│  [ DATA PLANE / NETWORK WORKER NODES ]                                      │
│                                                                             │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │ Node: k3s-cp1           │  │ Node: k3s-w1            │                   │
│  │ ┌─────────────────────┐ │  │ ┌─────────────────────┐ │                   │
│  │ │   MetalLB Speaker   │ │  │ │   MetalLB Speaker   │ │  (DaemonSet:      │
│  │ │   (Duta Jaringan)   │ │  │ │   (Duta Jaringan)   │ │   Berjalan di     │
│  │ └──────────┬──────────┘ │  │ └──────────┬──────────┘ │   setiap node)    │
│  │            │            │  │            │            │                   │
│  │  Leader    │ (ARP/NDP   │  │  Standby   │ (BGP Peer  │  Tugas:           │
│  │  Election  │  Broadcast)│  │            │  Peering)  │  Mengumumkan VIP  │
│  │            ▼            │  │            ▼            │  ke kabel fisik   │
│  │     Ethernet (eth0)     │  │     Ethernet (eth0)     │                   │
│  └────────────┬────────────┘  └────────────┬────────────┘                   │
│               │                            │                                │
│ ══════════════╪════════════════════════════╪═══════════════════════════════ │
│               ▼                            ▼                                │
│       Jaringan Fisik LAN / Switch Bare-Metal / Subnet Mac Host              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1. MetalLB Controller (`deployment/controller`)
- **Bentuk:** Kubernetes Deployment (biasanya 1 replica aktif dengan mekanisme leader election lease).
- **Tanggung Jawab:**
  - Bertindak sebagai otak alokasi (*control-plane allocator*).
  - Melakukan *watch* ke kube-apiserver terhadap Service bertipe `LoadBalancer`.
  - Jika ada Service baru yang belum punya IP, controller memeriksa sumber daya kustom `IPAddressPool`.
  - Memilih satu alamat IP yang masih kosong, lalu memperbarui field `.status.loadBalancer.ingress[0].ip` pada Service tersebut.
  - Jika Service dihapus, controller mengembalikan IP tersebut ke kolam (*pool*) agar dapat digunakan oleh Service lain.
  - **Fakta Penting:** Controller **sama sekali tidak menyentuh paket data jaringan**. Ia murni pengelola status di etcd/Kubernetes.

### 2. MetalLB Speaker (`daemonset/speaker`)
- **Bentuk:** Kubernetes DaemonSet (berjalan tepat 1 Pod di setiap node kluster).
- **Tanggung Jawab:**
  - Bertindak sebagai duta jaringan fisik (*data-plane advertiser*).
  - Berinteraksi langsung dengan network stack host Linux (menggunakan `hostNetwork: true`).
  - Berkomunikasi ke dunia luar menggunakan dua mode operasi:
    1. **Layer 2 Mode (L2):** Menggunakan protokol ARP (*Address Resolution Protocol*) untuk IPv4 atau NDP (*Neighbor Discovery Protocol*) untuk IPv6.
    2. **BGP Mode (L3):** Melakukan *peering* session BGP (port TCP 179) dengan router fisik data center.
  - Menjamin paket jaringan dari klien di luar kluster bisa diarahkan menuju interface node yang tepat, kemudian diteruskan oleh *kube-proxy* ke Pod tujuan.

---

## 5. Mengapa k3d Gagal untuk L2 MetalLB vs Kenapa OrbStack Machine Sukses?

Banyak pemula yang bertanya: *"Mengapa kita tidak memakai k3d saja untuk belajar MetalLB di laptop Mac agar hemat RAM?"*

Jawabannya terletak pada **topologi jaringan Layer 2 dan sifat paket ARP**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│            PERBEDAAN ISOLASI JARINGAN: K3D VS ORBSTACK LINUX VM             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ SKENARIO 1: K3D (DOCKER CONTAINER NETWORK) — GAGAL UNTUK L2 ]            │
│                                                                             │
│   Mac Host Terminal ──( Ping/Curl )──► [ Docker Bridge / NAT Gateway ]      │
│                                                     │                       │
│                                                     ▼ (ARP Tertahan!)       │
│                                        ┌────────────────────────┐           │
│                                        │ Container: k3d-node-1  │           │
│                                        │ Speaker MetalLB        │           │
│                                        └────────────────────────┘           │
│   Masalah: Paket ARP broadcast berhenti di virtual bridge Docker. Mac Host  │
│   tidak pernah mendengar jawaban "192.168.97.200 is at MAC xx". Akibatnya,  │
│   klien Mac tidak bisa membuat sambungan TCP ke VIP!                       │
│                                                                             │
│ ─────────────────────────────────────────────────────────────────────────── │
│                                                                             │
│  [ SKENARIO 2: ORBSTACK LINUX MACHINE (VM) — SUKSES & REALISTIS ]           │
│                                                                             │
│   Mac Host (en0/bridge) ◄═══════════════════════════════════════╗           │
│            ▲                                                    ║           │
│            │ ARP Request: "Who has 192.168.97.200?"             ║ Jaringan  │
│            │ ARP Reply:   "192.168.97.200 is at 02:xx:xx:xx"    ║ L2 Flat / │
│            ▼                                                    ║ Bridge    │
│   OrbStack Linux VM (k3s-cp1 / k3s-w1)                          ║           │
│   Speaker MetalLB menjawab langsung ke Mac Kernel! ═════════════╝           │
│                                                                             │
│   Hasil: macOS kernel mencatat IP 192.168.97.200 di tabel ARP Mac,         │
│   lalu mengirim frame Ethernet langsung ke interface VM. Sesuai 100%        │
│   dengan kondisi fisik server Bare-Metal di Data Center!                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Di k3d, node berjalan sebagai kontainer di dalam namespace Docker. Sementara di **OrbStack Linux Machine (VM)**, setiap node berjalan sebagai kernel Linux independen yang memiliki interface virtual yang langsung terhubung ke network bridge macOS. Ini memungkinkan kita menyimulasikan Layer 2 Broadcast Domain nyata persis seperti switch fisik di rak server.

---

## 6. Apa yang Bukan Tanggung Jawab MetalLB? (Batasan Sistem)

Untuk menghindari kesalahpahaman umum, berikut adalah hal-hal yang **TIDAK** dilakukan oleh MetalLB:

1. **Bukan Ingress Controller (Layer 7):** MetalLB tidak memahami protokol HTTP, URL path (`/api`, `/auth`), ataupun nama domain (`app.perusahaan.com`). MetalLB hanya bekerja di Layer 3/Layer 4 (IP dan Port TCP/UDP). Untuk routing berbasis URL atau domain, Anda tetap membutuhkan Ingress Controller (seperti Traefik atau NGINX Ingress) yang berjalan di belakang Service MetalLB.
2. **Tidak Mengelola DNS Otomatis:** MetalLB tidak membuat record DNS di Cloudflare, Route53, atau bind9 lokal Anda. MetalLB hanya mengalokasikan angka IP (misal `192.168.97.200`). Anda (atau alat seperti *external-dns*) yang bertanggung jawab memetakan nama domain ke IP tersebut.
3. **Bukan Hardware Appliance WAF:** MetalLB tidak memiliki fitur Web Application Firewall (WAF), inspeksi DDoS mitigasi mendalam, atau SSL/TLS Termination. MetalLB murni mekanisme routing IP agar paket dari luar kluster dapat mencapai service di dalam kluster.

```text
  Aliran Lengkap Arsitektur Web Enterprise On-Prem:
  
  Klien Browser (https://pos.retail.id)
       │
       ▼ DNS Query
  DNS Server On-Prem (pos.retail.id ──► 192.168.97.200)
       │
       ▼ Paket IP ke 192.168.97.200:443
  [ MetalLB Layer 4 VIP ]  (Mengarahkan paket ke node k3s)
       │
       ▼ Port 443
  [ Ingress Controller - Traefik/NGINX ] (Terminasi TLS SSL & Baca Host Header)
       │
       ▼ Host: pos.retail.id / Path: /api
  [ Service ClusterIP (pos-backend) ]
       │
       ▼
  [ Pod Kontainer (Spring Boot / Go / Node.js) ]
```

---

## 7. Alur Hidup End-to-End: Dari YAML Hingga Klien Menerima Data

Berikut adalah 7 tahapan siklus hidup saat Anda menerapkan Service LoadBalancer di lingkungan yang telah dipersenjatai MetalLB:

```text
 1. Engineer menjalankan: kubectl apply -f service-loadbalancer.yaml
      │
      ▼
 2. kube-apiserver menyimpan definisi Service; status awal EXTERNAL-IP=<pending>
      │
      ▼
 3. MetalLB Controller mendeteksi Service baru melalui mekanisme Kubernetes Watch API
      │
      ▼
 4. Controller memeriksa resource "IPAddressPool", menemukan IP 192.168.97.200 kosong,
    lalu menuliskan IP tersebut ke dalam Service status (EXTERNAL-IP tidak lagi pending!)
      │
      ▼
 5. MetalLB Speaker di semua node mendeteksi Service telah memiliki IP.
    Melalui protokol internal (Memberlist Gossip), mereka melakukan Leader Election.
    Satu node (misal: k3s-w1) terpilih sebagai "Juru Bicara Resmi" untuk IP 192.168.97.200.
      │
      ▼
 6. Klien di jaringan LAN (misal: Mac Anda) ingin mengakses http://192.168.97.200:
    Mac mengirim paket broadcast ARP: "Siapa pemilik 192.168.97.200?"
    Speaker di k3s-w1 merespons: "192.168.97.200 ada di MAC address saya!"
      │
      ▼
 7. Mac mengirim paket TCP SYN ke MAC address k3s-w1.
    Kube-proxy/iptables di node k3s-w1 menerima paket, lalu meneruskannya ke Pod aplikasi.
    Koneksi HTTP sukses tersambung (HTTP 200 OK)!
```

---

## 8. Verifikasi Lapangan & Prasyarat Kluster

Sebelum melanjutkan ke konfigurasi mendalam di modul berikutnya, mari verifikasi kondisi kluster k3s Anda di terminal Mac:

```bash
# 1. Pastikan context kubectl Anda benar-benar mengarah ke k3s VM (Bukan k3d!)
$ kubectl config current-context
k3s-ha-lab

# 2. Periksa apakah semua node dalam status Ready
$ kubectl get nodes -o wide
NAME      STATUS   ROLES                       AGE   VERSION        INTERNAL-IP     OS-IMAGE             KERNEL-VERSION
k3s-cp1   Ready    control-plane,etcd,master   2d    v1.31.5+k3s1   192.168.97.10   Ubuntu 22.04.5 LTS   6.6.65-orbstack
k3s-cp2   Ready    control-plane,etcd,master   2d    v1.31.5+k3s1   192.168.97.11   Ubuntu 22.04.5 LTS   6.6.65-orbstack
k3s-w1    Ready    worker                      2d    v1.31.5+k3s1   192.168.97.12   Ubuntu 22.04.5 LTS   6.6.65-orbstack

# 3. Pastikan fitur bawaan k3s "servicelb" (Klipper) telah dimatikan!
# Output HARUS KOSONG. Jika ada pod svclb-*, MetalLB akan bentrok!
$ kubectl get pods -A | grep -i svclb || echo "BERSIH: servicelb k3s sudah dinonaktifkan."
BERSIH: servicelb k3s sudah dinonaktifkan.

# 4. Catat subnet dan gateway jaringan Machine OrbStack Anda
$ orb list
NAME       STATE    ARCH    CPU  RAM    DISK
k3s-cp1    running  arm64   2    2 GiB  16 GiB
k3s-cp2    running  arm64   2    2 GiB  16 GiB
k3s-w1     running  arm64   2    2 GiB  16 GiB

$ orb ip k3s-cp1
192.168.97.10
```

> [!WARNING]
> **Bahaya Bentrok dengan `servicelb` (Klipper LB):**
> K3s secara default menyertakan implementasi LoadBalancer minimalis bernama `servicelb`. Pada Modul 2.2, kita telah menonaktifkannya menggunakan flag `--disable servicelb`. Jika `servicelb` masih berjalan saat MetalLB dipasang, kedua controller akan berebut mengalokasikan IP dan port yang sama, menyebabkan kondisi *race condition* yang sangat sulit di-debug.

---

## 9. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Gejala Masalah | Dugaan Penyebab Akar (*Root Cause*) | Perintah Investigasi | Langkah Pemulihan |
|---|---|---|---|
| **Service tetap `<pending>` berjam-jam** | MetalLB Controller belum terpasang, CRD `IPAddressPool` belum dibuat, atau pool kehabisan IP. | `kubectl get pods -n metallb-system`<br>`kubectl get ipaddresspool -A`<br>`kubectl describe svc <nama-svc>` | Pasang MetalLB Controller atau tambahkan rentang IP di `IPAddressPool`. |
| **`EXTERNAL-IP` terisi, tapi `ping`/`curl` timeout dari Mac** | Klien berada di luar subnet L2, atau Speaker MetalLB crash / tidak berjalan di node. | `kubectl get pods -n metallb-system -l component=speaker`<br>`sudo arping -c 3 <VIP>` | Pastikan pod speaker `Running` dan range IP satu subnet dengan Mac host di OrbStack. |
| **Ada dua IP di EXTERNAL-IP atau port bentrok** | `servicelb` bawaan k3s masih hidup berdampingan dengan MetalLB. | `kubectl get pods -A \| grep svclb`<br>`kubectl get svc <nama-svc> -o yaml` | Hapus DaemonSet svclb dan tambahkan `--disable servicelb` di `/etc/rancher/k3s/config.yaml`. |
| **Speaker Pod CrashLoopBackOff** | Port speaker (TCP 7946 untuk memberlist atau BGP 179) terhalang firewall node atau bentrok process lain. | `kubectl logs -n metallb-system -l component=speaker --tail=50` | Periksa konflik port di node atau izin jaringan host (`hostNetwork: true`). |

---

## 10. Ringkasan Konsep Kunci (Cheat Sheet)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                      RINGKASAN CEPAT KENAPA METALLB                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Kenapa Pending?   : Kubernetes core tidak punya kode driver switch fisik. │
│ 2. Apa itu MetalLB?  : Software pemberi IP & pengiklan VIP untuk bare-metal. │
│ 3. Dua Komponen      :                                                      │
│    - Controller      : Memilih & assign IP dari pool (Deployment).          │
│    - Speaker         : Berteriak ke jaringan via ARP atau BGP (DaemonSet).  │
│ 4. Beda dengan L7    : MetalLB = Layer 4 (IP/Port), Ingress = Layer 7 (URL).│
│ 5. Kenapa VM Mac?    : Butuh Layer 2 broadcast ARP tembus ke macOS kernel.  │
│ 6. Syarat Wajib      : servicelb bawaan k3s HARUS di-disable terlebih dahulu!│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Cek Pemahaman Mandiri

Jawablah pertanyaan-pertanyaan berikut untuk menguji pemahaman arsitektural Anda sebelum melangkah ke detail protokol Layer 2:

1. Jelaskan perbedaan mendasar apa yang terjadi di balik layar saat Anda mengeksekusi `kubectl apply -f service-lb.yaml` di AWS EKS dibandingkan di server bare-metal murni tanpa addon!
2. Mengapa komponen MetalLB Controller tidak membutuhkan hak istimewa jaringan host (`hostNetwork: true`), sedangkan komponen MetalLB Speaker mutlak membutuhkannya?
3. Sebutkan dua alasan teknis mengapa seorang SRE tidak disarankan menggunakan Service tipe `NodePort` sebagai pintu masuk utama layanan production enterprise!
4. Jika `kubectl get svc web` sudah menampilkan `EXTERNAL-IP: 192.168.97.200`, tetapi ketika Anda menjalankan `curl http://192.168.97.200` dari Mac hasilnya *Connection Timed Out*, komponen mana yang paling pertama Anda curigai: MetalLB Controller atau MetalLB Speaker? Jelaskan alasannya!
5. Mengapa k3d yang berjalan di atas Docker bridge Mac tidak dapat digunakan untuk menguji fungsionalitas penuh MetalLB mode L2?

---

*Langkah Selanjutnya:* Masuk ke [02 — L2 Mode: ARP/NDP & Advertisement](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.3-metallb/02-l2-mode-arp.md) untuk mempelajari secara rinci bagaimana protokol ARP bekerja, bagaimana mekanisme *leader election* speaker terjadi, dan bagaimana mendiagnosa Layer 2 menggunakan *arping*.