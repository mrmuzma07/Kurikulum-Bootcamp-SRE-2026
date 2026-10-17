# 02 — L2 Mode: Cara Kerja ARP/NDP, Leader Election, dan Batasan Layer-2

> **Ringkasan:** Mode Layer 2 (L2) adalah cara paling sederhana dan populer untuk mengoperasikan MetalLB di jaringan lokal. Pada mode ini, satu node terpilih menjadi "pemimpin" (*leader*) yang bertanggung jawab menjawab kueri ARP (untuk IPv4) atau NDP (untuk IPv6) atas nama Virtual IP (VIP) Service. Pelajari cara kerja resolusi alamat, mekanisme *leader election*, perbedaan kebijakan traffic `Cluster` vs `Local`, serta cara melakukan diagnosa bertingkat menggunakan *arping*.

---

## 1. Analogi Pemula: Megafon Ketua RT di Komplek Perumahan

Bagi orang awam, istilah "Layer 2", "ARP", dan "MAC Address" terdengar sangat abstrak. Mari kita gunakan analogi kehidupan sehari-hari:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PENGIRIMAN PAKET & MEGAFON KETUA RT                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Bayangkan sebuah komplek perumahan (Satu Subnet LAN / Broadcast Domain):   │
│  - Setiap rumah memiliki Nomor KTP Pemilik yang unik dan permanen di dunia  │
│    (ini adalah MAC Address fisik kartu jaringan).                           │
│  - Setiap rumah juga diberi Nomor Rumah yang bisa berganti sesuai kavling   │
│    (ini adalah Alamat IP).                                                  │
│                                                                             │
│  1. Datang seorang kurir pos (Klien Luar/Mac Anda) membawa surat tertuju    │
│     pada nomor rumah "192.168.97.200" (Virtual IP Service).                 │
│  2. Kurir ini belum tahu nomor KTP fisik dari pemilik rumah tersebut.       │
│     Maka kurir menyalakan megafon di pintu gerbang komplek dan berteriak:   │
│     "Halo seluruh warga komplek! Siapa yang memiliki nomor rumah           │
│      192.168.97.200?" (Ini adalah paket ARP Request Broadcast).            │
│  3. Seluruh warga komplek (Node k3s-cp1, k3s-cp2, k3s-w1) mendengar        │
│     teriakan tersebut. Namun, mereka sebelumnya sudah rapat RT              │
│     (Leader Election).                                                      │
│  4. Hasil rapat RT menetapkan bahwa Bapak k3s-w1 bertindak sebagai wakil    │
│     resmi rumah nomor 192.168.97.200.                                       │
│  5. Bapak k3s-w1 langsung menyahut lewat jendela:                           │
│     "Saya yang memegang nomor itu! Kirim ke KTP saya AA:BB:CC:DD:EE:03!"    │
│     (Ini adalah paket ARP Reply Unicast).                                   │
│  6. Kurir mencatat nomor KTP tersebut di buku sakunya (ARP Cache), lalu     │
│     menyerahkan surat langsung ke tangan Bapak k3s-w1.                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Mengapa IP Membutuhkan MAC Address? (Dasar Layer 2)

Di dunia jaringan komputer model OSI (*Open Systems Interconnection*):
- **Layer 3 (Network Layer):** Mengenal alamat logis seperti **IP Address** (`192.168.97.200`). Alamat ini berfungsi untuk menentukan rute pengiriman global antar-jaringan.
- **Layer 2 (Data Link Layer):** Bekerja pada kabel fisik tembaga, fiber optik, atau Wi-Fi. Kartu jaringan (*Network Interface Card / NIC*) **hanya bisa menerima frame data yang dialamatkan ke MAC Address perangkat kerasnya** (misal: `02:50:f2:8a:11:03`).

Ketika komputer Mac Anda ingin mengirim data ke IP `192.168.97.200`, sistem operasi macOS tidak bisa langsung menaruh bit-bit data ke kabel tanpa mengisi kolom **Destination MAC Address** di header Ethernet Frame.

