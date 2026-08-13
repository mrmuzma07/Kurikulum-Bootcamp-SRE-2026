# 04 — Konfigurasi & Integrasi MetalLB dengan k3s

> Panduan dari cluster k3s yang sudah siap sampai Service `type=LoadBalancer` memperoleh VIP yang dapat diakses dari Mac.

## Tujuan

- Memastikan k3s tidak menjalankan dua implementasi LoadBalancer
- Menentukan range VIP yang aman untuk mode L2
- Memasang MetalLB dengan manifest yang dipin ke versi tertentu
- Membuat `IPAddressPool` dan `L2Advertisement`
- Memverifikasi controller, speaker, assignment IP, ARP, dan traffic
- Mengetahui cleanup dan batasan lab OrbStack

## 1. Prasyarat dan Context Safety

Jalankan semua command dari terminal Mac, kecuali command yang diberi label VM. Sebelum operasi:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
```

Pastikan context menunjuk ke k3s VM dari Modul 2.2, **bukan k3d**. Untuk operasi yang dapat mengubah cluster, biasakan:

```bash
kubectl config get-contexts
kubectl config current-context
```

Checklist:

- [ ] Minimal satu node k3s `Ready`.
- [ ] Kubeconfig dapat mengakses API server.
- [ ] `servicelb`/klipper disabled saat instalasi k3s.
- [ ] Anda mengetahui IP dan subnet Machine OrbStack.
- [ ] Range VIP disediakan khusus untuk MetalLB.
- [ ] Range tidak overlap DHCP, IP node, gateway, atau service lain.
- [ ] Firewall dan network lab mengizinkan traffic yang diperlukan.

## 2. Pastikan `servicelb` Disabled

Modul 2.2 memasang k3s dengan flag:

```bash
--disable servicelb
```

Verifikasi proses dan Pod:

```bash
kubectl get pods -A | grep svclb || echo "svclb tidak ditemukan"
kubectl get daemonset -A | grep svclb || echo "daemonset svclb tidak ditemukan"
```

Pada host VM, bila perlu periksa argumen service k3s:

```bash
# Jalankan di VM k3s, bukan di Mac
sudo systemctl cat k3s | grep -E -- '--disable|ExecStart'
```

Jika `svclb-*` masih ada, jangan memasang MetalLB secara membabi buta. Perbaiki instalasi k3s atau buat cluster lab baru dengan `--disable servicelb`. Dua mekanisme LoadBalancer dapat menghasilkan perilaku yang sulit diprediksi.

## 3. Menentukan Subnet dan Range VIP

Dapatkan IP tiap Machine:

```bash
orb list
orb ip k3s-cp1
orb ip k3s-cp2
orb ip k3s-w1
```

Di VM, lihat interface dan route:

```bash
orb shell k3s-cp1 -- ip -br address
orb shell k3s-cp1 -- ip route
```

Contoh **ilustrasi saja**:

```text
Node cp1     192.168.97.10/24
Node cp2     192.168.97.11/24
Gateway      192.168.97.1
DHCP         192.168.97.100–192.168.97.180
MetalLB VIP  192.168.97.200–192.168.97.220
```

Jangan menyalin `192.168.97.0/24` tanpa memeriksa network Anda. Gunakan range yang:

1. berada di subnet L2 yang sama dengan node/client;
2. tidak termasuk DHCP pool;
3. tidak digunakan host, gateway, node, atau perangkat lain;
4. cukup untuk jumlah Service LoadBalancer;
5. dicatat/di-reserve di dokumentasi network.

Tiga sumber kebenaran sebelum memilih pool:

```bash
# Mac: route ke IP Machine
route -n get "$(orb ip k3s-cp1)"

# VM: interface + route
orb shell k3s-cp1 -- ip addr
orb shell k3s-cp1 -- ip route

