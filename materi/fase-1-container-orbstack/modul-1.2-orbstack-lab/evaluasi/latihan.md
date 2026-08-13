# Latihan — Modul 1.2 OrbStack sebagai Lab Harian

Latihan ini memperkuat pemahaman **OrbStack Machine**, **resource limit**, dan **k3d**. Kerjakan di terminal Mac (OrbStack).

> **Aturan:** kerjakan di terminal. Catat output penting di `m1.2/lab/log-latihan.md` di repo `sre-bootcamp`.

---

## Bagian 1 — OrbStack Machine

### 1.1 Buat & Eksplorasi Machine
```bash
orb create -a ubuntu:24.04 devbox 2>/dev/null || orb start devbox
orb ls
orb ssh devbox 'systemctl is-system-running; uname -m; uptime'
```
Catat: kenapa `systemctl is-system-running` bekerja di Machine tapi **tidak** di container biasa (coba `docker run --rm alpine systemctl is-system-running` → apa error-nya)?

### 1.2 IP Stabil
```bash
orb ip devbox
orb stop devbox && sleep 2 && orb start devbox && orb ip devbox
```
Catat: apakah IP berubah setelah stop/start? Jelaskan kenapa ini penting untuk simulasi on-prem.

### 1.3 Filesystem Sharing
```bash
orb ssh devbox 'ls /mnt/mac/Users 2>/dev/null | head; echo "---"; df -h /mnt/mac 2>/dev/null'
docker run --rm -v "$PWD:/work" alpine sh -c 'echo "host file:"; ls /work | head'
```
Catat: bagaimana folder Mac terlihat di Machine vs di container.

### 1.4 SSH Standar (untuk Ansible)
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@$(orb ip devbox)
# buat alias di ~/.ssh/config (lihat topik 01)
ssh devbox 'hostname; whoami'
```
Catat: beda `orb ssh devbox` vs `ssh devbox` — kenapa yang kedua penting untuk Ansible (Fase 4)?

---

## Bagian 2 — Resource Limit

### 2.1 Lihat & Set Limit
```bash
orb info 2>/dev/null || orb status
docker stats --no-stream
# Set memory limit via OrbStack UI: Settings → Resources (mis. 8GB)
```

### 2.2 Limit Per-Container vs Global
```bash
docker run -d --name hog --memory=1g alpine sh -c 'yes > /dev/null & sleep 3600'
docker stats hog --no-stream
# Amati CPU (1 core terpakai penuh oleh `yes`)
docker rm -f hog
```
Catat: kenapa memory limit **global** OrbStack harus ≥ jumlah limit **per-container** yang berjalan bersamaan?

### 2.3 Disk & Bersih-bersih
```bash
docker system df
docker images | wc -l
docker system prune -f
docker system df
```
Catat: berapa ruang yang dibebaskan? Kapan `prune -a` berbahaya dibanding `prune`?

---

## Bagian 3 — k3d Cluster

### 3.1 Cluster Multi-Node
```bash
brew install k3d kubectl
k3d cluster delete lab 2>/dev/null
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
kubectl get nodes -o wide
kubectl get pods -A | head
```
Catat: 1 server + 2 agent, semua Ready. Apa komponen bawaan k3s yang berjalan di `kube-system`?

### 3.2 Deploy & Ingress
Kerjakan [LAB-01](../lab/LAB-01-k3d-cluster.md) dan catat di laporan:
1. Bagaimana Pod tersebar di 3 node? Apakah anti-affinity default menyebar merata?
2. Setelah `docker stop k3d-lab-agent-0`, berapa detik sampai Pod reschedule ke node sehat?
3. Kenapa Ingress butuh host header (`-H 'Host: app.k3d.local'`) saat akses via curl?

### 3.3 Lifecycle
```bash
k3d cluster stop lab
kubectl get nodes            # error? kenapa?
k3d cluster start lab
kubectl get nodes
k3d cluster delete lab
docker ps | grep k3d         # harus kosong
```
Catat: apa yang terjadi pada Pod saat `k3d cluster stop` lalu `start`? Apakah Pod tetap ada?

---

## Bagian 4 — Soal Refleksi

Tulis jawaban singkat di `m1.2/lab/log-latihan.md`:
1. Seorang rekan bilang "OrbStack itu Docker Desktop yang diganti nama." Koreksi pernyataan ini dengan menjelaskan 2 perbedaan teknis.
2. Kenapa k3d cocok untuk latihan harian tetapi **tidak ideal** untuk simulasi MetalLB L2 mode?
3. Anda punya Mac 16GB. Berapa perkiraan limit OrbStack yang aman, dan kenapa bukan 14GB?
4. Jelaskan kapan Anda pakai **container** (`docker run`) vs **Machine** (`orb create`) vs **k3d cluster** di bootcamp ini — masing-masing 1 contoh.

---

## Catatan Performa

- [ ] Semua latihan di terminal OrbStack
- [ ] Output penting disimpan di `m1.2/lab/log-latihan.md` di repo
- [ ] Bisa menjelaskan OrbStack Machine vs container & kapan pakai masing-masing
- [ ] Bisa mengatur resource limit & menjelaskan hierarki (Mac → OrbStack → container)
- [ ] Bisa membuat/hapus cluster k3d multi-node & deploy app via Ingress
- [ ] Bisa menjelaskan kapan k3d vs k3s (fast lane vs production lane)