```text
  [ Ethernet Frame Fisik ]
  ┌──────────────────────┬────────────────────┬──────────────────────┬───────────┐
  │ Preamble             │ Destination MAC    │ Source MAC           │ Data (IP) │
  │ (Sinkronisasi Sinyal)│ ?? : ?? : ?? : ??  │ 3c:22:fb:12:44:90    │ TCP Syn.. │
  └──────────────────────┴────────────────────┴──────────────────────┴───────────┘
                                   ▲
                                   │ Harus diisi sebelum frame bisa dikirim!
```

### Bagaimana macOS Mengetahui MAC Address dari IP Tujuan?
Melalui protokol **ARP (Address Resolution Protocol)** untuk IPv4, atau **NDP (Neighbor Discovery Protocol)** untuk IPv6.

```text
   Mac Host (192.168.97.1)                      Node k3s-w1 (192.168.97.12)
          │                                                  │
          │ ─── 1. ARP Request (Broadcast ke ff:ff:ff:ff) ──►│ "Who has 192.168.97.200?
          │        (Didengar oleh SEMUA perangkat di LAN)    │  Tell 192.168.97.1"
          │                                                  │
          │ ◄── 2. ARP Reply (Unicast langsung ke Mac) ──────│ "192.168.97.200 is at
          │                                                  │  02:50:f2:8a:11:03"
          ▼                                                  ▼
   Disimpan di macOS:                                 Mulai menerima frame
   $ arp -a                                           TCP port 80/443
   192.168.97.200 at 02:50:f2:8a:11:03 on bridge100
```

> [!IMPORTANT]
> **Batasan Mutlak Layer 2 (Broadcast Domain):**
> Paket ARP Request bertipe **Broadcast** (`ff:ff:ff:ff:ff:ff`). Router standar **TIDAK AKAN PERNAH meneruskan paket broadcast keluar dari subnet asalnya**. Oleh karena itu, pada MetalLB mode L2, kumpulan IP yang dialokasikan (*IPAddressPool*) **wajib berada dalam satu subnet fisik/L2 yang sama** dengan klien atau gateway yang mengaksesnya.

---

## 3. Cara Kerja L2 Mode di Balik Layar

Mari kita bedah apa yang terjadi di dalam kluster k3s Anda ketika mode L2 MetalLB diaktifkan:

```text
 ┌───────────────────────────────────────────────────────────────────────────┐
 │                            KUBERNETES CLUSTER                             │
 │                                                                           │
 │  Service: web (Type: LoadBalancer) ──► IP Dialokasikan: 192.168.97.200    │
 ├───────────────────────────────────────────────────────────────────────────┤
 │                                                                           │
 │  Pod Speaker k3s-cp1        Pod Speaker k3s-cp2        Pod Speaker k3s-w1 │
 │  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐│
 │  │ Memberlist Node │◄──────►│ Memberlist Node │◄──────►│ Memberlist Node ││
 │  │   (Standby)     │  Gossip│   (Standby)     │  Gossip│    (LEADER!)    ││
 │  └─────────────────┘  7946  └─────────────────┘  7946  └────────┬────────┘│
 │                                                                 │         │
 └─────────────────────────────────────────────────────────────────┼─────────┘
                                                                   │ Menjawab
                                                                   │ ARP untuk
                                                                   │ .200
                                                                   ▼
                                                       Switch LAN / Bridge Host
                                                                   ▲
                                                                   │
                                                            Klien (Mac Host)
```

### Langkah-langkah Operasional:
1. **Pemberian IP oleh Controller:** MetalLB Controller mendeteksi Service baru, mengambil IP `192.168.97.200` dari pool, lalu menulisnya ke status Service.
2. **Koordinasi Antar-Speaker (Gossip Protocol):** Seluruh Pod Speaker yang berjalan di setiap node saling menyapa menggunakan protokol *Memberlist* (berjalan di port TCP/UDP 7946).
3. **Leader Election:** Berdasarkan algoritma konsensus internal (*hash ring* berbasis nama Service dan daftar node yang sehat), para speaker menyepakati **tepat satu node** sebagai pemimpin (*leader*) untuk Service tersebut (misalnya node `k3s-w1`).
4. **Menjawab Permintaan ARP:** Hanya speaker di node `k3s-w1` yang akan mendengarkan interface jaringan host (`eth0`) dan merespons setiap ada kueri ARP untuk IP `192.168.97.200`. Node `k3s-cp1` dan `k3s-cp2` akan tetap diam (*silent*).
5. **Meneruskan Paket ke Pod:** Paket data yang masuk ke node `k3s-w1` akan ditangkap oleh aturan `iptables` / *kube-proxy* Linux kernel, lalu diteruskan ke Pod aplikasi yang sebenarnya (walaupun Pod aplikasi tersebut mungkin sedang hidup di node lain!).

