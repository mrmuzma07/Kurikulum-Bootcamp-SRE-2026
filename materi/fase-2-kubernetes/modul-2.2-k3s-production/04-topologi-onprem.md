# 04 — Topologi On-Prem di OrbStack

> Merancang jaringan "mirip server sungguhan" di Mac: static IP, jalur akses dari luar, dan posisi LoadBalancer eksternal — semua sebagai simulasi sebelum MetalLB (Modul 2.3).

## Tujuan
- Bisa merancang topologi on-prem (control plane, worker, akses eksternal) di OrbStack
- Paham static IP vs DHCP & kenapa IP stabil penting untuk K8s on-prem
- Mengerti posisi external LoadBalancer & ingress dalam topologi
- Bisa menjelaskan jalur traffic: Mac → LB → ingress → Service → Pod
- Bisa membandingkan topologi cloud (auto-LB) vs on-prem (self-hosted LB)

## 1. Kenapa Topologi Penting?

Di cloud, detail jaringan "tersembunyi": Service `type=LoadBalancer` otomatis dapat IP publik dari cloud provider, DNS otomatis, node di VPC. Di **on-prem**, Anda **menjadi** network engineer: atur IP, pilih range LoadBalancer, konfigurasi firewall, dan pasang LB sendiri. Bootcamp ini menyimulasikannya di OrbStack — VM dengan IP stabil berperan sebagai server di datacenter.

Paham topologi = paham kenapa Modul 2.3 (MetalLB) perlu, kenapa static IP penting (Ansible inventory — Fase 4), dan kenapa ingress diletakkan di depan.

## 2. IP Stabil — Fondasi On-Prem

**DHCP** (IP berubah) = musuh K8s on-prem. Control plane, etcd peer, Ansible target — semua butuh **IP stabil**. OrbStack Machine punya IP stabil (Modul 1.2), mensimulasikan static IP.

```bash
# IP OrbStack Machine tidak berubah saat stop/start (bukan DHCP)
orb ip k3s-cp1
orb stop k3s-cp1 && orb start k3s-cp1 && orb ip k3s-cp1   # IP sama
```

**Di production nyata** (server fisik/Proxmox): set static IP via netplan/networkd, atau DHCP reservation (IP-MAC binding). OrbStack abstraksi ini — kita anggap IP stabil = static.

```
 Sumber IP on-prem (urut dari "paling produksi"):
 1. Static manual di /etc/netplan/        (server fisik)
 2. DHCP reservation (IP-MAC binding)    (VM di Proxmox/VMware)
 3. OrbStack Machine IP stabil           (lab bootcamp — simulasi static)
```

## 3. Topologi Lab — Jaringan "Datacenter Mini"

```
 ┌──────────────────── Mac (host) ─────────────────────────┐
 │                                                          │
 │   Browser/curl dari Mac                                   │
 │        │                                                  │
 │        │  (1) akses via MetalLB IP (Modul 2.3)            │
 │        │      atau NodePort sementara                      │
 │        ▼                                                  │
 │   ┌─────────────┐   LoadBalancer (MetalLB L2, Modul 2.3)  │
 │   │  LB / VIP   │   IP dari pool (mis. 192.168.97.200)    │
 │   └──────┬──────┘                                         │
 │          │  (2) route ke ingress controller Pod           │
 │          ▼                                                  │
 │   ┌─────────────┐   Ingress (Traefik/nginx)               │
 │   │  Ingress    │   host/path routing                      │
 │   └──────┬──────┘                                         │
 │          │  (3) backend Service                            │
 │          ▼                                                  │
 │   ┌──────┴──────┬────────────┐                              │
 │   │  Service A  │  Service B │   (ClusterIP, internal)     │
 │   └──────┬──────┴──────┬─────┘                              │
 │          │              │                                  │
 │          ▼              ▼                                  │
 │   ┌──────────────────────────┐   Pod (di worker node)     │
 │   │  k3s cluster (3cp + 2w)  │                            │
 │   │  192.168.97.10–14 (VM)   │                            │
 │   └──────────────────────────┘                            │
 │                                                          │
 └──────────────────────────────────────────────────────────┘
```

