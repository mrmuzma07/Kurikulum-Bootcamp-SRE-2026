# 04 — Perancangan Topologi Jaringan On-Premises di OrbStack

> **Ringkasan:** Membangun pola pikir arsitek infrastruktur on-premises: beralih dari kemanjaan otomatisasi cloud menuju realitas data center mandiri, merancang arsitektur **4 Lapisan Akses Jaringan**, memahami pembagian alokasi **Subnet IP** agar terbebas dari bencana tabrakan IP (*IP Collision*), menelusuri aliran paket data dari peramban laptop menuju Pod (*End-to-End Packet Flow*), serta menyusun dokumen cetak biru topologi resmi sebagai fondasi otomasi **Ansible** di Fase 4.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan menyelesaikan praktiknya di terminal, kamu akan mampu:
- Menjelaskan perbedaan mendasar pola pikir arsitektur jaringan antara **Cloud Terkelola (AWS EKS/GCP GKE)** vs **On-Premises Bare-Metal**.
- Menjelaskan mengapa alamat IP dinamis (DHCP tanpa reservasi) adalah musuh utama stabilitas cluster Kubernetes dan bagaimana **IP Statis** diimplementasikan di dunia nyata.
- Merancang dan memetakan arsitektur **4 Lapisan Akses Jaringan** (*External Client*, *Load Balancer VIP*, *Ingress Controller*, dan *Internal Cluster Service*).
- Membuat perencanaan pembagian rentang IP (*Subnet IP Planning*) yang memisahkan antara IP Node, IP Pod Overlay, IP ClusterIP, dan Pool Alamat MetalLB.
- Menelusuri perjalanan paket data langkah demi langkah dari saat pengguna mengetik URL di peramban hingga dijawab oleh Pod aplikasi.
- Menyusun dokumen spesifikasi topologi resmi (**`topologi.md`**) yang siap dikonversi menjadi file *inventory* otomasi Ansible di Fase 4.

---

## 1. Konsep Dasar: Menjadi Insinyur Jaringan di Lingkungan On-Premises

### Kemanjaan Cloud Publik vs Realitas Data Center Mandiri
Jika kamu mendeploy aplikasi ke layanan cloud terkelola seperti **Amazon EKS** atau **Google GKE**, kamu dimanjakan oleh otomatisasi penuh penyedia cloud:
- Ingin Load Balancer publik? Cukup tulis `type: LoadBalancer`, dalam 1 menit AWS otomatis membuatkan perangkat keras *Elastic Load Balancing (ELB)* dengan Alamat IP Publik.
- Ingin nama domain otomatis? AWS Route53 langsung menyambungkannya.
- Ingin harddisk persisten? AWS EBS langsung mencantolkannya secara otomatis.

**Namun di lingkungan On-Premises (Private Data Center, Pabrik, Perbankan, atau Bare-Metal):**
**TIDAK ADA TOMBOL AJAIB DARI CLOUD PROVIDER!**
Di lingkungan on-prem, **KITA ADALAH PENYEDIA INFRASTRUKTURNYA (*You Are The Cloud Provider*)**:
1. Kita yang menarik kabel LAN fisik dan mengonfigurasi VLAN di switch data center.
2. Kita yang membagi jatah subnet Alamat IP agar tidak terjadi tabrakan (*collision*).
3. Kita yang wajib menyediakan dan memasang sistem Load Balancer bare-metal sendiri (**MetalLB di Modul 2.3**).
4. Kita yang menyetel server DNS lokal dan sistem penyimpanan bersama (*Shared Storage*).

```text
 PERBANDINGAN PERAN INFRASTRUKTUR:
 ┌──────────────────────────┐         ┌──────────────────────────┐
 │     CLOUD PUBLIK         │         │   ON-PREMISES BARE-METAL │
 ├──────────────────────────┤         ├──────────────────────────┤
 │ Load Balancer : AWS ELB  │ ◄──VS──►│ Load Balancer : MetalLB  │
 │ DNS Publik    : Route 53 │         │ DNS Lokal     : CoreDNS  │
 │ IP Publik     : Otomatis │         │ IP Statis     : Netplan  │
 │ Storage Cloud : AWS EBS  │         │ Storage Lokal : Longhorn │
 └──────────────────────────┘         └──────────────────────────┘
```