---

## 4. Dua Sumber Daya Kustom (CRD) Utama: IPAddressPool & L2Advertisement

Mulai MetalLB versi modern (`v0.13.x` ke atas), konfigurasi tidak lagi menggunakan ConfigMap lama (`config`), melainkan menggunakan dua Custom Resource Definitions (CRDs) deklaratif:

### 1. `IPAddressPool` (Sumber Alokasi IP)
Resource ini mendefinisikan blok IP apa saja yang boleh dipinjam dan dibagikan oleh MetalLB ke Service:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  # Tentukan rentang IP yang aman (tidak bentrok dengan DHCP)
  addresses:
  - 192.168.97.200-192.168.97.220
  # Atau bisa juga dalam notasi CIDR:
  # - 192.168.97.200/29
  autoAssign: true
```

- `addresses`: Rentang IP (`start-end`) atau notasi CIDR (`/29`). Harus berada di subnet yang sama dengan jaringan host/VM Anda.
- `autoAssign`: Jika bernilai `true`, setiap Service `LoadBalancer` baru akan otomatis mengambil IP dari pool ini. Jika bernilai `false`, Service harus secara eksplisit meminta pool ini menggunakan anotasi:
  ```yaml
  metadata:
    annotations:
      metallb.io/address-pool: lab-pool
  ```

### 2. `L2Advertisement` (Pengumuman Iklan ke Jaringan)
Resource ini bertugas mengikat pool IP ke protokol Layer 2 ARP:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-l2-advert
  namespace: metallb-system
spec:
  # Sebutkan secara eksplisit pool mana yang diiklankan lewat L2
  ipAddressPools:
  - lab-pool
  # Opsional: Membatasi node mana saja yang boleh menjadi Speaker Leader
  # nodeSelectors:
  # - matchLabels:
  #     node-role.kubernetes.io/worker: "true"
```

- Jika `ipAddressPools` dikosongkan, MetalLB akan mengiklankan **seluruh pool** yang ada melalui mode L2. Namun sebagai praktik terbaik SRE, selalu sebutkan nama pool secara eksplisit untuk mencegah kesalahan rute.

---

## 5. Mekanisme Failover & Gratuitous ARP

Apa yang terjadi jika node yang sedang memegang status *Leader* mendadak mengalami mati lampu, kabel putus, atau *kernel crash*?

```text
[ KONDISI 1: NORMAL ]
  Klien ──► [ Node k3s-w1 (LEADER) ] ──► Pod Web
             (MAC: 02:50:f2:8a:11:03)

[ KONDISI 2: INSIDEN — NODE K3S-W1 MATI MENDADAK! ]
  1. Deteksi Kegagalan (Heartbeat Memberlist gagal dalam 1-3 detik).
  2. Pod Speaker di node k3s-cp2 mendeteksi hilangnya k3s-w1.
  3. Dilakukan Leader Re-election: k3s-cp2 terpilih sebagai LEADER BARU!

[ KONDISI 3: GRATUITOUS ARP (PENGUMUMAN MASSAL) ]
  Node k3s-cp2 menembakkan paket "Gratuitous ARP" ke seluruh jaringan:
  "PERHATIAN! Mulai detik ini, IP 192.168.97.200 beralih ke MAC saya (02:50:f2:8a:11:02)!"

[ KONDISI 4: PEMULIHAN ]
  Switch dan Klien (Mac Anda) langsung memperbarui tabel ARP lokal mereka.
  Traffic dialihkan ke k3s-cp2 secara otomatis!
```

