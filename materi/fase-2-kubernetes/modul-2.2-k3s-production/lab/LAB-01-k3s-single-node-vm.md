# LAB-01 — k3s Single-Node di VM OrbStack

> **Target:** memasang k3s di VM Ubuntu OrbStack (systemd, static IP), mengambil kubeconfig ke Mac, men-deploy app, dan merasakan install "production-like" — bukan container (k3d).

## Latar Belakang
Modul 2.1 semuanya di k3d (container). Sekarang pertama kali install k3s di **VM sungguhan**: systemd service, static IP stabil, kubeconfig disalin manual. Inilah cara server on-prem dipasang — skill yang di-automasi Ansible di Fase 4, dan fondasi MetalLB (Modul 2.3). Bandingkan langsung dengan k3d: lebih lambat setup, tapi "nyata".

## Prasyarat
- [ ] Modul 2.1 selesai (paham kubectl + objek inti; punya context k3d-lab)
- [ ] OrbStack jalan, resource limit ≥6GB
- [ ] Image app di GitLab registry (Fase 1)
- [ ] Repo `sre-bootcamp` di GitLab

## Waktu
± 75 menit

## Langkah

### 1. Buat VM & SSH Standar

```bash
orb create -a ubuntu:24.04 k3s-cp1 2>/dev/null || orb start k3s-cp1
orb ls
orb ip k3s-cp1                              # catat: IP stabil (mis. 192.168.97.10)

# SSH key (untuk Ansible nanti)
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@$(orb ip k3s-cp1)

# Alias SSH
cat >> ~/.ssh/config <<EOF
Host k3s-cp1
    HostName $(orb ip k3s-cp1)
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
EOF
ssh k3s-cp1 'hostname; uname -m; systemctl is-system-running'
```

Update OS:
```bash
ssh k3s-cp1 'sudo apt update && sudo apt -y upgrade'
```

### 2. Install k3s Single-Node

```bash
ssh k3s-cp1
# Di dalam VM:
curl -sfL https://get.k3s.io | sh -
# k3s terpasang sebagai systemd service: k3s.service

sudo systemctl status k3s --no-pager | head
sudo k3s kubectl get nodes                 # 1 node, Ready
sudo k3s kubectl get pods -A               # CoreDNS, Traefik, servicelb, metrics-server
```

Verifikasi systemd mengelola k3s (auto-start saat reboot):
```bash
sudo systemctl is-enabled k3s             # enabled
# Simulasi reboot (opsional, hati-hati): orb stop k3s-cp1 && orb start k3s-cp1 → k3s auto-up
```

### 3. Ambil Kubeconfig ke Mac

```bash
# Di Mac:
VMIP=$(orb ip k3s-cp1)

# Salin kubeconfig (butuh sudo di VM karena file root-owned)
ssh k3s-cp1 'sudo cat /etc/rancher/k3s/k3s.yaml' > /tmp/k3s-cp1.yaml

# Ganti 127.0.0.1 → IP VM (agar API server reachable dari Mac)
sed -i '' "s/127.0.0.1/$VMIP/g" /tmp/k3s-cp1.yaml

# Gabung ke kubeconfig Mac dengan context unik
KUBECONFIG=~/.kube/config:/tmp/k3s-cp1.yaml kubectl config view --flatten > ~/.kube/config.new
mv ~/.kube/config.new ~/.kube/config
kubectl config rename-context default k3s-cp1 2>/dev/null || true
kubectl config use-context k3s-cp1
kubectl get nodes                          # dari Mac, ke cluster VM
```

**Verifikasi dua lane:**
```bash
kubectl config get-contexts
# k3d-lab   (fast lane — Modul 2.1)
# k3s-cp1   (production lane — modul ini)
kubectl config use-context k3d-lab && kubectl get nodes
kubectl config use-context k3s-cp1 && kubectl get nodes
```

### 4. Deploy App (Manifest dari Modul 2.1)

Pakai manifest yang sudah Anda buat di Modul 2.1 LAB-01 (atau buat ulang cepat):
```bash
kubectl create ns demo
kubectl config set-context --current --namespace=demo

# Pakai deployment.yaml + service.yaml dari m2.1/lab01 (ganti image path bila perlu)
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
kubectl apply -f m2.1/lab01/deployment.yaml -n demo
kubectl apply -f m2.1/lab01/service.yaml -n demo
kubectl get pod -o wide -n demo            # Pod jalan di VM k3s-cp1
kubectl get svc -n demo
```

> **Kalau image dari GitLab private registry:** buat `imagePullSecret` (lihat Modul 2.1 LAB-01) di ns `demo`.

### 5. Akses dari Mac (Port-Forward — Sebelum Ingress/MetalLB)

```bash
kubectl port-forward svc/app 9090:80 -n demo &
sleep 2
curl -s http://localhost:9090/
kill %1 2>/dev/null
```

**Catatan:** tanpa port mapping k3d (`@loadbalancer`) atau MetalLB, akses external ke k3s butuh port-forward (debug) atau Ingress (Layer 7). Topologi akses "production-like" (MetalLB) datang Modul 2.3.

### 6. Opsional: Disable Traefik & Servicelb (Siap MetalLB)

