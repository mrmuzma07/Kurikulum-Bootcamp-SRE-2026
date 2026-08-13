# 01 — Kenapa MetalLB?

> Di cloud, LoadBalancer datang dari provider. Di bare-metal, tidak ada siapa-siapa — MetalLB mengisi kekosongan itu.

## Tujuan
- Bisa menjelaskan perbedaan `Service type=LoadBalancer` di cloud vs on-prem
- Paham problem yang diselesaikan MetalLB
- Bisa membedakan ClusterIP, NodePort & LoadBalancer dalam konteks akses external
- Mengerti komponen MetalLB (controller + speaker) secara konseptual
- Tahu mengapa k3d tidak cocok untuk lab MetalLB L2

## 1. Problem: Service LoadBalancer `<pending>`

Kubernetes punya abstraksi:

```yaml
apiVersion: v1
kind: Service
metadata: {name: web}
spec:
  type: LoadBalancer
  selector: {app: web}
  ports: [{port: 80, targetPort: 8080}]
```

Di **cloud** (EKS, GKE, AKS):
```
 kubectl apply Service LoadBalancer
      │
      ▼
 cloud-controller-manager → API cloud provider
      │                         (AWS ELB / GCP LB / Azure LB)
      ▼
 EXTERNAL-IP: 34.x.x.x   ← provider bikin LB & assign IP
```

Di **on-prem/bare-metal** (k3s VM tanpa addon):
```
 kubectl apply Service LoadBalancer
      │
      ▼
 tidak ada cloud provider → tidak ada yang assign IP
      │
      ▼
 EXTERNAL-IP: <pending>  ← user tidak bisa akses dari luar
```

Inilah kondisi yang Anda lihat di Modul 2.2 setelah `--disable servicelb`: **bukan error Kubernetes**, tapi tidak ada implementasi LoadBalancer.

## 2. Tiga Cara Ekspos Service

| Type | Endpoint | Kelebihan | Kekurangan / konteks |
|---|---|---|---|
| **ClusterIP** | IP virtual internal (`10.43.x.x`) | aman, internal, default | tidak bisa diakses dari luar cluster |
| **NodePort** | `NodeIP:30000–32767` | selalu tersedia, simpel untuk debug | port tinggi, satu port per service, expose node langsung |
| **LoadBalancer** | IP external/VIP (`192.168.x.x`) | endpoint standar, port 80/443, stabil | butuh provider LB (cloud/MetalLB) |
| **Ingress** | satu IP + host/path ke banyak Service | hemat VIP, HTTP routing, TLS | butuh ingress controller; bukan pengganti LB (Ingress sendiri perlu endpoint) |

```bash
# Bandingkan di cluster
kubectl get svc
# NAME  TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)
# web   ClusterIP      10.43.x.x    <none>        80/TCP
# api   NodePort       10.43.x.x    <none>        80:30080/TCP
# lb    LoadBalancer   10.43.x.x    <pending>     80:.../TCP
```

NodePort bisa menjadi workaround, tapi production on-prem butuh IP virtual yang stabil & port standar — MetalLB.

## 3. MetalLB = Implementasi LoadBalancer untuk Bare-Metal

**MetalLB** (Metal Load Balancer) adalah implementasi LoadBalancer Kubernetes untuk cluster bare-metal. Ia terdiri dari:

```
 ┌─ MetalLB Controller (1 Pod, control plane) ────────────────┐
 │ - watch Service type=LoadBalancer                          │
 │ - pilih IP bebas dari IPAddressPool                         │
 │ - assign status.loadBalancer.ingress.ip                     │
 └────────────────────────────────────────────────────────────┘
          │ assign & inform
          ▼
 ┌─ MetalLB Speaker (DaemonSet, tiap node) ───────────────────┐
 │ - advertise IP ke network (L2: ARP/NDP; BGP: peering)       │
 │ - satu speaker aktif untuk tiap VIP (leader election)       │
 │ - terima traffic, route ke kube-proxy/Service               │
 └────────────────────────────────────────────────────────────┘
```

- **Controller** = otak allocation: pilih IP, update Service `status.loadBalancer.ingress`.
- **Speaker** = network agent: iklankan IP ke jaringan. Ada di setiap node; leader election menentukan node yang jawab ARP untuk VIP.
- **IPAddressPool** = daftar/range IP yang boleh dipinjam Service.
- **Advertisement** = metode mengumumkan IP (L2 atau BGP).

## 4. Apa yang MetalLB Bukan

