# 03 — BGP Mode: Konsep Arsitektur Routing Production, ECMP, dan Autonomous Systems

> **Ringkasan:** Jika mode L2 mengandalkan broadcast ARP di dalam satu subnet lokal, mode BGP (*Border Gateway Protocol*) mengubah setiap node Kubernetes menjadi "stasiun router" yang secara aktif mengumumkan rute Virtual IP (VIP) ke router fisik data center. Mode ini adalah standar industri enterprise untuk mencapai pembagian beban sejati (*Equal-Cost Multi-Path / ECMP*), skalabilitas lintas-VLAN, dan penghapusan *single-node bottleneck*.

---

## 1. Analogi Pemula: Peta Navigasi Tol Antar-Kota vs Teriakan Megafon di Gang

Untuk memahami lompatan teknologi dari L2 ke BGP, bandingkan dua skenario sistem logistik ini:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PERBANDINGAN ANALOGI LOGISTIK: L2 VS BGP                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ MODE L2: MEGAFON DI GANG SATU PERUMAHAN ]                                │
│  - Kurir masuk ke dalam gang perumahan, lalu berteriak lewat megafon        │
│    (ARP Broadcast): "Siapa pemegang alamat 192.168.97.200?".                │
│  - Masalah: Suara megafon tidak bisa menembus bukit atau pagar beton batas  │
│    kota (Router tidak meneruskan paket broadcast).                          │
│  - Jika perumahan sangat padat dan mobil kurir ribuan per detik, gang      │
│    tersebut akan macet total (Single-Node Bottleneck).                      │
│                                                                             │
│  [ MODE BGP: PETA NAVIGASI GPS SISTEM JALUR TOL PUSAT ]                     │
│  - Setiap server k3s bertindak sebagai "Stasiun Pemancar GPS".              │
│  - Di atas jalan tol utama kota, berdiri Menara Pengendali Tol Pusat        │
│    (Router ToR / Core Switch Data Center).                                  │
│  - Setiap server k3s menghubungi menara tol lewat radio (BGP Session TCP 179)│
│    dan melaporkan: "Halo Menara Tol! Alamat 192.168.100.200 bisa diantar    │
│    lewat pintu keluar saya (Node k3s-w1), atau lewat pintu teman saya       │
│    (Node k3s-w2) dengan waktu tempuh dan tarif yang persis sama!".          │
│  - Menara tol mencatat informasi ini di komputer navigasi gerbang utama     │
│    (Routing Table). Ketika ada truk kontainer (traffic klien) datang dari   │
│    kota mana pun (lintas VLAN/Subnet), gerbang tol membagi mobil bernomor   │
│    genap ke Gate 1 dan mobil ganjil ke Gate 2 (ECMP). Beban terbagi rata!   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Mengapa Enterprise & Data Center Memilih BGP Dibanding L2?

Di lingkungan lab atau jaringan kantor kecil, mode L2 sangat menyenangkan karena tidak membutuhkan konfigurasi perangkat jaringan tambahan. Namun, ketika sebuah organisasi tumbuh menjadi skala data center komersial, arsitektur L2 menghadapi batas fisik (*architectural ceiling*):

```text
  MASALAH PADA LAYER 2 DI DATA CENTER BESAR:
  1. Broadcast Storm: Jutaan paket ARP dari ribuan server membanjiri switch.
  2. Spanning Tree Protocol (STP): Untuk mencegah looping, switch mematikan
     separuh jalur kabel jaringan secara sengaja (pemborosan kapasitas bandwidth!).
  3. Single-Node Ingress Bottleneck: Traffic sebesar 40 Gbps menuju satu VIP
     hanya ditampung oleh 1 port kabel server leader. Node lain menganggur!
  4. Tidak Bisa Menyeberang Router: Kluster di Data Center A tidak bisa diakses
     langsung oleh user di Data Center B tanpa konfigurasi tunnel rumit.
```

Data center modern dibangun dengan topologi **Spine-and-Leaf (Layer 3 Fabric)**. Di arsitektur ini:
- Tidak ada lagi kabel Layer 2 yang membentang antar-rak server.
- Setiap rak server memiliki router tersendiri yang disebut **ToR (Top-of-Rack) Switch**.
- Setiap rak memiliki subnet IP yang berbeda.
- Komunikasi antar-rak dan antar-node diatur sepenuhnya melalui tabel rute IP (*IP Routing Table*).

