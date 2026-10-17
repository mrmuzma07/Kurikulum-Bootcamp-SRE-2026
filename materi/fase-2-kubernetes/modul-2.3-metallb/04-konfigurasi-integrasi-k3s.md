# 04 — Konfigurasi & Integrasi MetalLB dengan k3s: Panduan Praktis dari Instalasi Manifest hingga Akses VIP Host Mac

> **Ringkasan:** Dokumen panduan langkah demi langkah untuk menginstal, mengonfigurasi, dan memvalidasi MetalLB di atas kluster k3s Multi-Node High Availability (VM OrbStack). Mulai dari audit penonaktifan *servicelb*, penentuan rentang IP pool yang aman dari konflik jaringan, penerapan sumber daya kustom *IPAddressPool* dan *L2Advertisement*, pengujian Service LoadBalancer nyata, hingga metodologi investigasi 5-Layer dari terminal Mac Anda.

---

## 1. Prasyarat & Prinsip Keamanan Konteks (Context Safety)

Sebelum mengetikkan perintah apa pun di terminal Mac Anda, seorang SRE wajib melakukan verifikasi konteks (*context safety check*). Hal ini penting untuk memastikan bahwa Anda tidak salah mengeksekusi perubahan pada kluster k3d atau kluster production lain yang tidak sengaja tersambung.

```bash
# 1. Periksa konteks kubectl yang sedang aktif di terminal Mac
$ kubectl config current-context
k3s-ha-lab

# 2. Periksa informasi kluster
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.97.10:6443

# 3. Periksa status seluruh node kluster (Pastikan semua node Ready)
$ kubectl get nodes -o wide
NAME      STATUS   ROLES                       AGE   VERSION        INTERNAL-IP     OS-IMAGE             KERNEL-VERSION
k3s-cp1   Ready    control-plane,etcd,master   2d    v1.31.5+k3s1   192.168.97.10   Ubuntu 22.04.5 LTS   6.6.65-orbstack
k3s-cp2   Ready    control-plane,etcd,master   2d    v1.31.5+k3s1   192.168.97.11   Ubuntu 22.04.5 LTS   6.6.65-orbstack
k3s-w1    Ready    worker                      2d    v1.31.5+k3s1   192.168.97.12   Ubuntu 22.04.5 LTS   6.6.65-orbstack
```

> [!CAUTION]
> **Peringatan Keselamatan Konteks:**
> Jika output perintah di atas menunjukkan `k3d-...` atau nama kluster lain, segera alihkan konteks Anda ke kluster VM k3s menggunakan perintah: `kubectl config use-context k3s-ha-lab`. Jangan pernah menginstal MetalLB L2 di dalam kluster k3d karena paket broadcast ARP akan terisolasi di dalam network bridge Docker!

---

## 2. Audit Pra-Instalasi: Pastikan `servicelb` k3s Telah Dinonaktifkan

Pada [Modul 2.2 — k3s Production](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.2-k3s-production/03-disable-komponen-ingress.md), kita telah memasang k3s dengan flag `--disable servicelb`. Mari kita audit kepatuhan konfigurasi ini:

```bash
# 1. Periksa apakah ada Pod svclb yang sedang berjalan di namespace mana pun
$ kubectl get pods -A | grep -i svclb || echo "AUDIT BERSIH: Tidak ada pod svclb k3s."
AUDIT BERSIH: Tidak ada pod svclb k3s.

# 2. Periksa apakah ada DaemonSet svclb yang tertinggal
$ kubectl get daemonset -A | grep -i svclb || echo "AUDIT BERSIH: DaemonSet svclb nihil."
AUDIT BERSIH: DaemonSet svclb nihil.
```

### Mengapa Langkah Audit Ini Sangat Krusial?
Jika `servicelb` (Klipper) masih aktif:
- Setiap kali Anda membuat Service bertipe `LoadBalancer`, `servicelb` akan otomatis membuat Pod DaemonSet bernama `svclb-<nama-svc>` di setiap node.
- Pod tersebut akan membajak port fisik node host (misal port 80) menggunakan `hostPort`.
- Ketika MetalLB mencoba mengalokasikan Virtual IP (VIP), terjadi tabrakan port dan rute (*routing race condition*) yang membuat koneksi jaringan menjadi acak dan tidak stabil.

