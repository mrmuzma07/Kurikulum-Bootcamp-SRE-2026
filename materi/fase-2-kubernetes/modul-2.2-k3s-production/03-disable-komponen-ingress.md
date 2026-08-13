# 03 — Disable Komponen Bawaan & Ingress

> k3s membawa Traefik & ServiceLB otomatis. Untuk on-prem nyata, sering kita matikan keduanya — agar MetalLB bisa mengambil alih, atau agar bisa pakai ingress controller sendiri.

## Tujuan
- Bisa menonaktifkan komponen bawaan k3s (`--disable traefik`, `--disable servicelb`, dll.)
- Paham apa itu servicelb (klipper) & kenapa disable sebelum MetalLB
- Bisa menjelaskan Traefik (bawaan) vs ingress controller alternatif & trade-off-nya
- Bisa install ingress controller alternatif (nginx-ingress) setelah Traefik disable
- Bisa memverifikasi komponen yang disable benar-benar hilang

## 1. Komponen Bawaan k3s

k3s membawa beberapa komponen yang otomatis jalan setelah install:

| Komponen | Fungsi | Default |
|---|---|---|
| **CoreDNS** | DNS internal cluster (resolve Service) | aktif |
| **Traefik** | Ingress controller (HTTP/HTTPS routing) | aktif |
| **ServiceLB** (klipper) | memberi IP untuk `Service type=LoadBalancer` di on-prem (pake hostIP) | aktif |
| **metrics-server** | resource metrics (untuk `kubectl top`, HPA) | aktif |
| **local-path-provisioner** | auto-buat PV untuk PVC (storage) | aktif |

```bash
kubectl get pods -n kube-system
# lihat: coredns, traefik, svclb-*, metrics-server, local-path-provisioner
```

Sebagian berguna (CoreDNS, metrics-server), sebagian bisa diganti/disable (Traefik, ServiceLB) tergantung kebutuhan on-prem.

## 2. Kenapa Disable? — Tiga Kasus

**Kasus A — MetalLB (Modul 2.3):** ServiceLB (klipper) "mencuri" alamat LoadBalancer dengan memakai hostIP. MetalLB butuh mengontrol sendiri range IP via ARP/BGP. **Keduanya bentrok** → disable servicelb dulu, baru pasang MetalLB. Ini alasan utama di bootcamp.

**Kasus B — Ingress controller pilihan:** Traefik bawaan berbeda konfigurasi dari nginx-ingress yang mungkin tim sudah pakai. Untuk konsistensi/standar, disable Traefik, pasang nginx-ingress sendiri.

**Kasus C — Minimalis (edge):** di perangkat kecil, matikan yang tidak dipakai (mis. metrics-server jika tidak pakai HPA) untuk hemat resource.

## 3. Cara Disable — Flag Install