Di sinilah **MetalLB Mode BGP** menjadi komponen krusial: MetalLB Speaker di setiap node Kubernetes berbicara langsung dengan switch ToR tersebut menggunakan bahasa standar routing internet, yaitu **BGP**.

---

## 3. Glosarium Fundamental BGP untuk SRE Pemula

Sebelum menyentuh manifes YAML, seorang engineer SRE wajib memahami kosakata dasar yang dipakai saat berdiskusi dengan tim Network Engineer (NetOps):

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GLOSARIUM DASAR PROTOKOL BGP                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ 1. BGP (Border Gateway Protocol):                                           │
│    Protokol dinamis Layer 3 yang bertugas saling bertukar informasi rute    │
│    (tabel jalan) antar-router di seluruh dunia dan data center. BGP         │
│    berjalan di atas koneksi TCP port 179.                                   │
│                                                                             │
│ 2. ASN (Autonomous System Number):                                          │
│    Nomor identitas unik sebuah entitas jaringan.                            │
│    - Public ASN: Digunakan oleh ISP dan perusahaan raksasa di internet.     │
│    - Private ASN: Rentang nomor 64512 – 65534 yang dialokasikan khusus     │
│      untuk jaringan internal perusahaan dan data center lokal.              │
│                                                                             │
│ 3. BGP Peering (Session):                                                   │
│    Hubungan pertemanan dua arah antara dua perangkat yang telah disepakati  │
│    untuk saling mengirimkan daftar rute melalui koneksi TCP 179.            │
│                                                                             │
│ 4. eBGP vs iBGP:                                                            │
│    - eBGP (External): Peering antara dua ASN berbeda (misal: Speaker K8s   │
│      di ASN 64512 berbicara ke Switch ToR di ASN 64501). Pola ini adalah   │
│      standar baku integrasi Kubernetes di data center.                      │
│    - iBGP (Internal): Peering di dalam satu nomor ASN yang sama.            │
│                                                                             │
│ 5. Prefix (/32 Host Route):                                                 │
│    Format pengumuman IP tujuan. Untuk satu IP Service VIP (misalnya         │
│    192.168.100.200), MetalLB akan mengumumkan prefix spesifik /32           │
│    (192.168.100.200/32) yang berarti "tepat satu host ini saja".            │
│                                                                             │
│ 6. Next-Hop:                                                                │
│    Alamat IP interface node yang menjadi pintu lompatan paket berikutnya.   │
│                                                                             │
│ 7. ECMP (Equal-Cost Multi-Path):                                            │
│    Fitur hardware router untuk membagi traffic secara bersamaan ke beberapa │
│    jalur rute yang memiliki bobot / nilai biaya (cost) yang sama.           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Cara Kerja ECMP (Equal-Cost Multi-Path) di Balik Layar

Fitur paling bertenaga dari mode BGP adalah **ECMP**. Mari pelajari bagaimana router fisik membagi beban ribuan klien tanpa membuat koneksi TCP menjadi rusak atau kacau:

```text
                        [ Klien Eksternal / Pengguna ]
                                      │
                                      ▼
                      ┌───────────────────────────────┐
                      │    Switch ToR / Core Router   │
                      │    (ASN 64501)                │
                      │                               │
                      │  Tabel Rute untuk VIP .200:   │
                      │  192.168.100.200/32 via:      │
                      │  - Node k3s-cp1 (Hop 1)       │
                      │  - Node k3s-w1  (Hop 2)       │
                      │  - Node k3s-w2  (Hop 3)       │
                      └───────┬───────┬───────┬───────┘
                              │       │       │
              Flow A (Hash 1) │       │       │ Flow C (Hash 3)
                              │       │ Flow B│ (Hash 2)
                              ▼       ▼       ▼
                        ┌────────┐┌────────┐┌────────┐
                        │k3s-cp1 ││ k3s-w1 ││ k3s-w2 │
                        │ Node   ││ Node   ││ Node   │
                        └────────┘└────────┘└────────┘
```

### Bagaimana Router Membagi Traffic Tanpa Merusak Sesi TCP?
Router **tidak melakukan round-robin per paket individual** (karena jika paket nomor 1 dikirim ke Node 1, dan paket nomor 2 dikirim ke Node 2, data TCP akan tiba tidak berurutan dan menyebabkan koneksi putus (*out-of-order packet retransmission*)).

