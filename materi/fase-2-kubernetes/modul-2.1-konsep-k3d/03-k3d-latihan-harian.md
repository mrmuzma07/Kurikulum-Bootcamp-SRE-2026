# 03 — k3d sebagai Fast Lane Latihan Harian

> k3d = k3s dalam container; cepat, ringan, dan `delete`-able — rumah harian untuk eksperimen manifest.

## Tujuan
- Bisa membuat & menghapus cluster k3d dengan cepat (single & multi-node)
- Paham port mapping `@loadbalancer` & akses dari Mac
- Bisa mengelola banyak kubeconfig (k3d vs k3s)
- Bisa menjalankan lifecycle cluster (start/stop/delete) & membersihkan resource
- Tegas kapan k3d vs k3s (fast lane vs production lane)

## 1. Mengapa k3d untuk Modul Ini?

Fase 2 butuh banyak iterasi: coba manifest, break, perbaiki, ulang. k3d cocok karena:
- **Cepat**: `k3d cluster create` ~detik (k3s di VM = menit).
- **Ringan**: cluster multi-node = beberapa container, bukan beberapa VM.
- **Bersih total**: `k3d cluster delete` hapus segalanya; tidak nyangkut di disk.
- **Multi-node mudah**: `--servers 1 --agents 2` = 3 node sebagai container.

Modul 1.2 sudah perkenalan; sekarang k3d jadi alat utama untuk seluruh **Modul 2.1 & sebagian 2.4**. k3s di VM (production-like) datang di Modul 2.2.

## 2. Create & Delete Cluster

```bash
# Single node (cepat, untuk coba-coba)
k3d cluster create demo --port 8080:80@loadbalancer
kubectl get nodes

# Multi-node (simulasi distribusi Pod)
k3d cluster create lab \
  --servers 1 \
  --agents 2 \
  --port 8080:80@loadbalancer \
  --port 8443:443@loadbalancer
kubectl get nodes -o wide
# NAME                STATUS   ROLES                  AGE   VERSION
# k3d-lab-server-0   Ready    control-plane,master   ...
# k3d-lab-agent-0     Ready    <none>                 ...
# k3d-lab-agent-1     Ready    <none>                 ...

# Hapus (bersih total)
k3d cluster delete lab
docker system prune -f
```

| Opsi | Arti |
|---|---|
| `--servers N` | node control plane (etcd/API); `--servers 3` = HA |
| `--agents N` | node worker (jalankan Pod) |
| `--port H:C@loadbalancer` | forward Mac:H → LB k3d → Ingress:C |
| `--volume` | mount file/konfig ke node |
| `-p` alias `--port` | |

## 3. Port Mapping & Akses dari Mac

k3d punya **loadbalancer internal** yang route ke **Traefik** (Ingress controller bawaan k3s). Forward port Mac ke LB = akses Ingress dari Mac.

```bash
k3d cluster create lab --port 8080:80@loadbalancer --port 8443:443@loadbalancer
# Mac:8080 → LB:80 → Traefik → Service → Pod
```

```bash
echo "127.0.0.1 app.k3d.local" | sudo tee -a /etc/hosts
# (deploy app + Service + Ingress — lihat topik 02)
curl -H 'Host: app.k3d.local' http://localhost:8080/
```

**Tanpa Ingress** (debug cepat), pakai `port-forward` (topik 04):
```bash
kubectl port-forward svc/app 9090:80    # Mac:9090 → Service:80
```

## 4. Kubeconfig — Banyak Cluster, Satu File

`kubectl` butuh kubeconfig untuk tahu cluster mana yang dituju. k3d bisa membuat/menggabung kubeconfig.

```bash
k3d kubeconfig get lab                  # dapat kubeconfig cluster lab
k3d kubeconfig merge lab --switch       # gabung + jadikan context aktif
kubectl config get-contexts
kubectl config use-context k3d-lab     # ganti context manual
kubectl config current-context

# Hapus context cluster yang sudah di-delete (bersih-bersih kubeconfig)
k3d cluster delete lab
kubectl config delete-context k3d-lab 2>/dev/null
```

**Penting nanti:** saat Modul 2.2 Anda punya cluster **k3s di VM** juga. kubeconfig akan punya 2 context: `k3d-lab` (fast) & `k3s-prod` (production). `kubectl config use-context` adalah cara berpindah. **Jangan salah `kubectl delete` di context production!**

