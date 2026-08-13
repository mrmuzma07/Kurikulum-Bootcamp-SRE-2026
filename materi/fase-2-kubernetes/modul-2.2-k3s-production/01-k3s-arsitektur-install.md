# 01 — k3s: Arsitektur & Install di VM OrbStack

> k3s = Kubernetes ringan untuk on-prem; satu biner, embedded etcd, install via `curl | sh`. Sekarang kita pasang di VM sungguhan — bukan container.

## Tujuan
- Bisa menjelaskan apa itu k3s & kenapa cocok untuk on-prem
- Bisa install k3s single-node di VM OrbStack (Ubuntu)
- Bisa ambil kubeconfig & menjalankan `kubectl` dari Mac
- Paham perbedaan install k3s (VM, systemd) vs k3d (container)

## 1. Apa Itu k3s?

**k3s** = distribusi Kubernetes ringan yang dibundel jadi **satu biner** (~70 MB). Semua komponen control plane (apiserver, etcd/SQLite, scheduler, controller-manager) + kubelet + kube-proxy + containerd digabung dalam satu proses. Dirancang Rancher untuk edge/IoT/on-prem — hemat resource, install mudah, dependensi minimal.

**Kenapa k3s untuk bootcamp on-prem:**
- **Ringan**: 512 MB RAM minimal (control plane); cluster 3 node muat di Mac.
- **Satu biner**: `curl -sfL https://get.k3s.io | sh` → jadi cluster. Tanpa paket besar.
- **Embedded etcd**: HA dengan etcd bawaan (3 server), tanpa setup etcd terpisah.
- **Bisa disable komponen**: Traefik, servicelb, metrics-server bisa dimatikan (topik 03) — penting untuk MetalLB.
- **Production-like**: install di VM dengan systemd, static IP, persis server sungguhan.

```
 ┌─ k3d (Modul 2.1) ──────┐   ┌─ k3s di VM (modul ini) ──────┐
 │ k3s jalan SEBAGAI      │   │ k3s jalan DI ATAS            │
 │ container Docker       │   │ VM Linux (systemd, kernel)   │
 │ cepat, ringan, delete  │   │ "server sungguhan"           │
 │ → latihan harian       │   │ → simulasi production        │
 └────────────────────────┘   └──────────────────────────────┘
```

## 2. Siapkan VM OrbStack

Pakai Machine OrbStack (Modul 1.2 topik 01). Untuk single-node, satu VM cukup:

```bash
# Buat VM (kalau belum; Ubuntu 24.04, arm64 native)
orb create -a ubuntu:24.04 k3s-cp1 2>/dev/null || orb start k3s-cp1
orb ls
orb ip k3s-cp1                              # catat IP (stabil)
```

Setup SSH standar (untuk Ansible nanti — Fase 4):
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@$(orb ip k3s-cp1)
# buat alias
cat >> ~/.ssh/config <<EOF
Host k3s-cp1
    HostName $(orb ip k3s-cp1)
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
EOF
ssh k3s-cp1 'hostname; uname -m'
```

Update OS:
```bash
ssh k3s-cp1 'sudo apt update && sudo apt -y upgrade'
```

## 3. Install k3s Single-Node

Install di dalam VM (server = control plane + worker gabungan):
```bash
ssh k3s-cp1
# di dalam VM:
curl -sfL https://get.k3s.io | sh -
# ini: download biner k3s, jalankan sebagai systemd service (k3s.service)
# Tunggu "k3s is now running" / service aktif

sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes                  # di dalam VM, kubectl via k3s wrapper
```

Default k3s:
- kubeconfig: `/etc/rancher/k3s/k3s.yaml` (milik root)
- `kubectl` bawaan: `k3s kubectl` (wrapper, tidak perlu install kubectl terpisah)
- API server: `https://127.0.0.1:6443`
- Node internal IP: IP VM (`orb ip`)

```bash
sudo cat /etc/rancher/k3s/k3s.yaml          # ini kubeconfig cluster
sudo k3s kubectl get pods -A                 # komponen bawaan: CoreDNS, Traefik, servicelb, metrics-server
```

## 4. `kubectl` dari Mac (Ambil Kubeconfig)

Inilah perbedaan besar dari k3d: kubeconfig harus **disalin manual** dari VM ke Mac, dengan IP server diganti agar API server reachable dari luar VM.

```bash
# Di Mac:
VMIP=$(orb ip k3s-cp1)

# Salin kubeconfig dari VM (butuh sudo karena file root)
scp ubuntu@$VMIP:/etc/rancher/k3s/k3s.yaml ./k3s-cp1.yaml

# Edit: ganti 127.0.0.1 → IP VM (agar Mac bisa reach API server)
sed -i '' "s/127.0.0.1/$VMIP/g" ./k3s-cp1.yaml

# Pasang ke kubeconfig Mac dengan context unik (hindari bentrok k3d)
KUBECONFIG=~/.kube/config:./k3s-cp1.yaml kubectl config view --flatten > ~/.kube/config.merged
mv ~/.kube/config.merged ~/.kube/config
kubectl config rename-context default k3s-cp1 2>/dev/null || true
kubectl config use-context k3s-cp1
kubectl get nodes
```