Empat lapisan akses (penting dihafal):
1. **External entry** — Mac client → LB (MetalLB IP atau NodePort sementara)
2. **LoadBalancer** — distributes traffic, beri IP eksternal (Modul 2.3)
3. **Ingress** — HTTP layer 7, route via host/path ke Service
4. **Service** — layer 4, route ke Pod via label selector

## 4. Range IP & Mencegah Bentrok

Pilih range IP LoadBalancer (pool MetalLB) yang **tidak dipakai DHCP/static lain**. Di OrbStack, VM dapat IP di subnet OrbStack (mis. `192.168.97.0/24`).

```bash
# Cek range OrbStack
orb ip k3s-cp1                      # mis. 192.168.97.10
# Pool MetalLB (Modul 2.3) pilih range yang tidak dipakai VM, mis.:
#   192.168.97.200–192.168.97.250   (asumsi VM di .10–.30)
```

**Aturan on-prem:** alokasikan range terpisah untuk:
- **Node IP** (static, server fisik/VM) — mis. `.10–.19`
- **Service ClusterIP** (otomatis k3s, `10.43.0.0/16`)
- **Pod IP** (otomatis k3s, `10.42.0.0/16`)
- **LoadBalancer IP pool** (MetalLB) — mis. `.200–.250`

Bentrok IP = fault misterius (ARP conflict, traffic ke salah tujuan). Dokumentasikan range di `m2.2/topologi.md`.

## 5. Jalur Lengkap Traffic (Mac → Pod)

Skenario: pengguna Mac membuka `http://app.k3s.local`.

```
 1. Mac resolve app.k3s.local (via /etc/hosts → 192.168.97.200, IP MetalLB)
 2. Mac → 192.168.97.200:80 (LoadBalancer IP, dipegang salah satu node via ARP)
 3. Node itu route ke Pod ingress-controller (Traefik/nginx)
 4. Ingress cocokkan host "app.k3s.local" → Service "app"
 5. Service "app" (ClusterIP) → Pod app (label selector)
 6. Pod app balas "hello..." → kembali lewat jalur yang sama
```

Tanpa MetalLB (sebelum Modul 2.3), langkah 1–3 diganti `NodePort` (`VMIP:30000+`) — bekerja tapi tidak "production-like" (port tinggi, tidak ada IP virtual).

## 6. Cloud vs On-Prem — Perbandingan

| Aspek | Cloud (EKS/GKE) | On-prem (k3s, bootcamp) |
|---|---|---|
| Node IP | VPC otomatis | **Anda atur** (static) |
| LoadBalancer | cloud provider (ELB) | **Anda pasang** (MetalLB) |
| DNS | Route53 otomatis | **Anda kelola** (CoreDNS/hosts) |
| Storage | EBS/GPD otomatis | **Anda kelola** (local-path/NFS) |
| Certificate | cert-manager + cloud DNS | cert-manager + DNS challenge manual |

Inti: on-prem = **anda menjadi infra provider**. Bootcamp ini melatih peran itu (IaC Fase 3, Ansible Fase 4, observability Fase 7) — semuanya yang di cloud "gratis" dari provider.

## 7. Diagram & Dokumentasi (Deliverable)

Buat `m2.2/topologi.md` di repo — diagram & tabel IP:

```markdown
# m2.2 Topologi k3s HA di OrbStack

## Node
| Nama      | IP             | Peran           | Catatan              |
|-----------|----------------|-----------------|----------------------|
| k3s-cp1   | 192.168.97.10  | server (etcd)   | cluster-init         |
| k3s-cp2   | 192.168.97.11  | server (etcd)   | join                 |
| k3s-cp3   | 192.168.97.12  | server (etcd)   | join                 |
| k3s-w1    | 192.168.97.20  | agent (worker)  | Pod jalan            |
| k3s-w2    | 192.168.97.21  | agent (worker)  | Pod jalan            |

## Range IP
- Node: 192.168.97.10–.21
- ClusterIP (Service): 10.43.0.0/16 (default k3s)
- Pod: 10.42.0.0/16 (default k3s)
- LoadBalancer pool (MetalLB, Modul 2.3): 192.168.97.200–.250

## Akses dari Mac
- kubectl: kubeconfig context `k3s-ha` → CP1:6443
- HTTP (nanti): app.k3s.local → 192.168.97.200 (MetalLB) → ingress → Service → Pod

## Komponen disable
- servicelb (siap MetalLB)
- (traefik tetap, atau diganti nginx — catat pilihan)
```

Dokumentasi topologi = deliverable SRE. Ansible (Fase 4) membaca "inventory" seperti tabel ini.

## 8. k3d vs k3s — Ringkasan Akhir

| Aspek | k3d (Modul 2.1) | k3s di VM (modul ini) |
|---|---|---|
| Tujuan | latihan harian, iterasi cepat | simulasi production on-prem |
| "Rumah" | container Docker | VM Linux (systemd) |
| IP node | container (berubah) | VM (stabil = static-like) |
| MetalLB L2 | sulit (ARP container) | bisa (jaringan VM) |
| HA control plane | `--servers 3` (etcd container) | `--cluster-init` + join (etcd VM) |
| Ansible target | tidak | ya |
| etcd backup (2.4) | kurang relevan | inti |
| Pakai di bootcamp | 2.1, sebagian 2.4 | 2.2, 2.3, 2.4, Fase 4–6 |

**Sekarang kedua lane solid.** Fase 6 GitOps akan menggabungkan: k3d = staging, k3s = production.

## Latihan Cepat (15 menit)

```bash
# 1. Catat IP semua node (kalau sudah punya cluster 2.2)
for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do echo "$n: $(orb ip $n)"; done 2>/dev/null

# 2. Cek IP stabil (stop/start 1 VM)
orb stop k3s-cp1 2>/dev/null; orb start k3s-cp1; orb ip k3s-cp1

# 3. Rencanakan range (tulis di m2.2/topologi.md)
#   - Node IP yang ada
#   - Pilih range LB pool yang tidak bentrok

# 4. (Tanpa MetalLB) akses app via NodePort sebagai placeholder
kubectl create deployment web --image=nginx:alpine -n default 2>/dev/null
kubectl expose deployment web --port=80 --type=NodePort -n default 2>/dev/null
kubectl get svc web -n default
NODEPORT=$(kubectl get svc web -n default -o jsonpath='{.spec.ports[0].nodePort}')
curl -s http://$(orb ip k3s-cp1):$NODEPORT/ | head -1
```

## Ringkasan

| Mau paham... | Inti |
|---|---|
| Kenapa IP stabil | K8s on-prem butuh node IP tetap (etcd peer, Ansible) |
| Lapisan akses | external → LB → ingress → Service → Pod |
| Range IP | node / ClusterIP / Pod / LB pool — terpisah, tidak bentrok |
| Cloud vs on-prem | cloud = gratis; on-prem = anda jadi infra provider |
| Dokumentasi | topologi.md = deliverable; jadi inventory Ansible (Fase 4) |

## Cek Pemahaman

1. Kenapa DHCP (IP berubah) bermasalah untuk control plane K8s on-prem?
2. Sebut 4 lapisan jalur traffic dari Mac client ke Pod (urut).
3. Kenapa range IP LoadBalancer (MetalLB pool) harus terpisah dari node IP?
4. Di cloud, `Service type=LoadBalancer` otomatis dapat IP. Di on-prem tanpa MetalLB, apa penggantinya sementara? Dan apa solusi "production-like" (modul mana)?
5. Sebut 1 alasan documentation topologi penting untuk Ansible (Fase 4).