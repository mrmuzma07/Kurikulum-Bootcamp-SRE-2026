# Latihan — Modul 2.2 k3s untuk Simulasi Production

Latihan ini memperkuat pemahaman **install k3s di VM**, **HA multi-node**, **disable komponen**, dan **topologi on-prem**. Kerjakan di terminal Mac (OrbStack VM).

> **Aturan:** kerjakan di terminal. Catat output penting di `m2.2/lab/log-latihan.md` di repo `sre-bootcamp`.

---

## Bagian 1 — Install k3s Single-Node

Kerjakan [LAB-01](../lab/LAB-01-k3s-single-node-vm.md) dan catat di laporan:
1. Setelah `curl | sh`, k3s jalan sebagai apa? (`systemctl status k3s` — service name, siapa yang restart saat VM reboot?)
2. Beda `sudo k3s kubectl` (di VM) vs `kubectl` (dari Mac, setelah kubeconfig disalin)?
3. Kenapa kubeconfig harus ganti `127.0.0.1` → IP VM sebelum dipakai dari Mac?

### 1.1 Verifikasi Dua Lane
```bash
kubectl config get-contexts
kubectl config use-context k3d-lab && kubectl get nodes      # k3d
kubectl config use-context k3s-cp1 && kubectl get nodes      # k3s
```
Catat: perbedaan node yang muncul di kedua context. Kenapa penting cek context sebelum operasi destruktif?

---

## Bagian 2 — Multi-Node HA

Kerjakan [LAB-02](../lab/LAB-02-k3s-multi-node-topologi.md) dan catat:
1. 3 server + 2 agent — berapa node `Ready`? Berapa `control-plane`?
2. Setelah `orb stop k3s-cp3` (1 server), apakah `kubectl get nodes` masih jalan? Kenapa (quorum)?
3. (Opsional) setelah stop 2 server, apa yang terjadi? Berapa lama "mati"? Pelajaran quorum.

### 2.1 Verifikasi etcd Quorum
```bash
kubectl get --raw='/readyz?verbose' 2>/dev/null | grep -i etcd
ssh k3s-cp1 'sudo k3s etcd-snapshot ls 2>/dev/null | head'
```
Catat: berapa etcd member? Apa yang terjadi pada quorum saat 1 server mati?

### 2.2 Token & Join
```bash
ssh k3s-cp1 'sudo cat /var/lib/rancher/k3s/server/node-token' | head -c 20; echo "..."
```
Catat: apa fungsi token? Apa risiko token bocor (di commit)? (pengantar Fase 6 secret handling)

---

## Bagian 3 — Disable Komponen

### 3.1 Verifikasi Disable
```bash
kubectl get pods -A | grep -E "traefik|svclb|metrics"
```
Catat: komponen apa yang ada? (traefik? svclb? metrics-server?)

### 3.2 Tes Service LB Pending
```bash
kubectl create deployment web --image=nginx:alpine -n default 2>/dev/null
kubectl expose deployment web --port=80 --type=LoadBalancer -n default 2>/dev/null
kubectl get svc web -n default
```
Catat: status `EXTERNAL-IP`? (pending bila servicelb disabled). Kenapa pending? (pengantar MetalLB Modul 2.3)

### 3.3 (Opsional) Reinstall dengan Disable
```bash
ssh k3s-cp1 'sudo /usr/local/bin/k3s-uninstall.sh 2>/dev/null'
ssh k3s-cp1 "curl -sfL https://get.k3s.io | sh -s - server --disable traefik --disable servicelb"
ssh k3s-cp1 'sudo k3s kubectl get pods -A | grep -E "traefik|svclb"'   # kosong
```
Catat: beda komponen sebelum vs setelah disable. Kapan Anda akan disable traefik juga?

---

## Bagian 4 — Topologi On-Prem

### 4.1 Dokumentasikan Topologi
Buat `m2.2/lab/log-latihan.md` (atau `topologi.md`) dengan:
```bash
for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do echo "| $n | $(orb ip $n) |"; done 2>/dev/null
```
Isi: tabel node IP, range ClusterIP/Pod/LB pool, komponen disable, jalur akses dari Mac.

### 4.2 IP Stabil
```bash
orb stop k3s-cp1; sleep 3; orb start k3s-cp1; sleep 5; orb ip k3s-cp1
```
Catat: apakah IP berubah? Kenapa IP stabil penting untuk: (a) etcd peer, (b) Ansible inventory, (c) MetalLB?

### 4.3 Akses Placeholder (Sebelum MetalLB)
```bash
kubectl expose deployment web --port=80 --type=NodePort -n default 2>/dev/null
NODEPORT=$(kubectl get svc web -n default -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null)
curl -s http://$(orb ip k3s-cp1):$NODEPORT/ | head -1
```
Catat: beda NodePort (sekarang) vs LoadBalancer IP (MetalLB, Modul 2.3) — mana yang "production-like"?

---

## Bagian 5 — Soal Refleksi

Tulis jawaban singkat di `m2.2/lab/log-latihan.md`:
1. Sebut 2 perbedaan teknis install k3d vs k3s (dari sudut "rumahnya": container vs VM, systemd, kubeconfig).
2. Kenapa jumlah server HA harus ganjil (3/5)? Apa yang terjadi pada 2 server saat 1 mati?
3. Kenapa `--disable servicelb` wajib sebelum pasang MetalLB? Apa yang bentrok?
4. Di cloud, Service `type=LoadBalancer` otomatis dapat IP. Di on-prem, apa kekosongan itu & siapa yang mengisi (modul mana)?
5. Anda menginstall k3s manual di 5 VM. Sebut 1 alasan ini painful & apa solusinya di bootcamp (fase mana)?
6. Sebut 2 kebiasaan agar tidak salah `kubectl delete` saat punya context k3d-lab & k3s-ha.

---

## Catatan Performa

- [ ] Semua latihan di terminal OrbStack VM + Mac
- [ ] Output penting disimpan di `m2.2/lab/log-latihan.md` di repo
- [ ] Bisa install k3s single-node di VM & ambil kubeconfig ke Mac (dua lane)
- [ ] Bisa install k3s HA multi-node (3 server + agent) & verifikasi quorum
- [ ] Bisa disable komponen (`servicelb`, `traefik`) & menjelaskan kenapa
- [ ] Bisa mendokumentasikan topologi on-prem (tabel IP, range, jalur akses)
- [ ] Bisa menjelaskan k3d (fast lane) vs k3s (production lane) secara final