Sebaliknya, router perangkat keras menggunakan algoritma **5-Tuple Hashing**:
1. Router membaca 5 informasi dari setiap paket data:
   - **Source IP** (IP Klien)
   - **Source Port** (Port dinamis Klien)
   - **Destination IP** (VIP MetalLB)
   - **Destination Port** (Port Service, misal 80/443)
   - **Protocol** (TCP atau UDP)
2. Kelima komponen ini dimasukkan ke fungsi hash matematis: `Hash = CRC32(SrcIP, SrcPort, DstIP, DstPort, Protocol)`.
3. Hasil modulo hash tersebut menentukan node mana yang akan dituju.

> [!TIP]
> **Dampak Luar Biasa dari 5-Tuple Hashing:**
> - Seluruh paket dari **satu sesi browsing klien tertentu** akan selalu menghasilkan nilai hash yang identik, sehingga paketnya dijamin **selalu tiba di node yang sama** (koneksi stabil dan aman).
> - Namun, ketika ada **ribuan pengguna baru** yang mengakses dari berbagai IP dan port berbeda, nilai hash mereka akan terdistribusi secara matematis merata ke seluruh node worker kluster! Bandwidth masuk 10 Gbps kluster kini terbagi rata: 3.3 Gbps di Node 1, 3.3 Gbps di Node 2, dan 3.3 Gbps di Node 3.

---

## 5. Sumber Daya Kustom BGP MetalLB: BGPPeer & BGPAdvertisement

Untuk mengaktifkan BGP di MetalLB, kita menggunakan dua Custom Resource Definitions (CRD):

### 1. `BGPPeer` (Mendefinisikan Kawan Bicara Router)
Resource ini memberitahu seluruh Pod Speaker di kluster Anda: siapa router fisik yang harus diajak *peering* dan apa nomor ASN-nya:

```yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: tor-switch-rack-01
  namespace: metallb-system
spec:
  # Nomor ASN lokal milik kluster Kubernetes kita
  myASN: 64512
  # Nomor ASN milik router switch fisik yang diajak bicara
  peerASN: 64501
  # Alamat IP router switch fisik
  peerAddress: 192.168.97.1
  # Port default BGP (179)
  peerPort: 179
  # Berapa lama session bertahan sebelum dianggap mati jika tidak ada kabar (default 90s)
  holdTime: 90s
  # Opsional: Jika menggunakan password proteksi MD5
  # password: "<SECRET_BGP_PASSWORD>"
```

### 2. `BGPAdvertisement` (Mendefinisikan Rute yang Diiklankan)
Resource ini mendefinisikan pool IP mana saja yang harus diumumkan ke router yang sudah diajak *peering*:

```yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: production-bgp-adv
  namespace: metallb-system
spec:
  # Hubungkan dengan IPAddressPool yang telah dibuat sebelumnya
  ipAddressPools:
  - production-pool
  # Panjang prefix yang diumumkan (default: 32 untuk host route individual)
  aggregationLength: 32
```

---

## 6. Efek Spektakuler: Kombinasi BGP + `externalTrafficPolicy: Local`

Inilah rahasia arsitektur berkinerja tinggi (*high performance architecture*) yang diterapkan oleh perusahaan teknologi papan atas dunia:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 BGP DENGAN EXTERNALLTRAFFICPOLICY: LOCAL                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Misalkan kita memiliki 3 Node Worker:                                      │
│  - Node k3s-cp1 : Menjalankan Pod Aplikasi Web (Sehat)                      │
│  - Node k3s-w1  : Menjalankan Pod Aplikasi Web (Sehat)                      │
│  - Node k3s-w2  : TIDAK menjalankan Pod Aplikasi Web (Kosong)               │
│                                                                             │
│  Ketika Service disetel: externalTrafficPolicy: Local                       │
│                                                                             │
│  1. Pod Speaker di k3s-cp1 melihat ada Pod sehat ──► ANNOUNCE rute ke Router│
│  2. Pod Speaker di k3s-w1 melihat ada Pod sehat  ──► ANNOUNCE rute ke Router│
│  3. Pod Speaker di k3s-w2 MELIHAT TIDAK ADA POD! ──► MENARIK DIRI (WITHDRAW)│
│     (Speaker di k3s-w2 TIDAK AKAN mengumumkan rute VIP ke Router ToR!)      │
│                                                                             │
│  HASIL PADA ROUTER HARDWARE:                                                │
│  Router hanya memiliki 2 jalur rute aktif: k3s-cp1 dan k3s-w1.              │
│  Traffic dari pengguna HANYA akan dikirim ke node yang benar-benar          │
│  menjalankan Pod aplikasi!                                                  │
│                                                                             │
│  KEUNTUNGAN MAKSIMAL:                                                       │
│  - Nol latensi extra hop (tidak pernah mampir ke node kosong).              │
│  - IP Klien 100% asli tercatat di log (tidak ada SNAT).                     │
│  - Jika Pod di k3s-cp1 mendadak mati, BGP seketika melakukan WITHDRAW       │
│    dalam hitungan sub-detik sehingga router tidak mengirim paket ke sana.   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Tabel Komparasi Head-to-Head: L2 Mode vs BGP Mode