> 🏢 **Peran OrbStack sebagai "Mini Data Center":**
> Di Bootcamp SRE 2026 ini, kita tidak menggunakan AWS yang mahal dan mengaburkan pemahaman fundamental jaringan. Kita menyimulasikan seluruh rak server data center tersebut di atas laptop MacBook menggunakan **OrbStack Linux Machines**:
> - Setiap mesin virtual (VM) bertindak layaknya satu rak server fisik (*bare-metal rack*).
> - Jaringan bridge virtual OrbStack bertindak layaknya switch jaringan fisik di data center.
> - Alamat IP stabil OrbStack (`198.19.249.x`) menjadi simulasi dari Alamat IP Statis kabel LAN data center!

---

## 2. Musuh Terbesar Kubernetes On-Prem: Alamat IP yang Berubah-ubah!

Salah satu penyebab paling sering hancurnya cluster Kubernetes on-premise milik pemula adalah membiarkan server menggunakan **DHCP Biasa Tanpa Reservasi**.

### Mengapa IP yang Berganti Berakibat Fatal bagi Kubernetes?
1. **Sertifikat TLS API Server Terkunci pada IP:** Saat k3s diinstal, sertifikat enkripsi SSL dibuat khusus untuk mengikat Alamat IP server master saat itu (melalui `--tls-san`). Jika besok pagi router memberikan IP baru ke server master, sertifikat akan ditolak dan `kubectl` tidak bisa terhubung!
2. **Sinkronisasi etcd Peer Terputus:** Anggota database `etcd` saling memanggil menggunakan Alamat IP server. Jika IP server 2 berganti, server 1 dan server 3 mengira server 2 mati, sehingga kuorum terancam pecah!
3. **Otomasi Ansible Lumpuh:** File inventaris server yang dicatat oleh tim SRE menjadi tidak valid karena alamat targetnya salah sasaran.

### Tiga Cara Menetapkan IP Statis di Dunia Nyata:
1. **Static IP Manual via Netplan (Server Fisik / Bare-Metal):** Mengedit file `/etc/netplan/01-netcfg.yaml` di Ubuntu dan mengunci IP secara permanen.
2. **DHCP Reservation / Static Lease (VM di Proxmox/VMware):** Mengunci alamat IP pada router switch berdasarkan *MAC Address* kartu jaringan server.
3. **OrbStack Machine (Simulasi Lab Kita):** OrbStack secara otomatis memberikan IP privat yang stabil dan tidak berubah meskipun mesin di-stop, di-start, atau laptop di-reboot.

---

## 3. Arsitektur 4 Lapisan Akses Jaringan (The 4 Access Layers)

Ketika seorang pengguna di luar cluster ingin membuka aplikasi web kita, paket data harus melewati **4 lapisan gerbang jaringan** secara berurutan:

```text
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ LAPISAN 1: KLIEN EKSTERNAL (Laptop Mac Host / Browser Pengguna)         │
 │ Pengguna membuka: http://portal.sre.local:80                             │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │ 1. DNS resolve ke 198.19.249.200
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ LAPISAN 2: BARE-METAL LOAD BALANCER (MetalLB VIP - Modul 2.3)           │
 │ Alamat IP Virtual (VIP): 198.19.249.200 (Mengambang via protokol ARP L2)│
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │ 2. Diteruskan ke Node penanggung jawab
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ LAPISAN 3: INGRESS CONTROLLER (Traefik / Ingress-Nginx)                 │
 │ Membaca Layer 7 HTTP Host Header: "Host: portal.sre.local"             │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │ 3. Merutekan ke Service backend yang pas
                                      ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ LAPISAN 4: INTERNAL CLUSTER (Service ClusterIP & Pod)                   │
 │ Service: portal-svc (10.43.100.50) ──► Pod Aplikasi (10.42.1.15)        │
 └─────────────────────────────────────────────────────────────────────────┘
```

### Penjelasan Peran Setiap Lapisan:
1. **Lapisan 1 (External Client):** Laptop Mac atau peramban pengguna yang berada di luar cluster. Klien ini hanya mengetahui nama domain (misal: `portal.sre.local`).
2. **Lapisan 2 (Load Balancer VIP):** Titik kontak fisik pertama. Alamat IP Virtual (VIP) mandiri yang dialokasikan oleh **MetalLB**. Alamat ini berada di subnet lokal data center sehingga bisa dijangkau langsung oleh laptop Mac.
3. **Lapisan 3 (Ingress Controller):** Petugas pemeriksa tiket di pintu masuk. Membaca protokol HTTP Layer 7 untuk membedakan apakah pengguna ingin menuju `/api` atau `/web`, lalu meneruskannya ke Service yang bersangkutan.
4. **Lapisan 4 (Internal Service & Pod):** Jaringan privat di dalam cluster yang menghubungkan Pod-Pod aplikasi.

---

## 4. Perencanaan Rentang Alokasi Subnet IP (IP Subnet Planning)