# Uji apakah kandidat sudah digunakan (ganti interface/range)
sudo arping -I <interface> -c 3 <candidate-ip>
```

Jika kandidat menjawab ARP, anggap IP sedang digunakan sampai terbukti sebaliknya. Jangan mengambil IP secara acak.

## 4. Pasang MetalLB dengan Versi Terpin

Pilih versi yang sudah diuji oleh materi/lab. Contoh berikut memakai `v0.14.9`; ganti hanya setelah membaca release notes dan memvalidasi CRD:

```bash
export METALLB_VERSION=v0.14.9
kubectl apply -f "https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml"
```

Untuk lingkungan dengan akses internet terbatas, unduh manifest melalui mirror resmi yang disetujui dan review file sebelum apply. Jangan menjalankan manifest remote yang tidak dikenal.

Tunggu komponen siap:

```bash
kubectl get ns metallb-system
kubectl wait --namespace metallb-system \
  --for=condition=available deployment/controller \
  --timeout=180s
kubectl rollout status daemonset/speaker \
  -n metallb-system --timeout=180s
kubectl get pods -n metallb-system -o wide
```

Expected minimal:

```text
controller-...   1/1 Running
speaker-...      1/1 Running   # satu per node yang eligible
```

Jika CRD belum tersedia, tunggu beberapa detik lalu cek:

```bash
kubectl get crd | grep metallb
kubectl api-resources | grep -i metallb
```

## 5. Buat IPAddressPool dan L2Advertisement

Simpan manifest di direktori lab, misalnya `m2.3/lab/metallb-l2.yaml`. Ganti range sesuai hasil inventarisasi network:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: orbstack-l2-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.97.200-192.168.97.220
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: orbstack-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - orbstack-l2-pool
```

Apply dan verifikasi:

```bash
kubectl apply -f metallb-l2.yaml
kubectl get ipaddresspool,l2advertisement -n metallb-system
kubectl describe ipaddresspool orbstack-l2-pool -n metallb-system
```

`IPAddressPool` hanya menyediakan alamat untuk dialokasikan. `L2Advertisement` menghubungkan pool dengan mekanisme ARP/NDP. Keduanya diperlukan untuk alur L2 yang lengkap.

## 6. Uji dengan Service LoadBalancer