| Dimensi Perbandingan | Mode Layer 2 (ARP/NDP) | Mode Layer 3 (BGP) |
|---|---|---|
| **Protokol Inti** | ARP Broadcast (IPv4) / NDP (IPv6) | TCP Port 179 (Border Gateway Protocol) |
| **Batas Jaringan Fisik** | Hanya 1 Subnet LAN / Broadcast Domain | Bebas lintas VLAN, lintas Subnet, & Router |
| **Kebutuhan Perangkat** | Switch unmanaged biasa (tanpa config) | Router/Switch ToR yang mendukung fitur BGP |
| **Pola Distribusi Beban** | Single Active (1 Node Leader menelan semua traffic) | Active-Active Multi-Node sejati via ECMP |
| **Bandwidth Maksimum** | Terbatas pada kapasitas 1 port NIC node leader | Terakumulasi di seluruh port NIC worker node |
| **Waktu Pemulihan Failover**| 1 – 5 detik (menunggu Gratuitous ARP) | Sub-detik (jika dipadukan dengan BFD) |
| **Preservasi IP Klien** | Tergantung NAT (`Cluster` vs `Local`) | Sempurna dengan `Local` policy |
| **Kompleksitas Operasi** | Sangat Rendah (Cukup apply YAML MetalLB) | Menengah-Tinggi (Perlu koordinasi tim NetOps) |
| **Risiko Operasional** | IP conflict lokal | *Route leaking* (kesalahan rute skala global) |
| **Skenario Penggunaan** | **Lab Laptop, Kantor Cabang, Kluster Kecil** | **Data Center Enterprise, Private Cloud Tier-1**|

---

## 8. Tata Kelola Keamanan & Risiko Operasional BGP

BGP adalah protokol pengatur lalu lintas tingkat infrastruktur data center. Kesalahan kecil pada konfigurasi BGP dapat berdampak meluas ke luar kluster Kubernetes Anda. Berikut adalah aturan tata kelola yang wajib diterapkan oleh tim SRE:

1. **Selalu Gunakan Private ASN Internal:** Gunakan rentang nomor ASN khusus jaringan privat (misalnya `64512` s.d. `65534`). Jangan pernah menggunakan nomor ASN publik internet tanpa registrasi resmi dari APNIC/IDNIC.
2. **Terapkan Prefix Filtering di Sisi Switch ToR:** Pastikan tim Network Engineer memasang *Route-Map* atau *Prefix-List* di router fisik. Konfigurasikan router agar **hanya menerima pengumuman IP dari blok pool yang telah disepakati** (misalnya hanya `192.168.100.0/24`). Jika ada kesalahan ketik di manifes MetalLB yang mengumumkan `0.0.0.0/0`, filter router akan memblokirnya dan mencegah internet satu kantor menjadi lumpuh (*BGP Route Hijacking*).
3. **Amankan Koneksi BGP dengan TCP MD5:** Pasang kata sandi bersama (*shared secret*) pada sesi BGP antara speaker dan router untuk mencegah pihak tak berwenang menyuntikkan rute palsu ke dalam tabel jaringan.

---

## 9. Mengapa Lab Bootcamp di Mac Menggunakan L2?

Sering kali peserta bootcamp bertanya: *"Jika BGP begitu hebat dan menjadi standar production data center, mengapa di modul berikutnya kita mengonfigurasi L2 di laptop Mac kita?"*

Alasannya sangat praktis dan realistis (*engineering pragmatism*):
- Workstation belajar Anda adalah sebuah laptop **MacBook Air / Apple Silicon** yang menjalankan virtualisasi **OrbStack**.
- Di dalam laptop portabel, Anda tidak memiliki router perangkat keras fisik berharga ratusan juta rupiah (seperti Cisco Nexus, Arista 7050, atau Juniper QFX) yang siap diajak *peering* BGP.
- Menginstal software router virtual tambahan (seperti FRRouting atau VyOS) di laptop akan menghabiskan CPU dan RAM Mac Anda yang berharga tanpa memberikan tambahan nilai praktis yang signifikan.