Untuk bersiap Modul 2.3, reinstall dengan disable (cara paling bersih untuk lab):
```bash
# Di VM:
ssh k3s-cp1
sudo /usr/local/bin/k3s-uninstall.sh
curl -sfL https://get.k3s.io | sh -s - server --disable traefik --disable servicelb
sudo k3s kubectl get pods -A              # traefik & svclb hilang

# Tes Service LB pending (MetalLB belum dipasang)
sudo k3s kubectl create deployment web --image=nginx:alpine -n default
sudo k3s kubectl expose deployment web --port=80 --type=LoadBalancer -n default
sudo k3s kubectl get svc web -n default    # EXTERNAL-IP: <pending> ← harapan
exit
```

> Kalau ingin tetap pakai Traefik untuk eksperimen Ingress, skip langkah ini (jangan reinstall). Simpan VM dengan Traefik untuk Lab ini; disable dilakukan di LAB-02 / Modul 2.3.

### 7. Catat & Commit

```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
git switch -c m2.2-lab01
mkdir -p m2.2/lab01
cat > m2.2/lab01/report.md <<'EOF'
# m2.2 LAB-01 — k3s Single-Node di VM OrbStack

## Bukti
- `kubectl get nodes` (dari Mac, context k3s-cp1):
  ```
  (tempel)
  ```
- `kubectl get pods -A` (komponen bawaan):
  ```
  (tempel)
  ```
- `curl http://localhost:9090/` (port-forward):
  ```
  (tempel)
  ```

## Catatan
- VM IP: ... (stabil? coba stop/start)
- Beda dengan k3d (Modul 2.1 LAB-01): (1-2 hal)
- systemd: `systemctl is-enabled k3s` = ?
- (kalau disable) Service LB pending terbukti: ?
EOF

git add m2.2/lab01/report.md
git commit -m "feat(m2.2): k3s single-node di VM OrbStack

- install k3s via curl|sh di ubuntu VM (systemd, static IP)
- kubeconfig ke Mac (ganti IP, context k3s-cp1)
- deploy app (manifest dari m2.1) + port-forward
- dua lane: k3d-lab vs k3s-cp1

Closes #<issue-m2.2-lab01>"
git push -u origin m2.2-lab01
```
Buat MR → squash & merge.

## Acceptance Criteria

- [ ] VM `k3s-cp1` (Ubuntu) dibuat, IP stabil, SSH standar (ssh-copy-id + config alias)
- [ ] k3s terinstall sebagai systemd service (`systemctl status k3s`, `is-enabled` = enabled)
- [ ] `kubectl` dari Mac berfungsi (context `k3s-cp1`) — `kubectl get nodes` dari Mac
- [ ] Dua context terlihat: `k3d-lab` (k3d) & `k3s-cp1` (k3s); bisa berpindah
- [ ] App ter-deploy di VM; `kubectl get pod -o wide` menunjukkan Pod di node `k3s-cp1`
- [ ] `port-forward` 9090:80 bisa di-curl dari Mac
- [ ] (Opsional) disable traefik+servicelb → Service LB `pending` terbukti
- [ ] `m2.2/lab01/report.md` ter-commit via MR

## Troubleshooting

| Gejala | Solusi |
|---|---|
| `scp /etc/rancher/k3s/k3s.yaml` permission denied | file root-owned; pakai `ssh k3s-cp1 'sudo cat /etc/rancher/k3s/k3s.yaml'` > file |
| `kubectl` dari Mac: connection refused 6443 | IP VM belum diganti di kubeconfig (`sed 127.0.0.1→IP`); cek `orb ip` |
| `kubectl` x509 cert error | `--tls-san` tidak termasuk IP VM; reinstall dengan `--tls-san <IP>` atau akses via `https://<IP>:6443` dengan `--insecure-skip-tls-verify` (debug saja) |
| k3s service failed | `sudo journalctl -u k3s --no-pager`; cek port bentrok (6443/8443) atau disk penuh |
| Pod `ImagePullBackOff` | image registry private tanpa secret; atau arch mismatch (arm64) |
| `orb ip` berubah | OrbStack Machine IP seharusnya stabil; pastikan bukan recreate (orb create vs start) |
| Mac lag | VM OrbStack makan RAM; `orb stop` VM tidak terpakai; naikkan limit sementara |

## Catatan SRE
- **VM + systemd = production-like.** k3s auto-start saat reboot (systemd), IP stabil (static-like), uninstall bersih (`k3s-uninstall`). Ini berbeda fundamental dari k3d (container, ephemeral).
- **Kubeconfig manual = disiplin.** k3d auto-merge; k3s Anda salin + ganti IP. Kebiasaan ini penting saat menyiapkan banyak cluster (Fase 6: staging + production).
- **Dua lane, dua context** — selalu `kubectl config current-context` sebelum operasi destruktif. Context salah = bencana di production (Fase 6).
- **Servicelb pending = "siap MetalLB".** Tanpa LB on-prem, Service LoadBalancer tidak dapat IP — itulah kekosongan yang MetalLB isi (Modul 2.3).

## Lanjut
[LAB-02: k3s Multi-Node & Topologi HA](LAB-02-k3s-multi-node-topologi.md)