---

## 3. Investigasi Subnet Jaringan & Pemilihan IP Pool yang Aman

Sebelum memasang MetalLB, Anda harus menentukan rentang alamat IP (*IP Address Pool*) yang akan diberikan kepada Service. Alamat-alamat ini **BUKAN** alamat IP sembarangan.

### Tiga Kriteria Alamat IP yang Aman untuk Mode L2:
1. **Satu Subnet L2 dengan Node dan Mac Host:** Paket ARP harus bisa didengar langsung oleh Mac Anda tanpa melalui router NAT.
2. **Di Luar Rentang Alokasi DHCP:** Jangan menggunakan IP yang dapat dibagikan secara acak oleh router Wi-Fi atau DHCP server ke laptop/handphone lain.
3. **Belum Digunakan oleh Host Fisik Mana Pun:** Lakukan verifikasi sebelum menetapkan pool.

Mari kita lakukan inventarisasi jaringan dari Mac:

```bash
# 1. Periksa IP masing-masing node k3s OrbStack Anda
$ orb list
NAME       STATE    ARCH    CPU  RAM    DISK
k3s-cp1    running  arm64   2    2 GiB  16 GiB
k3s-cp2    running  arm64   2    2 GiB  16 GiB
k3s-w1     running  arm64   2    2 GiB  16 GiB

$ orb ip k3s-cp1
192.168.97.10

# 2. Periksa interface jaringan Mac yang menuju ke subnet tersebut
$ route -n get $(orb ip k3s-cp1) | grep -E 'interface|gateway'
   gateway: 192.168.97.1
 interface: bridge100

# 3. Uji kandidat rentang IP (Misal kita pilih rentang: 192.168.97.200 s.d. 192.168.97.220)
# Pastikan IP 192.168.97.200 saat ini TIDAK ADA yang memakai (0% response)
$ sudo arping -I bridge100 -c 3 192.168.97.200
ARPING 192.168.97.200
--- 192.168.97.200 statistics ---
3 packets transmitted, 0 packets received, 100% unanswered
```

> [!TIP]
> **Hasil Uji yang Diharapkan:**
> Output `100% unanswered` pada perintah `arping` di atas adalah **tanda bahwa IP 192.168.97.200 masih perawan/kosong**. Ini menandakan IP tersebut sangat aman untuk dijadikan Virtual IP MetalLB! Jika perintah `arping` menghasilkan respon MAC address, berarti IP tersebut sudah digunakan oleh perangkat lain dan Anda harus memilih nomor IP yang lain.

---

## 4. Instalasi MetalLB Native Manifest (Versi Terpin)

Sebagai praktik terbaik SRE, kita selalu menyematkan nomor versi (*version pinning*) untuk menjamin reproduktibilitas dan mencegah *breaking changes* akibat perubahan manifest remote yang tidak terkontrol.

Kita menggunakan MetalLB versi stabil resmi: **`v0.14.9`**.

```bash
# 1. Deklarasikan variabel versi MetalLB
$ export METALLB_VERSION="v0.14.9"

# 2. Terapkan manifes resmi instalasi MetalLB Native
$ kubectl apply -f "https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml"
namespace/metallb-system created
customresourcedefinition.apiextensions.k8s.io/bgppeers.metallb.io created
customresourcedefinition.apiextensions.k8s.io/ipaddresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io created
serviceaccount/controller created
serviceaccount/speaker created
role.rbac.authorization.k8s.io/controller created
role.rbac.authorization.k8s.io/pod-lister created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
deployment.apps/controller created
daemonset.apps/speaker created

# 3. Tunggu hingga Controller dan seluruh Speaker berstatus Ready
$ kubectl wait --namespace metallb-system \
    --for=condition=available deployment/controller \
    --timeout=180s
deployment.apps/controller condition met

$ kubectl rollout status daemonset/speaker -n metallb-system --timeout=180s
daemon set "speaker" successfully rolled out

# 4. Verifikasi status Pod di namespace metallb-system
$ kubectl get pods -n metallb-system -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP              NODE
controller-65895b47d4-8m9p7   1/1     Running   0          45s   10.42.0.15      k3s-cp1
speaker-4b2kd                 1/1     Running   0          45s   192.168.97.10   k3s-cp1
speaker-9z8wq                 1/1     Running   0          45s   192.168.97.11   k3s-cp2
speaker-mpx29                 1/1     Running   0          45s   192.168.97.12   k3s-w1

# 5. Pastikan CRD MetalLB telah terdaftar sempurna di API Server
$ kubectl get crd | grep metallb
bfdprofiles.metallb.io           2026-10-17T06:30:00Z
bgpadvertisements.metallb.io     2026-10-17T06:30:00Z
bgppeers.metallb.io              2026-10-17T06:30:00Z
ipaddresspools.metallb.io        2026-10-17T06:30:00Z
l2advertisements.metallb.io      2026-10-17T06:30:00Z
```