Disable saat install via `--disable <nama>` (bisa berulang). Nama komponen sesuai [manifest bawaan k3s](https://github.com/k3s-io/k3s/tree/master/manifests):

```bash
# Contoh: disable Traefik + ServiceLB (siap MetalLB)
curl -sfL https://get.k3s.io | sh -s - server \
  --disable traefik \
  --disable servicelb \
  --cluster-init
```

Daftar komponen yang bisa di-disable:

| Flag | Yang dihilangkan |
|---|---|
| `--disable traefik` | Traefik Ingress controller |
| `--disable servicelb` | ServiceLB (klipber) — LoadBalancer on-prem bawaan |
| `--disable metrics-server` | metrics-server (`kubectl top` jadi tidak jalan) |
| `--disable local-storage` | local-path-provisioner (PVC otomatis hilang) |
| `--disable coredns` | CoreDNS (DNS internal — jarang didisable) |

### Disable di Cluster yang Sudah Ada (Tanpa Reinstall)

Untuk cluster single-node yang sudah terinstall, edit konfigurasi service k3s:

```bash
# Di VM server:
sudo systemctl edit k3s   # atau edit file env
# Tambahkan di /etc/systemd/system/k3s.service.env atau override:
#   K3S_DISABLE_TRAEFIK=true
#   K3S_DISABLE_SERVICELB=true
# Atau edit ExecStart di /etc/systemd/system/k3s.service:
sudo sed -i 's|server \\|server --disable traefik --disable servicelb \\|' /etc/systemd/system/k3s.service 2>/dev/null
# (cara paling bersih: uninstall + reinstall dengan flag — lihat troubleshooting)
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

> **Cara paling dapat diandalkan untuk lab:** `k3s-uninstall.sh` lalu reinstall dengan `--disable`. Untuk multi-node, terapkan flag saat install tiap server/agent. Fase 4 Ansible akan memastikan flag ini konsisten lintas node.

## 4. Verifikasi Disable

```bash
kubectl get pods -n kube-system
# Traefik tidak ada (kalau --disable traefik)
# svclb-* tidak ada (kalau --disable servicelb)

kubectl get svc -n kube-system | grep -E "traefik|svclb"   # harus kosong
kubectl get deploy -A | grep traefik                        # harus kosong

# Tes: buat Service type=LoadBalancer (kalau servicelb disable → status Pending)
kubectl create deployment web --image=nginx:alpine -n default
kubectl expose deployment web --port=80 --type=LoadBalancer
kubectl get svc web
# EXTERNAL-IP: <pending>  ← benar! servicelb hilang, MetalLB belum dipasang
# (ini "siap" untuk Modul 2.3)
```

## 5. Traefik (Bawaan) vs Ingress Alternatif

| Aspek | Traefik (bawaan k3s) | nginx-ingress-controller | HAProxy Ingress |
|---|---|---|---|
| Konfigurasi | IngressRoute CRD + annotation | Ingress standar + ConfigMap | Ingress + CRD |
| Auto-HTTPS | ya (Let's Encrypt bawaan) | perlu cert-manager + addon | perlu addon |
| Familiaritas tim | mungkin kurang umum | sangat umum (banyak tutorial) | umum di enterprise |
| Resource | ringan | sedang | sedang |
| Status di k3s | bawaan, tinggal pakai | harus install manual | harus install manual |

**Pilihan bootcamp:**
- **Fase 2–3:** pakai **Traefik bawaan** (k3d & k3s) — tidak perlu setup, fokus ke konsep K8s. (Modul 2.1 LAB-02 sudah pakai.)
- **Saat disable Traefik:** install nginx-ingress (contoh di bawah) — latih "memilih komponen sendiri" (skill Fase 4–5).

## 6. Install nginx-ingress (Setelah Traefik Disable)

```bash
# Pastikan Traefik sudah disable (--disable traefik saat install)
# Install nginx-ingress (Helm di Fase 5; sekarang via manifest resmi)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx -w    # tunggu Running
kubectl get svc -n ingress-nginx

# Untuk ekspos nginx-ingress di on-prem (tanpa cloud LB), pakai Service type=NodePort
# atau MetalLB (Modul 2.3). Sebelum MetalLB, akses via NodePort:
kubectl patch svc -n ingress-nginx ingress-nginx-controller -p '{"spec":{"type":"NodePort"}}' 2>/dev/null
kubectl get svc -n ingress-nginx ingress-nginx-controller
# nodePort (30000-32767) → akses dari Mac via VMIP:nodePort
```

Buat Ingress yang memakai nginx-ingress (beda class dari Traefik):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    kubernetes.io/ingress.class: nginx       # atau spec.ingressClassName: nginx
spec:
  rules:
  - host: web.k3s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: web, port: {number: 80}}
```

```bash
echo "127.0.0.1 web.k3s.local" | sudo tee -a /etc/hosts   # atau VMIP di /etc/hosts
# Akses via NodePort (sebelum MetalLB):
NODEPORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
curl -H "Host: web.k3s.local" http://$(orb ip k3s-cp1):$NODEPORT/
```

> **Sering lebih sederhana** untuk Fase 2: tetap pakai Traefik bawaan, fokus ke objek K8s. Pilihan ingress alternatif ini adalah latihan "saya bisa ganti komponen" — berguna saat produksi punya standar nginx-ingress.

## 7. IngressClass (Kubernetes ≥1.18)

Saat ada >1 ingress controller (mis. Traefik + nginx), K8s memakai **IngressClass** untuk memilih mana yang menangani Ingress mana. Tiap controller punya class sendiri (`traefik`, `nginx`, `haproxy`).

```bash
kubectl get ingressclass
# NAME    CONTROLLER                      PARAMETERS
# traefik traefik.io/ingress-controller    <none>
# nginx   k8s.io/ingress-nginx             <none>

# Ingress tunjuk class:
# spec: {ingressClassName: nginx}
# atau annotation: kubernetes.io/ingress.class: nginx (lama, deprecated)
```

## 8. Ringkasan Kapan Disable Apa

| Skenario | Disable | Pasang |
|---|---|---|
| Fase 2 latihan (fokus konsep) | (tidak perlu) | pakai Traefik bawaan |
| Siap MetalLB (Modul 2.3) | `servicelb` | MetalLB (Service LB) |
| Ganti ingress ke nginx | `traefik` | nginx-ingress |
| Edge minimalis | `metrics-server`, `traefik`, `servicelb` | sesuai kebutuhan |

## Latihan Cepat (20 menit)

```bash
# 1. Install k3s fresh dengan disable (di k3s-cp1 atau VM baru)
ssh k3s-cp1 'sudo /usr/local/bin/k3s-uninstall.sh 2>/dev/null; curl -sfL https://get.k3s.io | sh -s - server --disable traefik --disable servicelb --cluster-init'

# 2. Verifikasi komponen hilang
ssh k3s-cp1 'sudo k3s kubectl get pods -A | grep -E "traefik|svclb"'   # kosong

# 3. Tes Service LB pending (servicelb hilang, MetalLB belum)
ssh k3s-cp1 'sudo k3s kubectl create deployment web --image=nginx:alpine; sudo k3s kubectl expose deployment web --port=80 --type=LoadBalancer; sudo k3s kubectl get svc web'
# EXTERNAL-IP: <pending> ← harapan (MetalLB datang Modul 2.3)

# 4. (Opsional) install nginx-ingress, deploy Ingress, akses via NodePort
```

## Ringkasan

| Mau... | Caranya |
|---|---|
| Disable saat install | `curl ... \| sh -s - server --disable traefik --disable servicelb` |
| Disable pasca-install | edit k3s.service / uninstall + reinstall (paling bersih) |
| Verifikasi hilang | `kubectl get pods -A \| grep traefik` (kosong) |
| Kenapa disable servicelb | bentrok dengan MetalLB (LB on-prem) |
| Traefik vs nginx | Traefik=bawaan/auto-HTTPS; nginx=umum/standar tim |
| Pilih ingress multi-controller | IngressClass (`spec.ingressClassName`) |

## Cek Pemahaman

1. Apa itu ServiceLB (klipper) & kenapa harus disable sebelum pasang MetalLB?
2. Sebut 2 alasan seseorang menonaktifkan Traefik bawaan k3s.
3. Setelah `--disable servicelb`, Service `type=LoadBalancer` jadi apa statusnya? Kenapa?
4. Saat ada Traefik + nginx-ingress, bagaimana K8s tahu Ingress mana ditangani siapa?