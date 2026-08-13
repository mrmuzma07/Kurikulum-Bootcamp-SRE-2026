# 03 — BGP Mode: Konsep untuk Production

> L2 menjawab ARP untuk VIP. BGP mengumumkan route VIP ke router — pendekatan yang lebih scalable untuk jaringan on-prem multi-subnet.

## Tujuan

- Memahami BGP sebagai control-plane routing, bukan protokol aplikasi
- Menjelaskan peering antara MetalLB Speaker dan router
- Memahami ASN, prefix VIP, next-hop, dan ECMP
- Membandingkan BGP dengan L2 secara operasional
- Bisa membaca konfigurasi `BGPPeer` dan `BGPAdvertisement`
- Mengetahui mengapa lab laptop tetap memakai L2, sedangkan production dapat memilih BGP

## 1. Mengapa BGP?

Pada mode L2, client menemukan VIP dengan ARP/NDP. Cara ini sederhana, tetapi dibatasi oleh satu broadcast domain. Untuk jaringan production, node Kubernetes dan client sering berada di VLAN/subnet berbeda. Router perlu tahu bahwa VIP tertentu dapat dicapai melalui node Kubernetes.

BGP (Border Gateway Protocol) menyampaikan informasi routing antar perangkat. MetalLB Speaker bertindak sebagai BGP router kecil:

```text
             BGP peering (TCP/179)
  ┌──────────────────────────────────────────┐
  │ Router / ToR switch                      │
  │ ASN 64501                                │
  │ route: 192.168.100.200/32 via node      │
  └──────────────────┬───────────────────────┘
                     │
       ┌─────────────┼─────────────┐
       │              │             │
 ┌─────▼─────┐  ┌────▼──────┐  ┌───▼──────┐
 │ Speaker cp1│  │ Speaker w1│  │ Speaker w2│
 │ ASN 64512  │  │ ASN 64512  │  │ ASN 64512 │
 │ announce   │  │ announce   │  │ announce  │
 │ VIP /32    │  │ VIP /32    │  │ VIP /32   │
 └────────────┘  └────────────┘  └───────────┘
```

Alurnya:

1. Controller mengalokasikan VIP dari `IPAddressPool`.
2. Speaker membangun session BGP ke router yang dikonfigurasi.
3. Speaker mengumumkan prefix VIP, umumnya `/32` untuk IPv4.
4. Router memasukkan route ke routing table.
5. Router meneruskan traffic ke salah satu node yang mengumumkan VIP.
6. Kube-proxy/Service meneruskan traffic ke Pod.

Tidak ada ARP broadcast dari MetalLB yang harus melintasi router. Router menerima route secara eksplisit melalui BGP.

## 2. Istilah Dasar

| Istilah | Arti dalam MetalLB |
|---|---|
| **ASN** | Autonomous System Number; identitas router dalam BGP. Lab memakai private ASN seperti `64512`. |
| **BGP peer** | Alamat router yang diajak membuat session BGP. |
| **eBGP** | Peering antar-ASN berbeda, pola umum MetalLB ↔ ToR/router. |
| **iBGP** | Peering dalam ASN yang sama; desainnya lebih khusus dan perlu perhatian route reflector. |
| **Prefix** | Network yang diumumkan, misalnya VIP `192.168.100.200/32`. |
| **Next-hop** | Node/router yang menjadi tujuan paket untuk prefix tersebut. |
| **ECMP** | Equal-Cost Multi-Path; router menggunakan beberapa path setara untuk membagi traffic. |
| **TCP/179** | Port yang dipakai session BGP. Ini berbeda dari port aplikasi Service. |

Gunakan **private ASN** untuk lab/internal network. ASN publik tidak diperlukan untuk peering internal dan tidak boleh dipilih sembarangan pada jaringan organisasi.

## 3. BGPPeer & BGPAdvertisement

Contoh konseptual (API version harus sesuai CRD MetalLB yang terpasang):

```yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: tor-router
  namespace: metallb-system
spec:
  myASN: 64512
  peerASN: 64501
  peerAddress: 192.168.97.1
  # sourceAddress: 192.168.97.10
  # holdTime: 90s
  # ebgpMultiHop: false
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: production-vips
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
  # aggregationLength: 32
```

Makna penting:

- `BGPPeer` mendeskripsikan **siapa** router peer, ASN mereka, dan alamat session.
- `BGPAdvertisement` mendeskripsikan **prefix mana** yang diumumkan.
- `IPAddressPool` tetap menentukan **VIP mana** yang boleh dialokasikan.
- Mengubah pool tidak otomatis berarti router siap menerima route; session dan filter router juga harus benar.

> **Catatan versi:** API CRD dapat berbeda antar-release MetalLB. Selalu cek `kubectl explain bgppeer.spec` dan dokumentasi release yang dipakai. Jangan menyalin `apiVersion` contoh tanpa memeriksa CRD.

```bash
kubectl api-resources | grep -i metallb
kubectl explain bgppeer.spec
kubectl explain bgpadvertisement.spec
kubectl get crd | grep metallb
```

## 4. ECMP dan Distribusi Traffic

Dengan L2, satu VIP dipimpin satu Speaker. Dengan BGP, beberapa Speaker dapat mengumumkan prefix VIP yang sama ke router. Router dapat memilih beberapa next-hop setara melalui ECMP:

```text
Client → ToR router
          ├─ route VIP via cp1
          ├─ route VIP via w1
          └─ route VIP via w2
```

Router biasanya membagi **flow** berdasarkan hash (source/destination IP dan port), bukan memecah satu koneksi TCP ke banyak node. Akibatnya:

- beberapa koneksi dapat tersebar ke node berbeda;
- satu koneksi tetap konsisten ke satu path;
- distribusi bergantung pada kemampuan dan kebijakan router;
- hilangnya satu node menyebabkan router menarik route dari node tersebut setelah session/health berubah.

ECMP tidak menghilangkan kebutuhan readiness probe, Service, kube-proxy, dan observability. Ia hanya mengubah cara traffic menemukan node.

## 5. BGP vs L2

| Aspek | L2 ARP/NDP | BGP |
|---|---|---|
| Mekanisme | Speaker jawab ARP/NDP | Speaker announce route ke router |
| Batas network | Satu L2 subnet | Dapat lintas subnet/VLAN sesuai routing |
| Perangkat tambahan | Tidak perlu konfigurasi router khusus | Perlu router yang mendukung dan mengizinkan BGP |
| Ingress traffic | Satu node leader per VIP | Beberapa node dapat menerima via ECMP |
| Kompleksitas | Rendah | Lebih tinggi: ASN, peer, policy, filter |
| Failover | Leader election + refresh ARP | Withdraw route/session convergence |
| Cocok untuk | Lab, jaringan flat, cluster kecil | Production, banyak node/VIP, multi-subnet |
| Risiko utama | IP conflict, ARP tidak sampai | Route leak, salah policy, session flap |

Pemilihan mode bukan soal "BGP selalu lebih baik". L2 tepat bila network sederhana dan requirement traffic kecil. BGP tepat bila tim network siap mengoperasikan peering dan memerlukan routing yang deterministic.

## 6. Keamanan dan Operasi BGP

BGP adalah control-plane jaringan. Perlakukan konfigurasi peer sebagai perubahan production:

- Batasi TCP/179 hanya antara Speaker dan router yang sah.
- Gunakan prefix filter/route policy di router; jangan menerima atau mengiklankan semua prefix tanpa batas.
- Validasi ASN, source address, dan `peerAddress` sebelum apply.
- Gunakan password/MD5 atau mekanisme autentikasi yang didukung desain router bila kebijakan organisasi mengharuskannya.
- Monitor session state, jumlah prefix, route flap, dan perubahan next-hop.
- Uji withdraw/failover di maintenance window; jangan mematikan node production sembarangan.
- Simpan konfigurasi MetalLB dan konfigurasi router dalam review/GitOps, tanpa membocorkan secret.

Salah konfigurasi BGP dapat berdampak di luar cluster. Karena itu BGP lab dibahas sebagai konsep; implementasi nyata membutuhkan koordinasi dengan pemilik jaringan.

## 7. Mengapa Lab Mac Memakai L2?

OrbStack Machine memberi lingkungan yang cukup realistis untuk menguji ARP pada lab ini. Namun laptop biasanya tidak memiliki ToR router yang dapat dijadikan peer BGP. Membuat router virtual memang mungkin, tetapi menambah variabel dan tidak merepresentasikan policy jaringan organisasi.

Lane bootcamp:

```text
k3d container        → objek & operasi cepat (Modul 2.1)
k3s VM + MetalLB L2  → LoadBalancer bare-metal & ARP (Modul 2.3 lab)
BGP                   → desain production, diuji di environment network yang berizin
```

Jangan membuat peering ke gateway rumah/kantor tanpa otorisasi. Gateway yang tidak mengharapkan session BGP dapat menolak, log error, atau—lebih buruk—menerima route yang salah.

## 8. Observability BGP

Nama resource dan command dapat berbeda menurut mode speaker/release. Mulai dari status CRD dan log:

```bash
kubectl get bgppeers,bgpadvertisements -n metallb-system
kubectl describe bgppeer tor-router -n metallb-system
kubectl get pods -n metallb-system -l component=speaker -o wide
kubectl logs -n metallb-system -l component=speaker --tail=100 | grep -iE 'bgp|peer|establish|withdraw|error'
```

Di router, periksa:

```text
BGP neighbor state: Established
received/advertised prefixes
route 192.168.100.200/32
next-hop dan ECMP path
last reset / flap reason
```

Jika `EXTERNAL-IP` sudah terisi tetapi route tidak ada di router, fokus pada `BGPPeer`, TCP/179, ASN, firewall, dan route policy—bukan pada Deployment app terlebih dahulu.

## Ringkasan

1. L2 mengumumkan VIP melalui ARP/NDP; BGP mengumumkan route melalui router.
2. `IPAddressPool` mengalokasikan IP; `BGPAdvertisement` memilih pool yang diumumkan; `BGPPeer` menentukan router peer.
3. BGP dapat memberi multi-path/ECMP dan lintas subnet, tetapi membutuhkan desain jaringan dan operasi lebih matang.
4. Private ASN dan prefix filter penting untuk keamanan.
5. Untuk Mac/OrbStack bootcamp, L2 adalah hands-on lane; BGP adalah konsep dan desain production.

## Cek Pemahaman

1. Apa perbedaan fundamental antara ARP advertisement dan BGP route advertisement?
2. Apa peran `myASN`, `peerASN`, dan `peerAddress` dalam `BGPPeer`?
3. Mengapa beberapa Speaker boleh mengumumkan VIP yang sama pada BGP? Apa fungsi ECMP?
4. Sebutkan tiga risiko operasional jika route policy BGP tidak dibatasi.
5. Mengapa gateway rumah/kantor tidak boleh dijadikan BGP peer lab tanpa otorisasi?
6. Kapan L2 lebih tepat daripada BGP?
