# Kuis & Jawaban — Modul 2.3 MetalLB

> Target kelulusan: **minimal 80%**. Jawab bagian soal terlebih dahulu, lalu buka kunci untuk review.

## A. Pilihan Ganda

### 1. Mengapa `EXTERNAL-IP` Service LoadBalancer tetap `<pending>` pada k3s bare-metal dengan `servicelb` disabled?

A. Pod belum memiliki label
B. Tidak ada implementasi/provider LoadBalancer yang mengalokasikan dan mengiklankan IP
C. ClusterIP selalu pending
D. Ingress controller harus dipasang lebih dulu

### 2. Resource MetalLB yang memilih IP bebas untuk Service adalah …

A. Speaker
B. L2Advertisement
C. Controller
D. IngressClass

### 3. Fungsi utama `IPAddressPool` adalah …

A. Menentukan range IP yang boleh dialokasikan
B. Menjawab ARP secara langsung
C. Menyimpan Secret registry
D. Mengatur replica Deployment

### 4. Fungsi `L2Advertisement` adalah …

A. Menghubungkan pool dengan advertisement ARP/NDP
B. Membuat Pod web
C. Menentukan ASN router BGP saja
D. Mengubah ClusterIP menjadi NodePort

### 5. Mengapa pool L2 harus berada pada subnet yang sesuai?

A. ARP broadcast tidak dirutekan melewati router secara default
B. Kubernetes hanya mendukung subnet `/8`
C. Speaker hanya bisa memakai alamat `127.0.0.1`
D. ClusterIP harus sama dengan VIP

### 6. Pada satu VIP mode L2, berapa Speaker yang normalnya aktif menjawab ARP pada satu waktu?

A. Semua Speaker tanpa koordinasi
B. Satu Speaker leader
C. Tidak ada Speaker
D. Hanya controller

### 7. Saat leader Speaker mati, perilaku yang diharapkan adalah …

A. Controller menghapus semua Service
B. Speaker lain melakukan election dan mengambil alih advertisement setelah convergence
C. VIP otomatis berubah menjadi ClusterIP
D. Router cloud membuat ELB

### 8. Pernyataan yang tepat tentang MetalLB dan Ingress adalah …

A. MetalLB menggantikan routing host/path Ingress
B. Ingress menggantikan IP allocation MetalLB
C. MetalLB memberi reachability Service LB; Ingress controller melakukan routing HTTP
D. Keduanya resource yang sama

### 9. Keuntungan utama BGP dibanding L2 untuk production multi-subnet adalah …

A. Tidak memerlukan router
B. Route VIP dapat diumumkan ke router dan beberapa path dapat digunakan
C. Selalu lebih mudah dikonfigurasi
D. Menghapus kebutuhan Service

### 10. Port standar session BGP adalah …

A. TCP/22
B. TCP/53
C. TCP/179
D. UDP/4789

### 11. `peerASN` pada `BGPPeer` menyatakan …

A. ASN router yang menjadi peer
B. Port Service
C. CIDR Pod
D. Jumlah replica

### 12. Dengan `externalTrafficPolicy: Local`, risiko yang perlu diperhatikan adalah …

A. Source IP lebih mungkin dipertahankan, tetapi node penerima perlu memiliki endpoint lokal
B. Semua traffic pasti di-NAT
C. Tidak ada Service selector
D. ARP tidak lagi digunakan dalam mode L2

### 13. Jika `EXTERNAL-IP` sudah ada tetapi `arping` timeout, tahap yang paling relevan diperiksa adalah …

A. Allocation saja
B. Speaker, advertisement, subnet, interface, dan firewall/L2
C. Git commit message saja
D. ReplicaSet history saja

### 14. IP conflict pada VIP berarti …

A. Dua pihak menjawab/menggunakan alamat yang sama
B. Service tidak punya ClusterIP
C. BGP selalu Established
D. Pod pasti OOMKilled

### 15. Mengapa `servicelb` k3s perlu disabled sebelum MetalLB?

A. Agar dua mekanisme LoadBalancer tidak berebut mengelola Service
B. Agar kubelet berhenti
C. Agar etcd tidak memiliki quorum
D. Agar DNS internal mati

### 16. Perintah paling aman untuk memastikan target cluster sebelum perubahan adalah …

A. `kubectl delete -A`
B. `kubectl config current-context`
C. `docker system prune -a`
D. `sudo rm -rf /var/lib/rancher`

## B. Esai Singkat

### 17. Jelaskan alur lengkap dari `kubectl apply Service type=LoadBalancer` sampai client menerima HTTP response. Bedakan peran Controller, Speaker, kube-proxy/Service, dan Pod.

### 18. Bandingkan L2 ARP dengan BGP dari sisi mekanisme, batas network, failover, kompleksitas, dan kapan digunakan.