Salah satu keahlian wajib seorang SRE on-premises adalah merencanakan pembagian alokasi IP agar tidak terjadi **Tabrakan Alamat IP (*IP Collision / ARP Poisoning*)**.

Di lingkungan laboratorium OrbStack kita, seluruh mesin virtual berada di subnet kelas C privat (biasanya di rentang **`198.19.249.0/24`**). Kita membaginya menjadi 4 wilayah kekuasaan yang terisolasi ketat:

```text
 SUBSET RENTANG JARINGAN (SUBNET 198.19.249.0/24):
 ├── 198.19.249.1  - 198.19.249.9   ➔ Jatah Gateway & Router Host macOS
 ├── 198.19.249.10 - 198.19.249.30  ➔ JATAH NODE FISIK / VM ORBSTACK (Static)
 ├── 198.19.249.31 - 198.19.249.199 ➔ Ruang Cadangan Penambahan Server Masa Depan
 └── 198.19.249.200 - 198.19.249.250➔ JATAH POOL METALLB (Alamat IP Virtual / VIP)
```

| Wilayah Jaringan | Rentang Alamat IP | Siapa Pengelolanya? | Keterangan Teknis |
|---|---|---|---|
| **Node VM Jaringan Fisik** | `198.19.249.10 – .30` | OrbStack / Sysadmin | Alamat IP fisik mesin Ubuntu server (`k3s-cp1`, `k3s-w1`, dll). |
| **MetalLB VIP Pool** | `198.19.249.200 – .250` | **MetalLB Controller** | **Alamat IP Virtual publik cluster.** Harus di subnet yang sama dengan node agar bisa di-ping dari Mac, tetapi **DILARANG tumpang tindih dengan IP node mana pun!** |
| **Service ClusterIP** | `10.43.0.0/16` | Internal k3s | Jaringan virtual internal k8s. Hanya ada di dalam memori kernel via iptables. |
| **Pod Overlay Network** | `10.42.0.0/16` | CNI Flannel (k3s) | Jaringan virtual enkapsulasi VXLAN tempat masing-masing Pod bernaung. |

> 💥 **Bencana Tabrakan IP (IP Collision):**
> Jika kamu tidak sengaja memasukkan alamat IP milik node `k3s-cp1` (misal `198.19.249.10`) ke dalam daftar pool MetalLB, maka ketika ada Service yang meminta IP, MetalLB akan meminjamkan IP `.10` tersebut!
> Akibatnya: Dua mesin berbeda akan mengaku memiliki IP yang sama di jaringan LAN lokal. Terjadi kekacauan protokol ARP (*ARP Collision*), koneksi SSH ke master server akan terputus, dan trafik web akan terlempar secara acak!

---

## 5. Melacak Aliran Paket Data (*End-to-End Packet Flow*)

Mari kita telusuri secara presisi apa yang terjadi dalam hitungan milidetik ketika kamu membuka peramban di laptop Mac dan mengetik: **`http://portal.sre.local`**:

```text
 1. Resolusi Nama Domain (Mac OS):
    Laptop Mac membaca file /etc/hosts ➔ portal.sre.local = 198.19.249.200 (VIP MetalLB).
    │
    ▼
 2. Siapa Pemilik IP Ini? (Protokol ARP Layer 2):
    Mac menyiarkan pertanyaan ARP ke jaringan lokal: "Siapa pemilik IP 198.19.249.200?"
    Salah satu node worker (misal k3s-w1) menjawab: "Saya penanggung jawab IP tersebut! (MAC: xx:xx)".
    │
    ▼
 3. Paket Tiba di Pintu Gerbang Node (Node k3s-w1):
    Paket tiba di kartu jaringan k3s-w1 pada Port 80.
    Aturan iptables / kube-proxy di kernel Linux meneruskan paket ke Pod Ingress Controller.
    │
    ▼
 4. Pemeriksaan Tiket HTTP Layer 7 (Ingress Controller):
    Ingress Controller (Traefik / Nginx) membaca Host Header: "portal.sre.local".
    Melihat aturan: Host ini harus diteruskan ke Service internal 'portal-svc' di Port 8080.
    │
    ▼
 5. Pembagian Beban Internal (Kubernetes Service):
    Service memilih salah satu Pod aplikasi yang sehat dari daftar Endpoints (misal Pod di k3s-w2).
    │
    ▼
 6. Jawaban Dikirim Balik ke Pengguna:
    Pod memproses permintaan, menghasilkan HTML/JSON, lalu mengirimkannya kembali ke laptop Mac
    melalui rute pembalikan yang sama. Halaman web berhasil tampil di layar!
```

---