> [!NOTE]
> **Karakteristik Failover L2:**
> Proses failover membutuhkan waktu antara **1 hingga 5 detik**. Selama jeda waktu ini:
> - Koneksi TCP yang sedang aktif berjalan mungkin akan mengalami *connection reset* atau *timeout* sesaat (klien perlu melakukan *retry*).
> - Koneksi baru setelah tabel ARP ter-update akan langsung tersambung normal ke leader baru.
> Ini adalah kompromi yang sangat wajar untuk lingkungan lab dan aplikasi internal non-finansial bertraffic sedang.

---

## 6. Deep Dive: `externalTrafficPolicy: Cluster` vs `Local`

Salah satu keputusan arsitektur paling penting di Kubernetes saat menggunakan Service bertipe LoadBalancer adalah memilih nilai `externalTrafficPolicy`:

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster  # Nilai bawaan (default)
  # atau
  # externalTrafficPolicy: Local
```

Mari kita bandingkan keduanya secara mendalam:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│             PERBANDINGAN ALIRAN TRAFFIC: CLUSTER VS LOCAL                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ MODE 1: externalTrafficPolicy: Cluster (Default) ]                       │
│                                                                             │
│   Klien (192.168.97.1)                                                      │
│          │                                                                  │
│          ▼ Menuju Leader Node                                               │
│   Node k3s-w1 (Leader, TIDAK punya Pod Web)                                 │
│          │                                                                  │
│          ├─ SNAT (Source NAT): IP Klien diganti menjadi IP k3s-w1           │
│          │  (Pod Web tidak tahu IP asli klien!)                             │
│          ▼ Extra Network Hop melalui flannel/vxlan                          │
│   Node k3s-cp1 (Memiliki Pod Web)                                           │
│          ▼                                                                  │
│   [ Pod Web ] ──► Log Aplikasi: "Request from 192.168.97.10 (Bukan Mac!)"   │
│                                                                             │
│ ─────────────────────────────────────────────────────────────────────────── │
│                                                                             │
│  [ MODE 2: externalTrafficPolicy: Local ]                                   │
│                                                                             │
│   Klien (192.168.97.1)                                                      │
│          │                                                                  │
│          ▼ Menuju Leader Node                                               │
│   Node k3s-cp1 (Leader DIPILIH KHUSUS HANYA dari node yang punya Pod!)      │
│          │                                                                  │
│          ├─ TANPA SNAT (IP Klien dipertahankan asli!)                       │
│          ├─ TANPA Extra Network Hop (Langsung masuk ke Pod lokal)           │
│          ▼                                                                  │
│   [ Pod Web ] ──► Log Aplikasi: "Request from 192.168.97.1 (IP Mac Asli!)"  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tabel Komparasi Teknis

| Parameter | `externalTrafficPolicy: Cluster` | `externalTrafficPolicy: Local` |
|---|---|---|
| **Preservasi IP Klien** | ❌ Hilang (Klien di-SNAT menjadi IP Node) | ✅ Terjaga (Pod membaca IP asli klien) |
| **Jumlah Lompatan Jaringan** | Potensi 2 hop (Node Leader → Node Pod) | Tepat 1 hop (Node Leader = Node Pod) |
| **Syarat Pemilihan Leader** | Bebas node mana saja di kluster | **Hanya** node yang memiliki Pod sehat |
| **Keseimbangan Beban Traffic** | Lebih merata ke seluruh Pod kluster | Bisa *imbalance* jika sebaran Pod tidak rata |
| **Rekomendasi Pemakaian** | Sangat cocok untuk lab & general purpose | Wajib jika butuh audit log IP / Ingress Controller |

---

## 7. Investigasi Lapangan: Verifikasi Berlapis (Layer-by-Layer)

Sebagai calon SRE profesional, Anda tidak boleh berasumsi "web tidak bisa dibuka, berarti servernya rusak". Lakukan pengujian bertahap dari bawah ke atas mengikuti model OSI:

```text
  Layer 7 (Aplikasi HTTP)  ▲  curl -v http://192.168.97.200/
                           │
  Layer 4 (Transport TCP)  │  nc -zv 192.168.97.200 80
                           │
  Layer 3 (Network IP)     │  ping -c 3 192.168.97.200
                           │
  Layer 2 (Data Link ARP)  │  sudo arping -c 3 192.168.97.200
                           │
  Layer 1 (K8s Allocation) ┴  kubectl get svc -A
