# 03 — k3d: Kubernetes-in-Docker di OrbStack

> Cluster Kubernetes ringan yang jalan sebagai container — jembatan cepat dari container (Fase 1) ke Kubernetes (Fase 2).

## Tujuan
- Paham apa itu k3d & hubungannya dengan k3s
- Bisa install k3d & membuat/hapus cluster cepat
- Bisa membuat cluster multi-node & port mapping
- Bisa deploy app sederhana & mengaksesnya dari Mac
- Tahu kapan pakai k3d vs k3s

## 1. Apa Itu k3d?

**k3s** = distro Kubernetes ringan untuk production on-prem (Fase 2.2).
**k3d** = alat yang menjalankan **k3s di dalam container Docker** — bukan VM.

```
 ┌─ OrbStack ─────────────────────────────┐
 │  ┌─ container ──┐  ┌─ container ──┐     │
 │  │  k3s server  │  │  k3s agent   │ ... │  ← k3d menjalankan k3s sebagai container
 │  │  (control + │  │  (worker)    │     │
 │  │   kubelet)   │  │              │     │
 │  └──────────────┘  └──────────────┘     │
 └─────────────────────────────────────────┘
   ringan, mulai ~1 detik, tanpa VM tambahan
```

**Kenapa k3d untuk latihan harian:**
- Mulai **cepat** (detik, bukan menit seperti VM).
- Ringan: cluster multi-node = beberapa container, bukan beberapa VM.
- `k3d cluster delete` = bersih total (tidak nyangkut di disk).
- Cocok iterasi cepat: coba manifest, break, ulang.

**kapan k3s (VM OrbStack)**: simulasi production nyata — systemd, static IP, MetalLB bare-metal, Ansible bootstrap (Fase 2.2 & 4). VM = berat tapi "sungguhan".

## 2. Install k3d & kubectl

```bash
# k3d
brew install k3d
k3d version

# kubectl (kalau belum)
brew install kubectl
kubectl version --client
```

k3d butuh Docker API (OrbStack) — pastikan `docker version` jalan.

## 3. Membuat Cluster (Single Node)

```bash
# Cluster sederhana, 1 server
k3d cluster create demo --port 8080:80@loadbalancer
# --port 8080:80@loadbalancer: forward Mac:8080 → Traefik (loadbalancer k3d) :80
#   (k3d punya loadbalancer internal yang route ke Ingress Traefik)

kubectl get nodes
kubectl get pods -A                  # semua namespace; lihat Traefik, CoreDNS, dll
```

```bash
# Hapus cluster (bersih total)
k3d cluster delete demo
```

## 4. Cluster Multi-Node

```bash
k3d cluster create lab \
  --servers 1 \
  --agents 2 \
  --port 8080:80@loadbalancer \
  --port 8443:443@loadbalancer

kubectl get nodes
# NAME           STATUS   ROLES                  AGE   VERSION
# k3d-lab-server-0   Ready    control-plane,master   ...
# k3d-lab-agent-0     Ready    <none>                 ...
# k3d-lab-agent-1     Ready    <none>                 ...
```

| Opsi | Arti |
|---|---|
| `--servers N` | node control-plane (etcd + API) |
| `--agents N` | node worker (jalankan Pod) |
| `--port HOST:CONT@loadbalancer` | forward port Mac → LB k3d → Ingress |

**Catatan HA:** `--servers 3` = HA control plane (etcd clustered). Untuk lab, 1 server cukup. Untuk simulasi production (Fase 2.2) pakai k3s di VM, bukan k3d.

## 5. Deploy App & Akses dari Mac

```bash
# Pakai image dari Modul 1.1 LAB-01 (ganti dengan tag Anda)
REG=registry.gitlab.com/<username>/sre-bootcamp/app

# Deploy cepat via kubectl (Fase 2 akan pakai manifest/Helm)
kubectl create deployment demo --image="$REG:v1.0.0"
kubectl get deploy,pod
kubectl scale deployment demo --replicas=3
kubectl get pod -o wide                    # lihat Pod tersebar di node

# Expose via Ingress (Traefik bawaan k3d)
kubectl create service clusterip demo --tcp=8080:8080
# (Fase 2 bahas Ingress proper; di sini cukup lihat Pod jalan)
```

