# Modul 2.3 — MetalLB: LoadBalancer Bare-Metal

> **Tujuan akhir:** memberi cluster k3s on-prem kemampuan `Service type=LoadBalancer` — memasang MetalLB L2 (ARP/NDP), mengalokasikan IP pool yang aman, mengekspos app yang bisa diakses dari Mac, dan memahami BGP sebagai pilihan production.

## Capaian Modul (Wajib)

- [x] Bisa menjelaskan kenapa on-prem butuh MetalLB (cloud LB tidak tersedia)
- [x] Paham cara kerja L2 mode (ARP/NDP), IPAddressPool & L2Advertisement
- [x] Bisa install & integrasi MetalLB dengan k3s (servicelb disabled)
- [x] Bisa memilih IP pool yang tidak bentrok dengan DHCP/static IP/node
- [x] Bisa expose app via `Service type=LoadBalancer` & akses dari Mac lewat IP MetalLB
- [x] Bisa memakai `arping` untuk membuktikan advertisement IP
- [x] Paham BGP mode (peering, kapan dipakai production) & trade-off dengan L2
- [x] Bisa debug IP tidak assign, ARP conflict, strict ARP, dan traffic tidak sampai

## Rencana 2 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-kenapa-metallb](01-kenapa-metallb.md), [02-l2-mode-arp](02-l2-mode-arp.md), [04-konfigurasi-integrasi-k3s](04-konfigurasi-integrasi-k3s.md) | [LAB-01](lab/LAB-01-metallb-l2-expose.md) |
| 2 | [03-bgp-mode-konsep](03-bgp-mode-konsep.md), pendalaman troubleshooting | [LAB-02](lab/LAB-02-troubleshooting-arp.md) + [Latihan](evaluasi/latihan.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 2.2 selesai: cluster k3s HA (≥1 server, ideal 3 server + agent) di VM OrbStack
- **ServiceLB (klipper) sudah disabled** di semua server (`--disable servicelb`)
- IP OrbStack Machine stabil; tahu subnet VM (`orb ip k3s-cp1`)
- `kubectl` context k3s aktif dari Mac; paham Service, Ingress, label selector
- `arping` terpasang di Mac (`brew install arping`) atau tersedia di VM
- Sudah membaca [Fase 2 README](../README.md)

## Deliverables Modul

1. **MetalLB terpasang** di cluster k3s (controller + speaker `Running`).
2. **IPAddressPool + L2Advertisement** terkonfigurasi dengan range yang tidak bentrok.
3. **Service `type=LoadBalancer`** mendapat EXTERNAL-IP MetalLB dan app bisa di-curl dari Mac.
4. **Bukti ARP** (`arping`, `ip neigh`) dan catatan troubleshooting di `m2.3/lab/`.
5. **Diagram jalur traffic** Mac → VIP MetalLB → node → Service → Pod.
6. **Nilai kuis ≥ 80%**

## Cara Memulai

Modul 2.2 meninggalkan Service `type=LoadBalancer` dengan `EXTERNAL-IP: <pending>` karena `servicelb` dinonaktifkan — memang sengaja. Di cloud, cloud-controller-manager meminta IP ELB ke provider. Di on-prem tidak ada provider; **MetalLB menjadi provider LoadBalancer Anda sendiri**. Hari pertama memasang mode L2 (sederhana, ideal lab), hari kedua memahami BGP (cara production) dan membongkar kegagalan ARP/traffic. Jalankan lab di cluster k3s VM, bukan k3d — MetalLB L2 perlu jaringan VM yang bisa mengiklankan ARP.

## Kaitan dengan Modul Berikutnya

- **Service LoadBalancer yang sudah bekerja** menjadi fondasi Modul 2.4 (operasi & troubleshooting: debug Pending, OOMKilled, snapshot etcd).
- **ARP/L2 & IP pool** di sini = network foundation dari Modul 0.2 (ARP, CIDR, routing); kini diterapkan ke Kubernetes.
- **BGP** di sini = jembatan ke networking production; tidak diimplementasikan penuh di laptop, tapi Anda tahu kapan perlu router peering.
- **MetalLB + k3s** = on-prem cluster siap menerima Ingress/observability (Fase 5–7). ArgoCD nanti deploy manifest MetalLB secara GitOps (Fase 6).