```

Mari jalankan perintah pengujian ini langsung dari terminal macOS Anda:

```bash
# Simpan IP VIP ke variabel lingkungan
$ VIP=$(kubectl get svc web-svc -n demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ echo "VIP MetalLB kita adalah: $VIP"
VIP MetalLB kita adalah: 192.168.97.200

# LINGKUP 1: Validasi Status Alokasi di Kubernetes (Layer 1)
$ kubectl get svc web-svc -n demo
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
web-svc   LoadBalancer   10.43.55.109   192.168.97.200   80:31254/TCP   4m

# LINGKUP 2: Inspeksi Pod Speaker & Leader Election (Layer 2)
$ kubectl get pods -n metallb-system -l component=speaker -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP              NODE
speaker-4b2kd   1/1     Running   0          2d    192.168.97.10   k3s-cp1
speaker-9z8wq   1/1     Running   0          2d    192.168.97.11   k3s-cp2
speaker-mpx29   1/1     Running   0          2d    192.168.97.12   k3s-w1

# Periksa log speaker untuk melihat siapa yang memenangkan hak siar VIP
$ kubectl logs -n metallb-system -l component=speaker --tail=100 | grep -i "announcing"
{"caller":"announcer.go:115","event":"serviceAnnounced","ip":"192.168.97.200","msg":"service has IP, announcing","service":"demo/web-svc"}

# LINGKUP 3: Validasi Resolusi ARP Fisik dari macOS Host (Layer 2 / 3)
# Dapatkan nama interface jaringan default Mac (biasanya en0 atau bridge100)
$ IFACE=$(route -n get default | awk '/interface:/{print $2}')
$ echo "Interface jaringan aktif Mac: $IFACE"
Interface jaringan aktif Mac: bridge100

# Jalankan arping untuk membuktikan paket Layer 2 dijawab oleh Node Leader
$ sudo arping -I $IFACE -c 3 $VIP
ARPING 192.168.97.200
60 bytes from 02:50:f2:8a:11:03 (192.168.97.200): index=0 time=412.300 usec
60 bytes from 02:50:f2:8a:11:03 (192.168.97.200): index=1 time=348.110 usec
60 bytes from 02:50:f2:8a:11:03 (192.168.97.200): index=2 time=389.200 usec

--- 192.168.97.200 statistics ---
3 packets transmitted, 3 packets received,   0% unanswered

# Periksa tabel ARP cache di kernel macOS
$ arp -a | grep "$VIP"
? (192.168.97.200) at 2:50:f2:8a:11:3 on bridge100 ifscope [ethernet]

# LINGKUP 4: Validasi Port TCP Terbuka (Layer 4)
$ nc -zv $VIP 80
Connection to 192.168.97.200 port 80 [tcp/http] succeeded!

# LINGKUP 5: Validasi Respon Aplikasi Web (Layer 7)
$ curl -sI http://$VIP/
HTTP/1.1 200 OK
Server: nginx/1.27.4
Content-Type: text/html
Content-Length: 615
```

---

## 8. Kelebihan & Keterbatasan Arsitektur Mode L2

| Dimensi | Kelebihan Mode L2 | Keterbatasan / Konsekuensi Mode L2 |
|---|---|---|
| **Kebutuhan Perangkat** | Tidak butuh router khusus enterprise. Berjalan di switch unmanaged murah atau VM. | Terkunci di dalam **satu subnet / broadcast domain L2** yang sama. |
| **Kemudahan Konfigurasi** | Sangat mudah: cukup 1 file YAML `IPAddressPool` dan 1 `L2Advertisement`. | Tidak ada komputasi rute dinamis lintas gedung data center. |
| **Distribusi Beban (Ingress)** | Otomatis di-forward ke Pod oleh iptables kluster. | **Single Node Bottleneck:** Seluruh bandwidth masuk ditampung oleh 1 interface node leader. |
| **Kecepatan Failover** | Otomatis via Gratuitous ARP (1–5 detik). | Bukan *seamless zero-downtime*, koneksi TCP yang sedang mengalir akan terputus sesaat. |
| **Kesesuaian Lingkungan** | **Sangat ideal untuk lab lokal (OrbStack), kantor cabang, & kluster skala kecil.** | Kurang cocok untuk data center besar dengan traffic puluhan Gigabit/detik. |

---

## 9. Pola Kegagalan (Failure Modes) & Matriks Solusi

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    POHON PEMECAHAN MASALAH LAYER 2 METALLB                  │
└─────────────────────────────────────────────────────────────────────────────┘
  Masalah: Curl ke VIP Timeout dari Mac
    │
    ├─► Cek 1: Apakah 'arping -c 3 <VIP>' merespons?
    │     │
    │     ├── TIDAK (0% answered):
    │     │     ├─► Cek apakah IP pool satu subnet dengan Mac ('orb ip k3s-cp1')?
    │     │     ├─► Cek pod speaker: 'kubectl get pods -n metallb-system'
    │     │     └─► Cek apakah ada CRD 'L2Advertisement' yang mencakup pool tersebut?
    │     │
    │     └── YA (Ada jawaban MAC Address):
    │           │
    │           └─► Lanjut Cek 2: Apakah 'nc -zv <VIP> 80' sukses?
    │                 │
    │                 ├── TIDAK (Connection Refused):
    │                 │     ├─► Cek apakah Endpoints/Pods Service ada:
    │                 │     │   'kubectl get endpoints <nama-svc>'
    │                 │     └─► Cek apakah targetPort di Service cocok dengan
    │                 │         containerPort di Pod (misal 80 vs 8080)?
    │                 │
    │                 └── YA: Cek respon HTTP aplikasi ('curl -v') atau log Pod.
```

---

## 10. Ringkasan Praktis & Cheatsheet Perintah

```bash
# Melihat seluruh pool IP MetalLB yang terdaftar
kubectl get ipaddresspool -n metallb-system

# Melihat seluruh aturan iklan L2
kubectl get l2advertisement -n metallb-system

# Melihat siapa node yang memegang IP tertentu (speaker log)
kubectl logs -n metallb-system -l component=speaker --tail=50 | grep -i "announcing"

# Menguji resolusi ARP langsung ke interface jaringan
sudo arping -I <interface> -c 3 <VIP>

# Membersihkan entri ARP lokal Mac jika terjadi cache kadaluarsa
sudo arp -d <VIP>
```

---

## 11. Cek Pemahaman Mandiri

1. Mengapa paket ARP broadcast (`ff:ff:ff:ff:ff:ff`) tidak dapat melintasi router fisik yang menghubungkan dua gedung kantor berbeda? Apa implikasinya terhadap pemilihan IP untuk MetalLB L2 mode?
2. Pada mode L2, jika Anda memiliki 3 node worker dan 1 Service LoadBalancer, apakah ketiga node tersebut membagi traffic masuk secara seimbang di kartu jaringan fisik mereka? Jelaskan fenomena *single-node bottleneck*!
3. Apa perbedaan esensial dari cara kerja `externalTrafficPolicy: Cluster` vs `externalTrafficPolicy: Local` dalam hal pencatatan IP klien di log aplikasi web?
4. Mengapa kita harus menggunakan perintah `arping` terlebih dahulu sebelum menyalahkan konfigurasi NGINX ketika `curl` ke VIP menghasilkan pesan *Connection Timed Out*?
5. Jelaskan peran paket *Gratuitous ARP* saat node leader mengalami insiden mati mendadak (*crash*)!

---

*Langkah Selanjutnya:* Masuk ke [03 — BGP Mode: Konsep untuk Production](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.3-metallb/03-bgp-mode-konsep.md) untuk mempelajari bagaimana arsitektur data center enterprise berskala besar mengatasi keterbatasan Layer 2 menggunakan protokol routing dinamis BGP dan ECMP.