Perhatikan bahwa Pod **Speaker** berjalan dengan IP Node fisik (`192.168.97.x`) karena menggunakan `hostNetwork: true`, sedangkan Pod **Controller** berjalan dengan IP virtual internal Pod (`10.42.x.x`).

---

## 5. Penerapan Konfigurasi: IPAddressPool & L2Advertisement

Setelah komponen dasar MetalLB berjalan, kita harus mendefinisikan aturan alokasi IP dan metode pengumumannya melalui Custom Resource.

Buat file konfigurasi bernama `metallb-l2-config.yaml`:

```bash
$ cat << 'EOF' > metallb-l2-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: orbstack-l2-pool
  namespace: metallb-system
spec:
  # Rentang IP yang telah diverifikasi aman pada Langkah 3
  addresses:
  - 192.168.97.200-192.168.97.220
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: orbstack-l2-advert
  namespace: metallb-system
spec:
  # Kaitkan secara spesifik dengan pool yang telah didefinisikan di atas
  ipAddressPools:
  - orbstack-l2-pool
EOF
```

Terapkan manifes konfigurasi ke kluster:

```bash
$ kubectl apply -f metallb-l2-config.yaml
ipaddresspool.metallb.io/orbstack-l2-pool created
l2advertisement.metallb.io/orbstack-l2-advert created

# Verifikasi keberadaan dan status konfigurasi
$ kubectl get ipaddresspool -n metallb-system
NAME               AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
orbstack-l2-pool   true          false             ["192.168.97.200-192.168.97.220"]

$ kubectl get l2advertisement -n metallb-system
NAME                 IPADDRESSPOOLS         IPADDRESSPOOL SELECTORS   NODES
orbstack-l2-advert   ["orbstack-l2-pool"]
```

---

## 6. Hands-On: Pengujian Nyata dengan Service LoadBalancer

Sekarang saatnya membuktikan apakah MetalLB berhasil mengakhiri fenomena abadi `<pending>`! Kita akan membuat namespace `demo`, men-deploy aplikasi web NGINX sederhana dengan 2 replika, dan mengeksposnya menggunakan `type: LoadBalancer`.

Buat file bernama `demo-web.yaml`:

```bash
$ cat << 'EOF' > demo-web.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
  namespace: demo
  labels:
    app: nginx-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-web-svc
  namespace: demo
spec:
  type: LoadBalancer
  selector:
    app: nginx-web
  ports:
  - name: http
    port: 80
    targetPort: 80
EOF
```

Terapkan manifes aplikasi ke dalam kluster dan amati status Service:

```bash
$ kubectl apply -f demo-web.yaml
namespace/demo created
deployment.apps/nginx-web created
service/nginx-web-svc created

# Amati pembaruan EXTERNAL-IP secara real-time
$ kubectl get svc nginx-web-svc -n demo -w
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-web-svc   LoadBalancer   10.43.190.22   <pending>     80:31455/TCP   0s
nginx-web-svc   LoadBalancer   10.43.190.22   192.168.97.200 80:31455/TCP   2s
```

> [!NOTE]
> **Keajaiban Telah Terjadi!**
> Perhatikan kolom `EXTERNAL-IP`. Hanya dalam waktu 2 detik, status `<pending>` seketika berganti menjadi **`192.168.97.200`**! Ini adalah bukti nyata bahwa MetalLB Controller telah bekerja dengan sempurna mengambil IP pertama yang tersedia dari *orbstack-l2-pool*.

