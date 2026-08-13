# 02 — L2 Mode: ARP/NDP & Advertisement

> Cara paling sederhana MetalLB mengumumkan VIP: satu node menjawab ARP (IPv4) atau NDP (IPv6) atas nama Service.

## Tujuan
- Paham ARP (IPv4) & NDP (IPv6) dalam konteks MetalLB
- Bisa menjelaskan cara kerja L2 mode & leader election
- Bisa membuat IPAddressPool + L2Advertisement
- Paham trade-off L2: simpel vs single-node bottleneck/failover
- Bisa menggunakan `arping`, `ip neigh`, tcpdump untuk verifikasi

## 1. ARP — "Siapa Punya IP Ini?"

Di jaringan IPv4, komputer mengirim paket ke **MAC address**, bukan IP langsung. ARP (Address Resolution Protocol) memetakan IP → MAC:

```
 Mac ingin kirim ke 192.168.97.200 (VIP MetalLB)
      │
      ├─ ARP Request (broadcast): "Who has 192.168.97.200?"
      │  ff:ff:ff:ff:ff:ff (semua device di L2 network)
      │
      └─ MetalLB Speaker di node cp2:
         ARP Reply (unicast): "192.168.97.200 is at AA:BB:CC:..."
              │
              ▼
         Mac simpan IP→MAC di ARP cache (neighbor table)
         Mac kirim TCP packet ke MAC cp2
```

```bash
# Mac: lihat ARP/neighbor cache
arp -a
# atau:
ndp -a                         # IPv6 neighbor

# Cari siapa menjawab VIP (jalankan dari Mac/host satu subnet)
sudo arping -I <interface> 192.168.97.200
# Response from 192.168.97.200 [aa:bb:cc:dd:ee:ff]

# Linux VM:
ip neigh show
sudo arping -I eth0 192.168.97.200
```

**ARP hanya bekerja satu Layer-2 broadcast domain/subnet.** Router tidak meneruskan broadcast ARP ke subnet lain. Itu sebabnya IP pool L2 harus satu subnet yang reachable dari client & node.

## 2. NDP — Padanan IPv6

NDP (Neighbor Discovery Protocol) = mekanisme IPv6 yang menggantikan ARP, berbasis ICMPv6 Neighbor Solicitation/Advertisement. MetalLB L2 dapat advertise IPv4 via ARP & IPv6 via NDP. Bootcamp utama IPv4 (lebih umum on-prem lab), tapi konsep sama:

| | IPv4 | IPv6 |
|---|---|---|
| Resolusi | ARP | NDP (ICMPv6) |
| Query | Who has IP? | Neighbor Solicitation |
| Jawab | ARP Reply | Neighbor Advertisement |
| Tool | `arping` | `ndisc6`/`ping6` |

## 3. Bagaimana L2 Mode MetalLB Bekerja?

```
 ┌─ Controller ──────────────────────────────┐
 │ Service web type=LoadBalancer              │
 │ pilih 192.168.97.200 dari pool             │
 │ update status.loadBalancer.ingress.ip      │
 └────────────────────────────────────────────┘
                     │
                     ▼
 ┌─ Speaker di tiap node (DaemonSet) ─────────┐
 │ cp1: candidate (leader election)            │
 │ cp2: candidate                              │
 │ cp3: candidate                              │
 │ w1:  candidate                              │
 │ w2:  candidate                              │
 │                                            │
 │ Hanya leader (mis. cp2) jawab ARP VIP      │
 └────────────────────────────────────────────┘
                     │
                     ▼
          192.168.97.200 → MAC cp2
```

**Langkah:**
1. Controller watch Service `type=LoadBalancer` yang belum punya IP.
2. Pilih IP yang belum dipakai dari `IPAddressPool`.
3. Tulis `status.loadBalancer.ingress.ip` (muncul di `kubectl get svc`).
4. Speaker di semua node participate dalam **leader election** untuk VIP itu.
5. Satu speaker (leader) menjawab ARP/NDP atas nama VIP.
6. Traffic masuk ke node leader → kube-proxy/iptables → Service → Pod (bisa Pod di node lain).

## 4. IPAddressPool

Pool = range IP yang MetalLB boleh assign. Contoh CRD MetalLB modern (v0.13+):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.97.200-192.168.97.220
  # atau CIDR:
  # - 192.168.97.200/29       # .200–.207 (8 alamat)
  autoAssign: true
```

Aturan pool:
- IP harus **satu subnet L2** dengan node/client (mis. node `192.168.97.10`, pool `192.168.97.200` dalam `/24`).
- Jangan overlap DHCP pool, node IP, gateway, atau pool lain.
- Kapasitas = jumlah IP; 21 IP di atas = max 21 Service LB (satu VIP per Service default).
- `autoAssign: false` = pool hanya dipakai Service yang minta via annotation (untuk IP mahal/special).

```yaml
# Service meminta pool tertentu (kalau ada beberapa pool)
metadata:
  annotations:
    metallb.io/address-pool: production-pool
```

## 5. L2Advertisement

Pool hanya **alokasi**. Advertisement bilang "pool ini diiklankan via L2".

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: production-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
  # nodeSelectors opsional: hanya node berlabel tertentu advertise
  # nodeSelectors:
  # - matchLabels:
  #     metallb-role: speaker
```