Gunakan Deployment yang sudah ada atau buat app kecil:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
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
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: demo
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl create namespace demo --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f web-loadbalancer.yaml
kubectl get svc web -n demo -w
```

Expected setelah controller memproses:

```text
NAME   TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)
web    LoadBalancer   10.43.x.x    192.168.97.200   80:3xxxx/TCP
```

`EXTERNAL-IP` adalah VIP MetalLB, bukan IP Pod dan bukan ClusterIP.

## 7. Verifikasi Allocation → ARP → HTTP

```bash
VIP=$(kubectl get svc web -n demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
printf 'VIP=%s\n' "$VIP"

# Allocation dan endpoints
kubectl describe svc web -n demo
kubectl get endpointslice -n demo -l kubernetes.io/service-name=web

# Speaker
kubectl get pods -n metallb-system -o wide
kubectl logs -n metallb-system -l component=speaker --tail=80

# ARP dari Mac yang berada pada network reachable
IFACE=$(route get default | awk '/interface:/{print $2}')
sudo arping -I "$IFACE" -c 3 "$VIP"
arp -a | grep "$VIP" || true

# HTTP
curl --fail --connect-timeout 5 "http://${VIP}/"
```

Urutan diagnosis:

```text
EXTERNAL-IP <pending> → controller/pool/CRD
VIP ada, arping timeout → speaker, L2Advertisement, subnet, firewall
arping berhasil, curl timeout → Service, endpoints, kube-proxy, firewall TCP
curl berhasil → jalur dasar selesai; lanjut Ingress/DNS/TLS bila diperlukan
```

## 8. Hubungkan ke Ingress

MetalLB memberi endpoint Layer 3/4. Untuk HTTP host/path, expose ingress controller sebagai LoadBalancer:

```bash
kubectl get svc -A | grep -E 'traefik|ingress|LoadBalancer'
```

Jika Traefik k3s sengaja di-disable pada Modul 2.2, install ingress controller pilihan tim terlebih dahulu. Jangan menganggap MetalLB otomatis membuat route Ingress.

Jalur production-like:

```text
Mac / client
    │ DNS atau /etc/hosts → VIP MetalLB
    ▼
Service LoadBalancer (Traefik/nginx)
    │ host/path routing
    ▼
Service ClusterIP
    ▼
Pod
```

## 9. Troubleshooting Awal

### Controller atau speaker tidak Running

```bash
kubectl describe pod -n metallb-system <pod>
kubectl get events -n metallb-system --sort-by=.lastTimestamp
kubectl logs -n metallb-system deploy/controller --tail=100
```

Periksa image pull, resource VM, CRD, dan status node.

### `EXTERNAL-IP` tetap `<pending>`

```bash
kubectl describe svc web -n demo
kubectl get ipaddresspool,l2advertisement -n metallb-system -o yaml
kubectl logs -n metallb-system deploy/controller --tail=100
```

Kemungkinan: pool tidak ada, pool habis, `autoAssign: false` tanpa annotation, atau API/CRD version tidak cocok.

### IP terisi tetapi tidak dapat diakses

```bash
kubectl get pods -n metallb-system -o wide
kubectl logs -n metallb-system -l component=speaker --tail=100
sudo arping -I "$IFACE" -c 3 "$VIP"
kubectl get endpointslice -n demo -l kubernetes.io/service-name=web
```

Periksa subnet VIP, IP conflict, route OrbStack, firewall, dan apakah client memang berada pada L2 yang sama. Jika memakai kube-proxy IPVS, review `strictARP` sesuai dokumentasi k3s/MetalLB; k3s default biasanya iptables sehingga jangan mengubah setting tanpa bukti.

### IP conflict

Hentikan pengujian dan tarik VIP dari pool. Cari perangkat yang menjawab:

```bash
sudo arping -I "$IFACE" -c 5 "$VIP"
arp -a | grep "$VIP" || true
```

Jangan mengatasi conflict dengan mengubah MAC secara manual. Perbaiki reservation/range network.

## 10. Cleanup Lab

```bash
kubectl delete -f web-loadbalancer.yaml
kubectl delete -f metallb-l2.yaml
kubectl delete namespace demo --ignore-not-found
```

Untuk menghapus MetalLB, gunakan manifest **versi yang sama** dengan instalasi:

```bash
kubectl delete -f "https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml"
```

Pada lab berulang, pastikan tidak ada Service lama yang masih memegang VIP sebelum mengubah pool. Setelah cleanup, validasi kembali:

```bash
kubectl get svc -A | grep LoadBalancer || true
kubectl get pods -n metallb-system 2>/dev/null || true
```

## Acceptance Criteria

- [ ] Context `kubectl` benar-benar menunjuk k3s VM.
- [ ] Tidak ada `svclb-*` karena `servicelb` disabled.
- [ ] Controller dan satu Speaker per node `Running`.
- [ ] `IPAddressPool` dan `L2Advertisement` `Configured`/terbaca.
- [ ] Service `web` mendapat VIP dari range yang sudah di-reserve.
- [ ] `arping` menerima reply dari VIP.
- [ ] `curl http://<VIP>/` berhasil dari Mac.
- [ ] Catatan membedakan kegagalan allocation, ARP, TCP, dan HTTP.
- [ ] Manifest dan bukti lab disimpan melalui Git/MR tanpa secret.

## Kaitan dengan Modul Berikutnya

- LAB-01 mempraktikkan seluruh alur install dan expose.
- LAB-02 menggunakan pola **allocation → speaker → ARP → TCP → HTTP** untuk troubleshooting.
- Modul 2.4 akan memperluas diagnosis ke node failure, `Pending`, `CrashLoopBackOff`, dan snapshot etcd.
- Fase 5/6 akan memindahkan manifest manual ini ke Helm dan GitOps; range VIP tetap harus menjadi keputusan network yang direview.