---

## 7. Rantai Investigasi 5-Layer (The 5-Layer Verification Chain)

Sebagai SRE, melihat IP terisi di terminal `kubectl` barulah tahap awal. Kita harus membuktikan seluruh rantai konektivitas jaringan dari Layer 1 hingga Layer 7 berjalan tanpa cacat:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                      RANTAI VERIFIKASI 5-LAYER SRE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. K8s Allocation : Apakah Kube API menetapkan IP & Endpoints Pod sehat?    │
│ 2. Speaker Leader : Node mana yang memenangkan hak siar ARP untuk IP .200?  │
│ 3. Layer 2 (ARP)  : Apakah kernel macOS menerima balasan MAC dari .200?     │
│ 4. Layer 4 (TCP)  : Apakah port TCP 80 terbuka dan menerima SYN-ACK?        │
│ 5. Layer 7 (HTTP) : Apakah web server NGINX mengembalikan HTTP 200 OK?      │
└─────────────────────────────────────────────────────────────────────────────┘
```

Mari kita jalankan verifikasi langkah demi langkah:

```bash
# Simpan IP VIP ke variabel sesi shell
$ VIP=$(kubectl get svc nginx-web-svc -n demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ echo "VIP pengujian: $VIP"
VIP pengujian: 192.168.97.200

# LAYER 1: Validasi Endpoints Kubernetes
$ kubectl describe svc nginx-web-svc -n demo
Name:                     nginx-web-svc
Namespace:                demo
Type:                     LoadBalancer
IP:                       10.43.190.22
LoadBalancer Ingress:     192.168.97.200
Port:                     http  80/TCP
TargetPort:               80/TCP
Endpoints:                10.42.1.8:80,10.42.2.14:80
Events:
  Type    Reason            Age   From                Message
  ----    ------            ----  ----                -------
  Normal  IPAllocated       2m    metallb-controller  Assigned IP ["192.168.97.200"]

# LAYER 2: Validasi Speaker Leader Election
$ kubectl logs -n metallb-system -l component=speaker --tail=100 | grep "192.168.97.200"
{"caller":"announcer.go:115","event":"serviceAnnounced","ip":"192.168.97.200","msg":"service has IP, announcing","service":"demo/nginx-web-svc"}

# LAYER 3: Validasi ARP Resolution dari Terminal Mac Host
$ IFACE=$(route -n get default | awk '/interface:/{print $2}')
$ sudo arping -I $IFACE -c 3 $VIP
ARPING 192.168.97.200
60 bytes from 02:50:f2:8a:11:03 (192.168.97.200): index=0 time=392.110 usec
60 bytes from 02:50:f2:8a:11:03 (192.168.97.200): index=1 time=340.220 usec
60 bytes from 02:50:f2:8a:11:03 (192.168.97.200): index=2 time=365.180 usec

--- 192.168.97.200 statistics ---
3 packets transmitted, 3 packets received,   0% unanswered

# Periksa tabel tetangga ARP Mac
$ arp -a | grep "$VIP"
? (192.168.97.200) at 2:50:f2:8a:11:3 on bridge100 ifscope [ethernet]

# LAYER 4: Validasi Port TCP 80 Terbuka
$ nc -zv $VIP 80
Connection to 192.168.97.200 port 80 [tcp/http] succeeded!

# LAYER 5: Validasi Respon HTTP Aplikasi Web dari Mac
$ curl -sI http://${VIP}/
HTTP/1.1 200 OK
Server: nginx/1.27.4
Date: Sat, 17 Oct 2026 06:45:12 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
```

Anda juga dapat membuka peramban web Safari atau Chrome di Mac Anda dan mengetikkan `http://192.168.97.200`. Halaman selamat datang *"Welcome to nginx!"* akan tampil dengan sempurna!

---

## 8. Pola Arsitektur Produksi: Menghubungkan MetalLB dengan Ingress Controller

Di lingkungan production berskala besar, seorang SRE **tidak akan membuat 100 Service bertipe `LoadBalancer`** untuk 100 microservice (karena ini akan menghabiskan 100 kuota alamat IP publik/VIP jaringan).

Pola industri yang baku (*standard reference architecture*) adalah **mengombinasikan MetalLB dengan Ingress Controller**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ARSITEKTUR PRODUCTION: METALLB + INGRESS                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Klien Eksternal / Browser                                                  │
│        │                                                                    │
│        ├─► https://app.perusahaan.id  ──┐                                   │
│        ├─► https://api.perusahaan.id  ──┼──► DNS: *.perusahaan.id           │
│        └─► https://auth.perusahaan.id ──┘          │                        │
│                                                    ▼                        │
│                                           192.168.97.200 (VIP MetalLB Tunggal)
│                                                    │                        │
│                                                    ▼                        │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Service: ingress-nginx / traefik (Type: LoadBalancer)                   │ │
│ │ Memiliki SATU IP VIP MetalLB: 192.168.97.200                            │ │
│ └────────────────────────────────────┬────────────────────────────────────┘ │
│                                      │                                      │
│                                      ▼ Meneruskan Traffic Layer 7 (HTTP)    │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Ingress Controller Pods (Inspeksi HTTP Host & Path Header)              │ │
│ └──────┬─────────────────────────────┼─────────────────────────────┬──────┘ │
│        │ Host: app.perusahaan.id     │ Host: api.perusahaan.id     │        │
│        ▼                             ▼                             ▼        │
│ ┌───────────────┐           ┌───────────────┐           ┌───────────────┐   │
│ │ Svc ClusterIP │           │ Svc ClusterIP │           │ Svc ClusterIP │   │
│ │ (Frontend)    │           │ (Backend API) │           │ (Auth Svc)    │   │
│ └──────┬────────┘           └──────┬────────┘           └──────┬────────┘   │
│        ▼                           ▼                           ▼            │
│    Pod Frontend                Pod Backend                  Pod Auth        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Dengan pola arsitektur ini:
- Anda **hanya membutuhkan 1 alamat IP Virtual MetalLB** untuk seluruh kluster!
- Ingress Controller menangani SSL/TLS termination, kompresi gzip, dan pembagian domain.
- MetalLB menangani kestabilan pintu gerbang Layer 4 fisik data center.

---

## 9. Playbook Troubleshooting: Matriks Investigasi Insiden

Jika dalam pelaksanaan lab Anda menemui kendala, gunakan matriks rujukan berikut untuk mendeteksi akar masalah:

| Gejala Masalah | Titik Investigasi | Dugaan Akar Masalah (*Root Cause*) | Solusi Pemulihan |
|---|---|---|---|
| **`EXTERNAL-IP` tertahan `<pending>`** | `kubectl get pods -n metallb-system`<br>`kubectl describe svc <nama-svc>` | Controller tidak jalan, atau pool IP kehabisan stok, atau CRD `IPAddressPool` belum ter-apply. | Pastikan pod controller `Running` dan pastikan rentang `addresses` di pool cukup luas. |
| **`EXTERNAL-IP` terisi, tapi `arping` 100% timeout** | `kubectl get l2advertisement -A`<br>`kubectl logs -n metallb-system -l component=speaker` | File `L2Advertisement` lupa di-apply, atau subnet pool salah dan tidak cocok dengan kartu jaringan VM/Mac. | Pastikan `L2Advertisement` sudah dibuat dan IP pool berada pada subnet yang sama dengan `orb ip k3s-cp1`. |
| **`arping` sukses, tapi `curl` menghasilkan `Connection Refused`** | `kubectl get endpoints <nama-svc>`<br>`kubectl describe pod -l app=<label>` | Pod aplikasi crash/belum siap (`CrashLoopBackOff`), atau targetPort di Service salah ketik. | Periksa apakah Pod aplikasi berstatus `Running 1/1` dan perbaiki targetPort agar cocok dengan containerPort. |
| **Terdeteksi IP Conflict (Dua MAC menjawab satu IP)** | `sudo arping -I bridge100 -c 5 <VIP>` | IP pool yang dipilih ternyata sudah dipakai oleh perangkat fisik lain di LAN kantor/rumah. | Ganti rentang alamat di `IPAddressPool` ke rentang IP yang benar-benar kosong. |

---

## 10. Prosedur Teardown & Pembersihan (Clean-up) Lab

Setelah menyelesaikan latihan dan pengujian, lakukan pembersihan dengan urutan yang benar agar tidak meninggalkan konfigurasi *orphaned* di kluster:

```bash
# 1. Hapus aplikasi demo dan servicenya terlebih dahulu
$ kubectl delete -f demo-web.yaml
namespace "demo" deleted
deployment.apps "nginx-web" deleted
service "nginx-web-svc" deleted

# 2. Hapus konfigurasi pool MetalLB
$ kubectl delete -f metallb-l2-config.yaml
ipaddresspool.metallb.io "orbstack-l2-pool" deleted
l2advertisement.metallb.io "orbstack-l2-advert" deleted

# 3. (Opsional) Jika ingin menghapus seluruh MetalLB dari kluster:
# $ kubectl delete -f "https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml"

# 4. Validasi kebersihan kluster
$ kubectl get svc -A | grep -i loadbalancer || echo "Kluster bersih dari LoadBalancer."
Kluster bersih dari LoadBalancer.
```

---

## 11. Acceptance Criteria (Kriteria Keberhasilan Praktik)

Sebelum melangkah ke modul berikutnya, pastikan seluruh kriteria operasional berikut telah terpenuhi di lingkungan Anda:

- [ ] Konteks `kubectl` terverifikasi menunjuk ke kluster k3s VM (`k3s-ha-lab`).
- [ ] Fitur bawaan k3s `servicelb` dipastikan bersih dan nonaktif (`svclb` nihil).
- [ ] MetalLB Controller (Deployment) dan Speaker (DaemonSet) berstatus `1/1 Running`.
- [ ] CRD `IPAddressPool` dan `L2Advertisement` terdaftar dan aktif di namespace `metallb-system`.
- [ ] Service `nginx-web-svc` berhasil memperoleh Virtual IP nyata tanpa tertahan `<pending>`.
- [ ] Perintah `arping -I <interface> -c 3 <VIP>` dari terminal Mac menghasilkan `0% unanswered` (mendapat respon MAC address).
- [ ] Perintah `curl -sI http://<VIP>/` dari terminal Mac mengembalikan status `HTTP/1.1 200 OK`.
- [ ] Seluruh tahapan verifikasi terdokumentasi dan dipahami alurnya secara Layer-by-Layer.

---

## 12. Cek Pemahaman Mandiri

1. Mengapa kita harus menginspeksi ketiadaan DaemonSet `svclb` sebelum menginstal MetalLB di kluster k3s?
2. Apa yang akan terjadi jika kita mendefinisikan `IPAddressPool` pada subnet `10.200.0.0/24`, sementara mesin VM k3s dan Mac host berada pada subnet `192.168.97.0/24`? Apakah perintah `arping` akan berhasil? Jelaskan!
3. Sebutkan perbedaan fungsi antara Custom Resource `IPAddressPool` dan `L2Advertisement`! Mengapa MetalLB v0.14 memisahkan kedua objek ini?
4. Pada rantai investigasi 5-Layer, jika perintah `nc -zv <VIP> 80` berhasil (*succeeded*), tetapi perintah `curl http://<VIP>/` menghasilkan pesan error `HTTP 502 Bad Gateway`, di layer manakah masalah tersebut berada dan komponen apa yang harus Anda periksa?
5. Mengapa arsitektur production enterprise lebih memilih memadukan MetalLB dengan Ingress Controller dibandingkan membuat Service bertipe `LoadBalancer` untuk setiap aplikasi microservice?

---

*Langkah Selanjutnya:* Praktikkan seluruh rangkaian langkah di atas secara mandiri pada [LAB-01 — Instalasi MetalLB & Expose Service LoadBalancer](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.3-metallb/lab/LAB-01-install-metallb-expose-svc.md), kemudian lanjutkan ke [Modul 2.4 — Troubleshooting Kluster Kubernetes](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-2-kubernetes/modul-2.4-troubleshooting/README.md) untuk mendalami seni mitigasi bencana dan crashloop di lingkungan production.