## 5. Lifecycle Cluster

```bash
k3d cluster list                        # daftar cluster
k3d cluster stop lab                    # hentikan container (state tetap)
k3d cluster start lab                   # nyalakan lagi
k3d cluster delete lab                  # hapus total
```

`stop` vs `delete`:
- **stop**: container k3s berhenti, etcd/state **tetap** di volume; `start` → Pod kembali. Mac RAM dilepas.
- **delete**: hapus container + state; cluster hilang total. Pakai untuk bersih-bersih.

## 6. Multi-Cluster Workflow (Lab vs Coba-coba)

Pisahkan cluster "tahan lama" (untuk lab berhari-hari) dari "sekali coba":

```bash
# Cluster "sandbox" — untuk coba manifest, break, delete ulang
k3d cluster create sandbox --port 8080:80@loadbalancer
kubectl config use-context k3d-sandbox
# ... coba, break ...
k3d cluster delete sandbox

# Cluster "lab" — untuk lab berhari-hari (jangan di-delete sampai selesai)
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
```

**Aturan:** hanya satu cluster "besar" jalan pada satu waktu agar Mac tidak lag. `k3d cluster list` sebelum buat baru; `delete` yang tidak terpakai.

## 7. Image di k3d — Registry & Pull Secret

k3d (k3s) pakai containerd. Untuk image di **GitLab private registry**, butuh `imagePullSecret` (topik 02 LAB-01).

```bash
# (kalau repo private)
kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.com \
  --docker-username=<gitlab-user> \
  --docker-password=<PAT-read_registry>

# Atau import image lokal ke cluster (tanpa registry — untuk dev cepat):
k3d image import myapp:v1 -c lab       # load image Mac ke k3d node
```

`k3d image import` berguna untuk tes cepat tanpa push registry. Tapi untuk GitOps/production (Fase 6) **wajib** lewat registry — image harus ada di tempat yang dapat di-pull ulang.

## 8. k3d vs k3s — Keputusan Final

| Aspek | k3d (container) | k3s di VM OrbStack |
|---|---|---|
| Tujuan | latihan harian, iterasi cepat, CI | simulasi production on-prem |
| Startup | detik | menit |
| Resource | sangat ringan | lebih berat (VM) |
| Static IP | tidak | ya (IP stabil) |
| MetalLB L2 | sulit (ARP di container) | bisa (jaringan nyata) |
| etcd backup (snapshot) | bisa, tapi kurang relevan | inti Modul 2.4 |
| Ansible target | tidak | ya (Fase 4) |

**Aturan bootcamp (diulang):**
- **Modul 2.1, 2.4 (debug)** → k3d (cepat).
- **Modul 2.2–2.3, Fase 4 (Ansible)** → k3s di VM OrbStack.
- **Fase 6 GitOps** → k3d "staging" + k3s "production" (capstone).

## Latihan Cepat (20 menit)

```bash
# 1. Buat & hapus cepat
k3d cluster delete demo 2>/dev/null
k3d cluster create demo --port 8080:80@loadbalancer
kubectl get nodes
k3d cluster delete demo

# 2. Multi-node + lifecycle
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
kubectl get nodes
k3d cluster stop lab && k3d cluster start lab
kubectl get nodes
k3d kubeconfig merge lab --switch

# 3. Cek context
kubectl config current-context
kubectl config get-contexts

# 4. Bersihkan
k3d cluster delete lab
docker system prune -f
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Buat cluster | `k3d cluster create <name> --servers N --agents M --port H:C@loadbalancer` |
| Hapus | `k3d cluster delete <name>` |
| Stop/start | `k3d cluster stop/start <name>` |
| Set context | `k3d kubeconfig merge <name> --switch` |
| Import image lokal | `k3d image import <img> -c <name>` |
| Daftar cluster | `k3d cluster list` |
| Fast vs prod lane | k3d=cepat, k3s di VM=production-like |

## Cek Pemahaman

1. Beda `k3d cluster stop` vs `delete` (apa yang terjadi pada state/Pod)?
2. Kenapa k3d tidak cocok untuk simulasi MetalLB L2? (aspek teknis)
3. Anda punya context `k3d-lab` & `k3s-prod`. Bagaimana berpindah & apa risiko salah context?
4. Kapan pakai `k3d image import` vs push ke GitLab registry?