## 6. Cetak Biru Dokumentasi Topologi Resmi SRE (`topologi.md`)

Di dunia kerja nyata, seorang SRE wajib mendokumentasikan topologi infrastruktur yang dibangunnya. Dokumen ini bukan sekadar catatan pasif, melainkan menjadi acuan mutlak bagi tim keamanan dan **bahan baku penyusunan file *Inventory* Ansible pada Fase 4**.

Berikut adalah standar format dokumentasi topologi resmi yang wajib kamu buat di dalam repositorimu (`materi/fase-2-kubernetes/modul-2.2-k3s-production/topologi.md`):

```markdown
# Dokumentasi Topologi Cluster k3s High Availability On-Premises

## 1. Daftar Node Infrastruktur (Server VM)
| Nama Host | Alamat IPv4 | Peran Cluster | CPU / RAM | Sistem Operasi |
|---|---|---|---|---|
| **k3s-cp1** | 198.19.249.10 | Control Plane + Embedded etcd (Init) | 2 Core / 2 GB | Ubuntu 24.04 LTS (ARM64) |
| **k3s-cp2** | 198.19.249.11 | Control Plane + Embedded etcd (Join) | 2 Core / 2 GB | Ubuntu 24.04 LTS (ARM64) |
| **k3s-cp3** | 198.19.249.12 | Control Plane + Embedded etcd (Join) | 2 Core / 2 GB | Ubuntu 24.04 LTS (ARM64) |
| **k3s-w1**  | 198.19.249.20 | Worker Agent (Beban Kerja Pod)       | 2 Core / 2 GB | Ubuntu 24.04 LTS (ARM64) |
| **k3s-w2**  | 198.19.249.21 | Worker Agent (Beban Kerja Pod)       | 2 Core / 2 GB | Ubuntu 24.04 LTS (ARM64) |

## 2. Alokasi Rentang Subnet Jaringan
- **Subnet Jaringan Fisik Node:** `198.19.249.0/24` (Gateway: `198.19.249.1`)
- **Alokasi IP Node Server:** `198.19.249.10 – 198.19.249.30`
- **Pool Alamat MetalLB (VIP):** `198.19.249.200 – 198.19.249.250` (Tersedia 51 Alamat VIP)
- **Subnet Virtual Service k8s (ClusterIP):** `10.43.0.0/16` (Bawaan k3s)
- **Subnet Virtual Pod k8s (Overlay Flannel):** `10.42.0.0/16` (Bawaan k3s)

## 3. Kebijakan Komponen Cluster
- **ServiceLB (Klipper):** Dinonaktifkan (`--disable servicelb`) untuk memberi jalan pada MetalLB di Modul 2.3.
- **Traefik Ingress:** Aktif / Dinonaktifkan (Sesuai kebutuhan pengujian IngressClass).
- **CoreDNS & Local Path Storage:** Aktif penuh.
```

---

## 7. Hands-On Mandiri Step-by-Step (15 Menit)

Mari kita uji konektivitas topologi jaringan on-premise ini dari laptop Mac milikmu:

```bash
# --- LANGKAH 1: Verifikasi Alamat IP Seluruh Node VM ---
echo "Memeriksa ketersediaan IP stabil node:"
for node in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do
  echo "$node ➔ IP: $(orb ip $node 2>/dev/null || echo 'Belum dibuat')"
done

# --- LANGKAH 2: Menguji Latensi Jaringan Antarnode ---
# Kirim paket ICMP ping dari k3s-cp1 menuju k3s-w1:
CP1_IP=$(orb ip k3s-cp1)
W1_IP=$(orb ip k3s-w1)

ssh k3s-cp1 "ping -c 3 ${W1_IP}"

# --- LANGKAH 3: Simulasi Akses Sementara via NodePort Sebelum MetalLB ---
# Terapkan aplikasi nginx pada namespace default:
kubectl create deployment web-topologi --image=nginx:1.27-alpine
kubectl expose deployment web-topologi --port=80 --type=NodePort

# Ambil nomor NodePort acak yang dialokasikan (rentang 30000-32767):
NODE_PORT=$(kubectl get svc web-topologi -o jsonpath='{.spec.ports[0].nodePort}')
echo "Aplikasi dapat diakses sementara di seluruh IP Node pada Port: ${NODE_PORT}"

# Uji akses dari Mac langsung ke IP k3s-w1 dan k3s-w2:
curl -I "http://${W1_IP}:${NODE_PORT}"
curl -I "http://$(orb ip k3s-w2):${NODE_PORT}"

# Bersihkan deployment uji coba:
kubectl delete svc web-topologi
kubectl delete deployment web-topologi
```

