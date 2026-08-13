# LAB-01 — MetalLB L2: Expose Service dari k3s VM

> **Skenario:** Anda menjalankan cluster k3s on-prem di VM OrbStack. `servicelb` sengaja disabled agar peserta memasang provider LoadBalancer sendiri. Pasang MetalLB mode L2, alokasikan VIP dari pool yang aman, lalu akses aplikasi dari Mac.

## Tujuan

- Memasang MetalLB pada k3s cluster Modul 2.2.
- Membuat `IPAddressPool` dan `L2Advertisement`.
- Mendapatkan `EXTERNAL-IP` untuk Service `type=LoadBalancer`.
- Membuktikan VIP diiklankan melalui ARP.
- Menggambar jalur Mac → VIP → node → Service → Pod.

## Prasyarat

- Modul 2.2 selesai; gunakan k3s VM OrbStack, **bukan k3d**.
- Minimal satu node k3s `Ready`; HA 3 server + worker dianjurkan.
- k3s dibuat dengan `--disable servicelb`.
- `kubectl` dari Mac sudah memakai context k3s yang benar.
- `arping` tersedia (`brew install arping` bila belum ada).
- Anda sudah menentukan subnet dan range VIP yang di-reserve. Contoh di bawah memakai `192.168.97.200-192.168.97.220`; **ganti dengan network Anda**.
- RAM OrbStack cukup untuk cluster dan MetalLB.

## Aturan Keamanan Lab

1. Jalankan `kubectl config current-context` sebelum command yang mengubah cluster.
2. Jangan memakai `kubectl delete -A`.
3. Jangan mengambil IP dari DHCP pool atau IP perangkat lain.
4. Review manifest remote dan pin versi MetalLB; jangan pipe konten URL yang tidak dikenal ke shell.
5. Jangan commit kubeconfig, token k3s, PAT GitLab, atau Secret plain text.

## Arsitektur Lab

```text
Mac / client
   │  ARP untuk VIP 192.168.97.200
   ▼
MetalLB Speaker (leader pada salah satu node k3s)
   │  kube-proxy / Service routing
   ▼
Service web (LoadBalancer)
   │
   ▼
Pod web (2 replica)
```

## Bagian 1 — Verifikasi Context dan Network

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl cluster-info

kubectl get pods -A | grep svclb || echo "svclb tidak ditemukan — siap MetalLB"
orb list
orb ip k3s-cp1
```

Catat pada laporan lab:

```text
Context:
Node dan IP:
Subnet node:
Gateway:
DHCP range (jika diketahui):
Range VIP yang di-reserve:
```

Jika `svclb-*` masih ada, berhenti dan perbaiki cluster k3s. Jangan lanjut memasang dua controller LoadBalancer.

## Bagian 2 — Pasang MetalLB

Gunakan versi yang sudah dipilih oleh instruktur. Contoh:

```bash
export METALLB_VERSION=v0.14.9
kubectl apply -f "https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml"
```

Tunggu rollout:

```bash
kubectl wait --namespace metallb-system \
  --for=condition=available deployment/controller \
  --timeout=180s
kubectl rollout status daemonset/speaker \
  -n metallb-system --timeout=180s
kubectl get pods -n metallb-system -o wide
```

Catat jumlah Pod controller dan speaker. Expected: controller `Running` dan satu speaker `Running` pada setiap node yang eligible.

Jika gagal:

```bash
kubectl get events -n metallb-system --sort-by=.lastTimestamp
kubectl describe pod -n metallb-system <nama-pod>
```

## Bagian 3 — Tentukan VIP dan Terapkan Konfigurasi L2

Buat file lokal `metallb-l2.yaml`. Ganti range contoh sesuai hasil inventarisasi.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.97.200-192.168.97.220
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - lab-pool
```

Terapkan dan verifikasi:

```bash
kubectl apply -f metallb-l2.yaml
kubectl get ipaddresspool,l2advertisement -n metallb-system
kubectl describe ipaddresspool lab-pool -n metallb-system
```

**Catatan:** `IPAddressPool` mengatur alokasi; `L2Advertisement` mengatur advertisement. Keduanya bukan pengganti satu sama lain.

## Bagian 4 — Deploy App dan Service LoadBalancer

Buat file `web-loadbalancer.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-lab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: metallb-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports:
        - name: http
          containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 2
          periodSeconds: 5
        resources:
          requests:
            cpu: 10m
            memory: 16Mi
          limits:
            cpu: 100m
            memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: metallb-lab
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: http
```

```bash
kubectl apply -f web-loadbalancer.yaml
kubectl rollout status deployment/web -n metallb-lab --timeout=120s
kubectl get pods -n metallb-lab -o wide
kubectl get endpointslice -n metallb-lab -l kubernetes.io/service-name=web
kubectl get svc web -n metallb-lab -w
```

Hentikan `watch` dengan `Ctrl-C` setelah `EXTERNAL-IP` terisi. Ambil VIP:

```bash
VIP=$(kubectl get svc web -n metallb-lab -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
printf 'VIP=%s\n' "$VIP"
```

Expected: VIP berada pada range `lab-pool`, bukan `<pending>`, ClusterIP, atau IP Pod.

## Bagian 5 — Buktikan ARP dan Akses dari Mac

Pilih interface Mac yang memiliki route ke VM:

```bash
IFACE=$(route get default | awk '/interface:/{print $2}')
printf 'Interface=%s\n' "$IFACE"

sudo arping -I "$IFACE" -c 5 "$VIP"
arp -a | grep "$VIP" || true
curl --fail --connect-timeout 5 "http://${VIP}/"
```

Jika interface default bukan jalur ke subnet OrbStack, gunakan interface yang ditunjukkan oleh `route -n get <IP_VM>` dan ulangi. Simpan bukti output `arping` dan `curl` pada laporan.

Verifikasi sisi Kubernetes:

```bash
kubectl describe svc web -n metallb-lab
kubectl get pods -n metallb-system -o wide
kubectl logs -n metallb-system -l component=speaker --tail=80
```

Jelaskan:

- Speaker mana yang saat ini menjadi leader untuk VIP?
- MAC address apa yang dikembalikan `arping`?
- Apakah Pod web berada di node yang sama dengan speaker leader?
- Mengapa traffic tetap dapat mencapai Pod pada node lain saat `externalTrafficPolicy` default `Cluster`?

## Bagian 6 — Uji Self-Healing dan Failover

Hapus satu Pod web dan amati Deployment membuat pengganti:

```bash
kubectl get pods -n metallb-lab -w
# Pada terminal lain:
kubectl delete pod -n metallb-lab -l app=web --wait=false
```

Uji request berulang:

```bash
for i in $(seq 1 5); do curl --fail -s "http://${VIP}/" >/dev/null && echo "request $i OK"; sleep 1; done
```

Jika cluster memiliki lebih dari satu VM node dan Anda memahami risiko operasi, dokumentasikan speaker Pod dan simulasi failover pada **satu node lab saja**. Ikuti panduan instruktur OrbStack untuk menghentikan Machine; jangan mematikan node control plane production.

```bash
kubectl get pods -n metallb-system -l component=speaker -o wide
# Setelah node leader dihentikan oleh instruktur:
sleep 10
sudo arping -I "$IFACE" -c 3 "$VIP"
curl --fail --connect-timeout 5 "http://${VIP}/"
```

Catat bahwa:

- satu speaker aktif mengiklankan VIP pada satu waktu;
- failover membutuhkan convergence dan refresh neighbor cache;
- koneksi TCP yang sedang berjalan dapat reset;
- Service dan Pod tidak sama dengan leader advertisement.

## Bagian 7 — Diagram dan Laporan

Buat `m2.3/lab-01-report.md` berisi:

1. Context, daftar node/IP, subnet, dan range VIP.
2. Versi MetalLB dan output status Pod.
3. Manifest `IPAddressPool`/`L2Advertisement` yang sudah direview.
4. Output `kubectl get svc`, `arping`, `curl`.
5. Speaker leader dan hasil uji failover/self-healing.
6. Diagram jalur traffic.
7. Satu masalah yang ditemukan dan langkah diagnosisnya.

Contoh diagram:

```text
Mac (curl)
  │ ARP: siapa punya VIP?
  ▼
VIP 192.168.97.200 → MAC node speaker leader
  │
  ▼
kube-proxy → Service web:80
  │
  ▼
Pod web-1 / web-2
```

## Acceptance Criteria

- [ ] Context yang digunakan adalah k3s VM yang benar.
- [ ] `servicelb` tidak aktif.
- [ ] Controller dan speaker MetalLB `Running`.
- [ ] Pool dan advertisement berhasil dibuat tanpa overlap IP.
- [ ] Service web mendapat `EXTERNAL-IP` dari pool.
- [ ] Minimal tiga reply `arping` diterima.
- [ ] `curl http://<VIP>/` berhasil dari Mac.
- [ ] EndpointSlice menunjukkan dua endpoint Ready.
- [ ] Laporan menjelaskan assignment Controller vs advertisement Speaker.
- [ ] Diagram traffic dan bukti command tersimpan melalui MR.

## Troubleshooting Ringkas

| Gejala | Pemeriksaan pertama |
|---|---|
| `EXTERNAL-IP: <pending>` | `kubectl describe svc`, pool, controller logs |
| VIP ada, `arping` timeout | subnet, interface, L2Advertisement, speaker logs |
| ARP berhasil, HTTP gagal | endpoints, Service port/targetPort, kube-proxy, firewall |
| IP conflict | hentikan test, `arping`, review DHCP/static reservation |
| speaker tidak Ready | `describe pod`, events, resource dan node status |

## Kaitan

- LAB-02 memakai bukti dari lab ini untuk membedakan kegagalan allocation, ARP, TCP, dan aplikasi.
- Modul 2.4 akan menggunakan pola observasi yang sama untuk node failure, probes, dan etcd.