> [!NOTE]
> **Jalur Pembelajaran Kurikulum SRE:**
> - **Di Lab Laptop (Modul 2.3 & 2.4):** Kita menguasai **Mode L2** untuk membangun pemahaman fundamental alokasi IP, Service LoadBalancer, dan resolusi ARP yang nyata dan dapat diuji dengan `arping`.
> - **Di Dunia Kerja Nyata (Production Bare-Metal):** Prinsip arsitektur BGP, ASN, dan ECMP yang Anda pelajari di modul teori ini akan langsung Anda gunakan saat merancang arsitektur bersama tim Network Data Center.

---

## 10. Observability & Cara Memeriksa Rute BGP

Jika Anda bekerja di data center yang menggunakan mode BGP, berikut adalah perintah operasional standar untuk memvalidasi kesehatan rute:

```bash
# 1. Dari sisi Kubernetes: Periksa status Pod Speaker dan resource BGP
$ kubectl get bgppeers -n metallb-system
NAME                 PEER-ADDRESS   PEER-ASN   MY-ASN   STATUS
tor-switch-rack-01   192.168.97.1   64501      64512    Established

# 2. Periksa log Pod Speaker untuk memverifikasi BGP Handshake
$ kubectl logs -n metallb-system -l component=speaker --tail=50 | grep -i bgp
{"caller":"bgp.go:88","event":"peerEstablished","peer":"192.168.97.1","msg":"BGP session established"}

# 3. Dari sisi Router ToR (CLI Cisco / FRRouting / Arista):
# router# show ip bgp summary
# router# show ip route 192.168.100.200
# Routing entry for 192.168.100.200/32
#   Known via "bgp 64501", distance 20, metric 0
#   Multipath: Equal-Cost Multi-Path (ECMP)
#     Paths: (3 available, best #1)
#       192.168.97.10 (Node k3s-cp1)
#       192.168.97.11 (Node k3s-cp2)
#       192.168.97.12 (Node k3s-w1)
```

---

## 11. Ringkasan Konsep Kunci (Cheat Sheet BGP)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       RINGKASAN CEPAT BGP MODE METALLB                      │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Mekanisme Utama   : Pengumuman rute IP (/32) ke router via TCP port 179. │
│ 2. Keunggulan No. 1  : ECMP — traffic masuk dibagi rata ke banyak node.     │
│ 3. Tanpa Batas Subnet: Bisa diakses lintas VLAN/Data Center tanpa L2 bridge.│
│ 4. CRD Utama         : BGPPeer (siapa router) & BGPAdvertisement (apa IP).  │
│ 5. Formula Emas      : BGP + externalTrafficPolicy: Local = Nol latensi hop │
│                        dan IP klien asli 100% dipertahankan!                │
│ 6. Keamanan Wajib    : Gunakan Private ASN (64512+) & pasang Prefix Filter. │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Cek Pemahaman Mandiri

1. Mengapa mode Layer 2 MetalLB rentan mengalami *single-node ingress bottleneck*, sedangkan mode BGP dapat mengatasinya secara elegan menggunakan fitur ECMP?
2. Bagaimana router fisik menggunakan algoritma *5-Tuple Hashing* untuk mendistribusikan traffic ke berbagai node k3s tanpa menyebabkan paket TCP satu sesi menjadi berantakan?
3. Jelaskan apa yang terjadi pada pengumuman rute BGP ketika sebuah Service dikonfigurasi dengan `externalTrafficPolicy: Local`, dan salah satu node k3s tidak memiliki Pod aplikasi yang sedang berjalan!
4. Apa yang dimaksud dengan *Private ASN*, dan mengapa kita tidak boleh sembarangan menggunakan nomor ASN publik saat mengonfigurasi BGP di jaringan internal kantor?
5. Mengapa kesalahan konfigurasi BGP (*route leak*) dianggap jauh lebih berisiko bagi stabilitas jaringan data center dibandingkan kesalahan konfigurasi mode L2?

---

*Langkah Selanjutnya:* Masuk ke [04 — Konfigurasi & Integrasi MetalLB dengan k3s](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.3-metallb/04-konfigurasi-integrasi-k3s.md) untuk memulai implementasi langsung, langkah demi langkah, dari pemasangan manifest resmi hingga verifikasi Virtual IP dari terminal Mac Anda.