---

## 8. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 masalah klasik dalam perancangan topologi jaringan on-premise beserta solusinya:

### Kasus 1: Akses Layanan Putus-Nyambung (*Intermittent Network Drops*)
- **Gejala:** Saat melakukan curl ke IP Load Balancer, 1 kali berhasil, lalu 2 kali *Connection Timed Out*, lalu berhasil lagi secara bergantian.
- **Penyebab:** Terjadi **Tabrakan Alamat IP (*IP Conflict*)**! Ada dua perangkat atau dua node yang sama-sama mengklaim Alamat IP tersebut di tabel ARP router.
- **Diagnosa SRE:**
  Periksa tabel tetangga ARP dari terminal Mac atau node Linux:
  ```bash
  arp -a | grep <IP-yang-bermasalah>
  ```
  Jika muncul pesan *duplicate IP address detected* atau alamat MAC berubah-ubah, segera ganti rentang pool MetalLB agar berada di luar jatah IP perangkat lain.

---

### Kasus 2: Domain `app.sre.local` Gagal Dibuka di Browser Mac
- **Gejala:** Browser menampilkan pesan error `DNS_PROBE_FINISHED_NXDOMAIN`.
- **Penyebab:** Kamu lupa mendaftarkan pemetaan nama domain tersebut ke file resolver `/etc/hosts` di laptop Mac milikmu.
- **Solusi:** Buka `/etc/hosts` di Mac dan tambahkan baris pemetaan ke Alamat IP tujuan:
  ```bash
  echo "<ALAMAT_IP_VIP> app.sre.local" | sudo tee -a /etc/hosts
  ```

---

### Kasus 3: Node Pod Tidak Bisa Mengakses Internet Luar (*Egress Blocked*)
- **Gejala:** Container di dalam Pod gagal melakukan `curl https://google.com` atau gagal mengunduh dependensi dari internet.
- **Penyebab:** Konfigurasi *IP Forwarding* di kernel sistem operasi server Linux dalam keadaan mati (`net.ipv4.ip_forward = 0`).
- **Solusi SRE:** Aktifkan IP forwarding di dalam seluruh VM:
  ```bash
  sudo sysctl -w net.ipv4.ip_forward=1
  echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/99-kubernetes.conf
  ```

---

## 9. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Topologi Jaringan (Cheat Sheet)

| Kebutuhan Perancangan | Aturan Praktis SRE |
|---|---|
| **Alamat IP Node Fisik/VM** | Wajib **Statis** (tidak boleh berubah saat reboot) |
| **Alokasi Pool MetalLB** | Wajib di **Subnet LAN yang sama**, tetapi di luar jatah IP Node |
| **Subnet Pod & Service** | Ditentukan otomatis oleh k3s (`10.42.0.0/16` & `10.43.0.0/16`) |
| **Akses HTTP Domain** | Klien ➔ MetalLB VIP ➔ Ingress ➔ Service ➔ Pod |
| **Dokumen Topologi** | `topologi.md` menjadi acuan inventory Ansible di Fase 4 |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan reflektif berikut untuk menguji kesiapan arsitekturalmu sebelum memasuki laboratorium MetalLB:

1. **Jelaskan perbedaan mendasar antara cara kerja perolehan Alamat IP LoadBalancer di Cloud Publik (seperti AWS EKS) dibandingkan di lingkungan On-Premises bare-metal data center!**
2. **Mengapa alamat IP dari Pool MetalLB harus berada pada subnet LAN yang sama dengan node-node k3s, namun dilarang keras menggunakan angka IP yang sudah dipakai oleh server fisik? Apa bahaya fatal jika terjadi tumpang tindih?**
3. **Urutkan dan jelaskan 4 lapisan akses jaringan (*The 4 Access Layers*) yang dilalui oleh sebuah paket data dari saat pengguna menekan tombol Enter di browser Mac hingga dijawab oleh Pod backend!**
4. **Mengapa dokumen perancangan topologi jaringan (`topologi.md`) menjadi artefak kerja yang sangat berharga bagi seorang SRE saat mempersiapkan otomatisasi infrastruktur menggunakan Ansible di Fase 4?**

---
*Selamat! Kamu telah menguasai seluruh materi teori Modul 2.2. Lanjutkan ke laboratorium praktik: [lab/LAB-01-k3s-single-node-vm.md](lab/LAB-01-k3s-single-node-vm.md) dan [lab/LAB-02-k3s-multi-node-topologi.md](lab/LAB-02-k3s-multi-node-topologi.md) untuk mulai membangun cluster High Availability secara nyata di atas VM OrbStack!*