**Sekarang:** dari Mac, `kubectl` bisa berpindah antara `k3d-lab` (fast lane) & `k3s-cp1` (production lane):
```bash
kubectl config get-contexts
kubectl config use-context k3d-lab       # k3d
kubectl config use-context k3s-cp1      # k3s di VM
```

> **Alternatif lebih ringkas:** `export KUBECONFIG=~/.kube/config:./k3s-cp1.yaml` lalu `kubectl config use-context ...`. Pilih yang konsisten untuk Anda.

## 5. Beda Install k3s vs k3d

| Aspek | k3d (Modul 2.1) | k3s di VM (modul ini) |
|---|---|---|
| Install | `k3d cluster create` | `curl -sfL ... \| sh -` di VM |
| Jalan sebagai | container Docker | systemd service (`k3s.service`) |
| Kubeconfig | auto-merge (`k3d kubeconfig merge`) | salin manual, ganti IP |
| IP node | IP container (berubah) | IP VM stabil (`orb ip`) |
| Lifecycle | `k3d cluster delete` | `sudo /usr/local/bin/k3s-uninstall.sh` |
| MetalLB L2 | sulit | bisa (topik 04 / Modul 2.3) |
| Ansible target | tidak | ya (Fase 4) |
| Restart VM → cluster? | tidak relevan (container) | ya, systemd auto-start k3s |

## 6. Deploy App & Verifikasi

```bash
# Dari Mac (context k3s-cp1)
kubectl create ns demo
kubectl config set-context --current --namespace=demo
kubectl create deployment web --image=nginx:alpine --replicas=2
kubectl get pod -o wide                  # Pod jalan di VM k3s-cp1
kubectl expose deployment web --port=80
kubectl port-forward svc/web 9090:80    # akses dari Mac:9090
# (buka http://localhost:9090)
```

**Catatan:** tanpa `--port H:C@loadbalancer` (k3d), akses dari Mac ke k3s butuh **port-forward** atau (nanti) Ingress/MetalLB. Topik 04 & Modul 2.3 menyelesaikan akses external yang "production-like".

## 7. Uninstall k3s

```bash
# Di VM:
sudo /usr/local/bin/k3s-uninstall.sh     # single-node
sudo systemctl status k3s                 # hilang
# Mac: hapus context
kubectl config delete-context k3s-cp1 2>/dev/null
```

Bersih-bersih VM:
```bash
orb stop k3s-cp1 && orb rm k3s-cp1        # kalau tidak dipakai lagi
```

## Latihan Cepat (20 menit)

```bash
# 1. VM & SSH
orb create -a ubuntu:24.04 k3s-cp1 2>/dev/null || orb start k3s-cp1
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@$(orb ip k3s-cp1)

# 2. Install
ssh k3s-cp1 'curl -sfL https://get.k3s.io | sh -; sudo k3s kubectl get nodes'

# 3. Kubeconfig ke Mac
scp ubuntu@$(orb ip k3s-cp1):/etc/rancher/k3s/k3s.yaml /tmp/k3s.yaml
sudo sed -i '' "s/127.0.0.1/$(orb ip k3s-cp1)/" /tmp/k3s.yaml   # atau edit manual; scp butuh sudo di VM
# (kalau scp gagal permission: ssh k3s-cp1 'sudo cat /etc/rancher/k3s/k3s.yaml' > /tmp/k3s.yaml; lalu sed)
KUBECONFIG=~/.kube/config:/tmp/k3s.yaml kubectl config view --flatten > ~/.kube/config
kubectl config use-context default
kubectl get nodes
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Install k3s (di VM) | `curl -sfL https://get.k3s.io \| sh -` |
| kubectl di VM | `sudo k3s kubectl ...` |
| Ambil kubeconfig | `scp ... /etc/rancher/k3s/k3s.yaml`, ganti `127.0.0.1`→IP VM |
| Uninstall | `sudo /usr/local/bin/k3s-uninstall.sh` |
| Beda k3d | k3d=container/auto-kubeconfig; k3s=VM/systemd/salin manual |

## Cek Pemahaman

1. Kenapa k3s cocok untuk on-prem/IoT dibanding K8s "penuh"? (sebut 2)
2. Setelah `curl | sh`, k3s jalan sebagai apa? (proses/service apa, siapa yang start ulang saat VM reboot?)
3. Kenapa kubeconfig dari VM harus diganti `127.0.0.1` → IP VM sebelum dipakai dari Mac?
4. Sebut 2 perbedaan teknis install k3d vs k3s (dari sudut "rumahnya" — container vs VM).