### 19. Service mendapat VIP, `arping` berhasil, tetapi curl timeout. Tulis minimal lima command diagnosis berurutan dan jelaskan evidence yang dicari.

### 20. Mengapa IP pool tidak boleh overlap DHCP pool, IP node, gateway, atau perangkat lain? Jelaskan dampak operationalnya dan cara pencegahan.

## Kunci Jawaban Pilihan Ganda

| No. | Jawaban | Penjelasan |
|---:|:---:|---|
| 1 | B | Bare-metal tanpa provider tidak punya komponen yang mengalokasikan/mengiklankan VIP. |
| 2 | C | Controller watch Service dan memilih IP dari pool. |
| 3 | A | Pool adalah sumber alamat yang diizinkan untuk assignment. |
| 4 | A | Advertisement mengaitkan pool dengan mekanisme L2. |
| 5 | A | ARP adalah broadcast Layer 2; router tidak meneruskannya seperti traffic routed biasa. |
| 6 | B | Leader election mencegah banyak node menjawab VIP secara bersamaan dalam L2. |
| 7 | B | Speaker lain mengambil alih setelah election/convergence; koneksi aktif dapat reset. |
| 8 | C | MetalLB memberi endpoint/reachability; Ingress melakukan routing Layer 7. |
| 9 | B | BGP mengumumkan prefix ke router dan mendukung multi-path sesuai router/policy. |
| 10 | C | BGP menggunakan TCP port 179. |
| 11 | A | `peerASN` adalah ASN perangkat/router lawan. |
| 12 | A | Local mempertahankan source IP, tetapi memerlukan endpoint di node penerima. |
| 13 | B | Allocation selesai; fokus berpindah ke advertisement dan jalur network. |
| 14 | A | Ada lebih dari satu pemilik/pemberi jawaban untuk VIP yang sama. |
| 15 | A | Dua controller dapat berebut assignment/handling LoadBalancer. |
| 16 | B | Verifikasi context adalah guardrail sebelum operasi. |

## Pedoman Jawaban Esai

### 17. Alur End-to-End

Jawaban baik menyebut: API server menerima Service; MetalLB Controller memilih VIP dari `IPAddressPool` dan memperbarui status; Speaker mengiklankan VIP melalui ARP/NDP atau BGP; client menemukan next-hop/MAC; traffic masuk ke node; kube-proxy/Service memilih endpoint; Pod memproses request; response kembali melalui jalur network. Assignment dan advertisement adalah tahap berbeda.

### 18. L2 vs BGP

Jawaban minimal: L2 memakai ARP/NDP dan satu broadcast domain, satu leader per VIP, mudah untuk lab; BGP mengiklankan prefix melalui TCP/179 ke router, dapat lintas subnet dan ECMP, tetapi membutuhkan ASN/peer/policy serta koordinasi network. Keduanya memiliki mekanisme failover dan operational trade-off.

### 19. VIP Ada, ARP Berhasil, Curl Gagal

Contoh urutan:

```bash
kubectl describe svc web -n metallb-lab
kubectl get endpointslice -n metallb-lab -l kubernetes.io/service-name=web -o wide
kubectl get pods -n metallb-lab -o wide
kubectl get pods -n metallb-system -o wide
kubectl logs -n metallb-system -l component=speaker --tail=100
curl -v --connect-timeout 5 http://<VIP>/
```

Jawaban harus membedakan port/targetPort, endpoint Ready, speaker, firewall/TCP, kube-proxy, dan `externalTrafficPolicy`.

### 20. IP Pool Conflict

Overlap dapat membuat MetalLB mengklaim alamat milik DHCP/perangkat/node; terjadi ARP flapping, koneksi intermittent, salah arah traffic, atau gangguan perangkat lain. Pencegahan: inventory subnet, reservation network, pool khusus, review perubahan, dan `arping` sebelum menggunakan kandidat.

## Rubrik Esai

| Kriteria | Poin per soal |
|---|---:|
| Ketepatan konsep | 2 |
| Urutan diagnosis/causal reasoning | 1 |
| Command/evidence relevan | 1 |
| Pertimbangan safety/production | 1 |

Total: 16 pilihan ganda + 4 × 5 esai = 36 poin. Nilai kelulusan 80% = minimal 29 poin, atau ikuti rubrik instruktur.

## Remediasi

Jika nilai <80%:

1. Ulangi diagram Controller vs Speaker.
2. Praktikkan satu Service pending dan satu bukti `arping`.
3. Baca ulang [02 L2](../02-l2-mode-arp.md), [03 BGP](../03-bgp-mode-konsep.md), dan [04 Integrasi k3s](../04-konfigurasi-integrasi-k3s.md).
4. Ulangi dua soal esai dengan evidence command.
5. Review bersama instruktur sebelum masuk Modul 2.4.