- Bukan Ingress controller (tidak route host/path HTTP). MetalLB hanya memberi **IP + reachability** ke Service LoadBalancer. Traefik/nginx tetap route HTTP.
- Bukan external hardware load balancer penuh (tidak punya WAF, TLS termination otomatis, health UI). Ia mengintegrasikan IP ke K8s network.
- Bukan pengganti Service/Ingress. Ia implementasi `type=LoadBalancer`.
- Tidak mengelola DNS. Anda tetap atur DNS/`/etc/hosts` → VIP.

```
 Mac curl app.k3s.local
   │ DNS (/etc/hosts) → 192.168.97.200 (MetalLB VIP)
   ▼
 MetalLB (IP reachability) → Traefik Ingress (host/path)
                              ▼
                         Service → Pod
```

## 5. Mengapa k3d Tidak Ideal untuk MetalLB L2?

k3d = k3s sebagai container Docker. L2 MetalLB butuh **ARP broadcast** ke jaringan tempat client berada:

- Speaker mengirim ARP "192.168.97.200 ada di MAC saya".
- Di VM OrbStack, network VM punya jalur ke Mac (bisa ARP/neighbor).
- Di k3d, speaker ada di network container; ARP berhenti di bridge Docker, tidak terpropagasi ke network Mac seperti server bare-metal.

Akibat: MetalLB mungkin assign IP (status terlihat), tapi `curl` dari Mac tidak sampai (ARP tidak resolve). Gunakan **k3s di VM** (Modul 2.2) untuk lab ini.

## 6. Syarat Sebelum Install

Checklist:

- [ ] `servicelb` (klipper) **disabled** di semua k3s server. Kalau aktif, dua controller berebut LoadBalancer.
- [ ] Cluster k3s **Ready** & context benar (`kubectl config current-context`).
- [ ] Pilih IP pool satu subnet dengan node/client (L2) atau reachable dari router.
- [ ] IP pool tidak dipakai node, DHCP, service lain, atau host Mac.
- [ ] Firewall mengizinkan ARP (L2) & traffic TCP 80/443.
- [ ] Tidak ada IP conflict (cek `arping`, `ip neigh`).

## 7. Alur MetalLB End-to-End

```
 1. kubectl apply Service type=LoadBalancer
 2. Controller melihat Service baru
 3. Controller pilih IP bebas: 192.168.97.200 dari IPAddressPool
 4. Controller update Service status: EXTERNAL-IP=192.168.97.200
 5. Speaker leader mengiklankan VIP (ARP L2 / BGP route)
 6. Client resolve app.k3s.local → VIP
 7. Traffic ke node speaker → kube-proxy → Service → Pod
```

**Catatan:** assignment IP (controller) & advertisement (speaker) adalah dua tahap terpisah. Jika `EXTERNAL-IP` ada tapi traffic gagal → masalah speaker/ARP/firewall (bukan allocation).

## Latihan Cepat (15 menit)

```bash
# 1. Pastikan context k3s (bukan k3d!)
kubectl config current-context
kubectl get nodes -o wide

# 2. Lihat Service pending sebelum MetalLB
kubectl get svc -A | grep LoadBalancer

# 3. Catat network
orb ip k3s-cp1
route -n get $(orb ip k3s-cp1) 2>/dev/null | head
# pilih pool di subnet yang benar (topik 04)

# 4. Lihat komponen servicelb (harus kosong)
kubectl get pods -A | grep svclb || echo "servicelb disabled — siap MetalLB"
```

## Ringkasan

| Pertanyaan | Jawaban |
|---|---|
| Kenapa `<pending>`? | on-prem tidak punya cloud-controller/LB provider |
| MetalLB memberi apa? | IP external + advertisement/reachability untuk Service LB |
| Siapa pilih IP? | Controller dari IPAddressPool |
| Siapa iklankan IP? | Speaker (L2 ARP/NDP atau BGP) |
| MetalLB = Ingress? | Tidak; MetalLB Layer 3/4, Ingress Layer 7 |
| L2 lab di mana? | k3s VM (bisa ARP), bukan k3d container |

## Cek Pemahaman

1. Mengapa Service LoadBalancer `<pending>` di on-prem tanpa MetalLB? Apa yang otomatis dilakukan cloud provider?
2. Bedakan peran Controller vs Speaker MetalLB.
3. Mengapa NodePort bukan solusi production yang ideal dibanding MetalLB?
4. MetalLB memberi IP tapi curl timeout. Apakah itu pasti controller bug? (petakan tahap alur yang mungkin gagal)
5. Mengapa lab MetalLB L2 menggunakan k3s VM, bukan k3d?