Atau uji paling minimal (nginx, tanpa registry):
```bash
kubectl create deployment web --image=nginx:alpine
kubectl expose deployment web --port=80
kubectl port-forward svc/web 8081:80      # Mac:8081 → service
# buka http://localhost:8081 di browser
# Ctrl+C untuk stop port-forward
```

## 6. Ingress (Port Mapping ke Mac)

k3d forward port ke **loadbalancer** internal yang route ke **Traefik Ingress** (bawaan k3s/k3d). Untuk akses via nama host:

```bash
# Tambahkan hosts (seperti Modul 0.2)
echo "127.0.0.1 demo.k3d.local" | sudo tee -a /etc/hosts

# Ingress manifest (Fase 2 detail; preview):
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
spec:
  rules:
  - host: demo.k3d.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
EOF

# Akses dari Mac (port 8080 sudah di-forward ke LB → Traefik)
curl -H 'Host: demo.k3d.local' http://localhost:8080/
```

## 7. Lifecycle & Troubleshooting k3d

```bash
k3d cluster list                          # daftar cluster
k3d cluster start lab                     # start (setelah stop)
k3d cluster stop lab
k3d cluster delete lab                    # hapus
k3d kubeconfig get lab                    # dapat kubeconfig
k3d kubeconfig merge lab --switch         # set context aktif

# Troubleshooting
kubectl get nodes                         # NotReady? tunggu sebentar / cek resource
kubectl describe node k3d-lab-server-0
docker ps | grep k3d                      # cluster = container, lihat di docker
kubectl get pods -A                       # pending? cek describe
```

```bash
# Kalau kubeconfig nyangkut/context salah:
k3d kubeconfig merge lab --switch
kubectl config use-context k3d-lab
kubectl config get-contexts
```

## 8. k3d vs k3s — Kapan Pakai Yang Mana

| Aspek | k3d (container) | k3s di VM OrbStack |
|---|---|---|
| Jalan sebagai | container Docker | VM Linux (systemd) |
| Startup | detik | menit |
| Resource | sangat ringan | lebih berat (VM) |
| Static IP | tidak (container IP) | ya (VM IP stabil) |
| MetalLB L2 | sulit (ARP di container) | bisa (VM di jaringan nyata) |
| Ansible target | tidak | ya |
| Cocok untuk | latihan harian, iterasi cepat, CI | simulasi production on-prem |

**Aturan bootcamp:**
- **Fase 2.1 (latihan Kubernetes)** → k3d (cepat, ringan).
- **Fase 2.2–2.3 (k3s multi-node, MetalLB)** → k3s di VM OrbStack (production-like).
- **Fase 6 GitOps** → k3d "staging" + k3s "production" (capstone).

## Latihan Cepat (20 menit)

```bash
# 1. Install (kalau belum)
brew install k3d kubectl

# 2. Cluster multi-node
k3d cluster delete lab 2>/dev/null
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
kubectl get nodes

# 3. Deploy nginx + port-forward
kubectl create deployment web --image=nginx:alpine
kubectl rollout status deployment/web
kubectl expose deployment web --port=80
kubectl port-forward svc/web 8081:80 &
sleep 2
curl -s http://localhost:8081/ | head -1
kill %1 2>/dev/null

# 4. Lihat Pod tersebar
kubectl scale deployment web --replicas=4
kubectl get pod -o wide

# 5. Bersihkan
k3d cluster delete lab
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Buat cluster | `k3d cluster create <name> --servers N --agents M --port H:C@loadbalancer` |
| Hapus cluster | `k3d cluster delete <name>` |
| Daftar cluster | `k3d cluster list` |
| Set kubeconfig | `k3d kubeconfig merge <name> --switch` |
| Lihat node | `kubectl get nodes` |
| Forward port | `kubectl port-forward svc/<x> LOCAL:PORT` |
| k3d vs k3s | k3d=cepat/harian, k3s di VM=production-like |

## Cek Pemahaman

1. Beda k3d vs k3s secara teknis (jalan sebagai apa)?
2. Kenapa k3d cocok untuk latihan harian tetapi **tidak ideal** untuk MetalLB L2 mode?
3. Apa fungsi `--port 8080:80@loadbalancer` saat `k3d cluster create`?
4. Anda butuh simulasi Ansible menginstal k3s di 3 server. k3d atau VM OrbStack? Kenapa?