Jika `ipAddressPools` kosong, advertisement berlaku semua pool. Lebih eksplisit sebut pool untuk mencegah salah advertise.

```bash
kubectl apply -f ip-pool.yaml
kubectl apply -f l2-advertisement.yaml
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

## 6. Leader Election & Failover

L2 **bukan active-active** untuk satu VIP: satu node mengiklankan VIP pada satu waktu. Tapi semua node punya speaker; jika leader mati:

```
 1. cp2 leader jawab ARP VIP
 2. cp2 mati / speaker down
 3. speaker lain detect via memberlist/lease
 4. cp1 jadi leader → jawab ARP VIP
 5. client refresh ARP cache → traffic ke cp1
```

**Dampak:** failover butuh detik; koneksi TCP yang sedang berjalan bisa reset (client retry). New connections tetap jalan. Ini bukan load balancing per-packet — failover/active-standby per VIP. kube-proxy kemudian bisa route Pod ke node mana pun.

## 7. Service & `externalTrafficPolicy`

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster   # default
```

- **Cluster:** traffic dari speaker node bisa di-forward ke Pod di node lain. Load tersebar, tapi source IP client bisa di-NAT (Pod melihat node IP).
- **Local:** hanya route ke Pod **di node yang menerima traffic**. Source IP client dipertahankan, tapi kalau node leader tidak punya Pod → traffic drop (atau MetalLB speaker memilih node dengan endpoint, konfigurasi bergantung versi). Cocok untuk source IP logging/ACL, perlu DaemonSet/Pod tersebar.

```bash
kubectl patch svc app -p '{"spec":{"externalTrafficPolicy":"Local"}}'
kubectl get svc app -o jsonpath='{.spec.externalTrafficPolicy}'
```

**Untuk lab:** pakai `Cluster` default agar traffic selalu route meski Pod hanya di worker lain. Bahas `Local` sebagai keputusan production.

## 8. Verifikasi L2 Secara Layer-by-Layer

```bash
VIP=$(kubectl get svc app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $VIP

# Layer 1: Kubernetes allocation
kubectl get svc app -o wide
kubectl describe svc app                    # events, IP

# Layer 2: Speaker & advertisement
kubectl get pods -n metallb-system -o wide  # controller + speaker tiap node
kubectl logs -n metallb-system -l component=speaker --tail=50

# Layer 3: Host ARP
sudo arping -c 3 $VIP                      # ada ARP reply?
arp -a | grep $VIP
# Linux:
ip neigh show | grep $VIP

# Layer 4: TCP/HTTP
curl -v http://$VIP/health
```

Debug dari bawah ke atas: **allocation → speaker → ARP → TCP → HTTP**. Jangan langsung menyalahkan app kalau ARP saja tidak resolve.

## 9. L2 Limitations

| Kelebihan L2 | Keterbatasan L2 |
|---|---|
| mudah, tidak perlu router BGP | satu node aktif per VIP (traffic masuk tidak load-balance antar node) |
| tidak perlu peering/config router | failover ada gap detik, koneksi aktif reset |
| cocok lab & jaringan flat sederhana | hanya satu L2 subnet (ARP broadcast tidak lewat router) |
| semua router/switch dukung ARP | scale/routing production terbatas |

Untuk network production multi-subnet/traffic besar, BGP (topik 03) lebih tepat.

## Latihan Cepat (20 menit)

```bash
# Setelah MetalLB terpasang (LAB-01):
# 1. Dapatkan VIP
kubectl get svc -A | grep LoadBalancer
VIP=$(kubectl get svc web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 2. ARP dari Mac (ganti interface: en0/wlan)
IFACE=$(route get default | awk '/interface:/{print $2}')
sudo arping -I $IFACE -c 3 $VIP

# 3. Neighbor cache
arp -a | grep $VIP

# 4. Matikan node leader (lihat speaker logs / describe) — amati failover
kubectl get pods -n metallb-system -l component=speaker -o wide
# orb stop <node>; sleep 10; arping lagi; curl lagi
```

## Ringkasan

| Konsep | Inti |
|---|---|
| ARP | IP→MAC IPv4 via broadcast; harus satu L2 subnet |
| NDP | padanan IPv6 dari ARP |
| Controller | pilih IP dari pool, update Service status |
| Speaker | advertise VIP via ARP/NDP; DaemonSet tiap node |
| L2Advertisement | deklarasikan pool mana diiklankan via L2 |
| Leader election | satu speaker aktif per VIP; failover jika mati |
| `Cluster` policy | route ke Pod node lain; source IP mungkin NAT |
| `Local` policy | source IP terjaga; perlu Pod di node leader |

## Cek Pemahaman

1. Mengapa IP pool L2 harus satu subnet dengan node/client? Apa batasan ARP broadcast?
2. Bedakan IPAddressPool (alokasi) dan L2Advertisement (iklan).
3. Mengapa hanya satu speaker menjawab ARP per VIP? Apa yang terjadi saat leader mati?
4. Beda `externalTrafficPolicy: Cluster` vs `Local` — trade-off source IP dan reachability.
5. MetalLB `EXTERNAL-IP` sudah ada tapi `arping` timeout. Di layer mana debug dimulai?