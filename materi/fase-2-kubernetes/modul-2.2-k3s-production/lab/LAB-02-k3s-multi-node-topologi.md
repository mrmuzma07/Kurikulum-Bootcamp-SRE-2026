# LAB-02 — k3s Multi-Node HA & Topologi On-Prem

> **Target:** memasang cluster k3s HA (3 server + 2 agent) di 5 VM OrbStack, memverifikasi etcd quorum, menonaktifkan servicelb (siap MetalLB), dan mensimulasikan kegagalan node — sambil mendokumentasikan topologi on-prem.

## Latar Belakang
LAB-01 install k3s single-node — mudah, tapi satu titik gagal. Sekarang bangun **HA**: 3 server (etcd quorum) + 2 agent (worker). Ini simulasi datacenter mini: static IP per node, join via token, disable servicelb untuk persiapan MetalLB (Modul 2.3). Install manual di 5 VM akan terasa berulang — itulah kenapa Fase 4 (Ansible) meng-automasi ini. Dokumentasikan topologi (deliverable SRE).

## Prasyarat
- [ ] LAB-01 selesai (paham install k3s single-node, kubeconfig ke Mac)
- [ ] OrbStack resource limit ≥10–12 GB (5 VM) — naikkan sementara
- [ ] Repo `sre-bootcamp`, paham quorum etcd (topik 02)

## Waktu
± 120 menit

## Langkah

### 1. Siapkan 5 VM OrbStack

```bash
# (k3s-cp1 mungkin sudah ada dari LAB-01; uninstall k3s di dalamnya dulu agar bersih)
ssh k3s-cp1 'sudo /usr/local/bin/k3s-uninstall.sh 2>/dev/null; exit' 2>/dev/null

# Buat 4 VM lain
for n in k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do
  orb create -a ubuntu:24.04 $n 2>/dev/null || orb start $n
done
orb ls

# Catat IP semua node (stabil) — ini dasar topologi
echo "=== Node IP ==="
for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do echo "$n: $(orb ip $n)"; done
```

SSH setup (Fase 4 meng-automasi; sekarang loop):
```bash
for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do
  ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@$(orb ip $n) 2>/dev/null
  ssh ubuntu@$(orb ip $n) 'sudo apt update && sudo apt -y upgrade' 2>/dev/null
  # SSH config alias
  cat >> ~/.ssh/config <<EOF
Host $n
    HostName $(orb ip $n)
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
EOF
done
```

### 2. Install Server Pertama (cluster-init, disable servicelb)

Disable `servicelb` sejak awal — siap MetalLB (Modul 2.3). Pertahankan Traefik untuk Ingress (bisa diganti nanti).

```bash
CP1_IP=$(orb ip k3s-cp1)

ssh k3s-cp1 "curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --disable servicelb \
  --tls-san $CP1_IP \
  --node-external-ip $CP1_IP"

# Tunggu node Ready
ssh k3s-cp1 'sudo k3s kubectl get nodes'
```

Ambil token (dipakai node lain untuk join):
```bash
TOKEN=$(ssh k3s-cp1 'sudo cat /var/lib/rancher/k3s/server/node-token')
echo "Token: $TOKEN"
```

### 3. Join Server 2 & 3 (etcd quorum 3)

```bash
for n in k3s-cp2 k3s-cp3; do
  IP=$(orb ip $n)
  ssh $n "curl -sfL https://get.k3s.io | sh -s - server \
    --server https://$CP1_IP:6443 \
    --token $TOKEN \
    --disable servicelb \
    --node-external-ip $IP"
done

# Verifikasi 3 server Ready
ssh k3s-cp1 'sudo k3s kubectl get nodes -o wide'
```

### 4. Join Agent (Worker) 2

```bash
for n in k3s-w1 k3s-w2; do
  IP=$(orb ip $n)
  ssh $n "curl -sfL https://get.k3s.io | sh -s - agent \
    --server https://$CP1_IP:6443 \
    --token $TOKEN \
    --node-external-ip $IP"
done

# Verifikasi 5 node
ssh k3s-cp1 'sudo k3s kubectl get nodes -o wide'
# 3 control-plane + 2 worker, semua Ready
```

### 5. Kubeconfig ke Mac & Verifikasi Quorum

```bash
ssh k3s-cp1 'sudo cat /etc/rancher/k3s/k3s.yaml' | sed "s/127.0.0.1/$CP1_IP/" > /tmp/k3s-ha.yaml
KUBECONFIG=~/.kube/config:/tmp/k3s-ha.yaml kubectl config view --flatten > ~/.kube/config.new
mv ~/.kube/config.new ~/.kube/config
kubectl config rename-context default k3s-ha 2>/dev/null || true
kubectl config use-context k3s-ha

kubectl get nodes -o wide
kubectl get pods -n kube-system | grep -E "etcd|coredns|traefik|metrics"
# svclb-* harus TIDAK ada (servicelb disabled)

# Verifikasi etcd healthy (3 member)
kubectl get --raw='/readyz?verbose' 2>/dev/null | grep -i etcd
# atau via k3s (di server):
ssh k3s-cp1 'sudo k3s etcd-snapshot ls 2>/dev/null | head'   # konfirmasi etcd
```

### 6. Deploy App & Amati Distribusi Pod (5 Node)

```bash
kubectl create ns demo && kubectl config set-context --current --namespace=demo
kubectl create deployment app --image=nginx:alpine --replicas=5
kubectl get pod -o wide        # tersebar di server + agent (5 Pod, 5 node idealnya)
```

### 7. Simulasi Kegagalan & Quorum

```bash
# (A) Matikan 1 SERVER (cp3) — quorum masih 2/3, cluster tetap jalan
orb stop k3s-cp3
sleep 15
kubectl get nodes              # cp3 NotReady, cp1+cp2 Ready
kubectl get pod -o wide        # Pod di cp3 reschedule ke node lain
kubectl get --raw='/readyz?verbose' 2>/dev/null | grep -i etcd   # etcd masih quorum

# Pulihkan
orb start k3s-cp3
sleep 20
kubectl get nodes              # cp3 Ready lagi, rejoin etcd
```

```bash
# (B — opsional, hati-hati) Matikan 2 SERVER → quorum hilang
orb stop k3s-cp2 k3s-cp3
sleep 10
kubectl get nodes 2>&1         # error/timeout — etcd read-only, cluster stop
# (ini mengajarkan: 3 server tahan 1, bukan 2)

# PULIHKAN (jangan biarkan begini)
orb start k3s-cp2 k3s-cp3
sleep 20
kubectl get nodes              # kembali normal setelah quorum terbentuk
```

> **Catat hasil bagian B:** berapa lama cluster "mati"? Ini mengapa production sering pakai 5 server (tahan 2).

### 8. Dokumentasikan Topologi (Deliverable)

```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
mkdir -p m2.2/lab02
cat > m2.2/lab02/topologi.md <<EOF
# m2.2 LAB-02 — Topologi k3s HA di OrbStack

## Node
$(for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do echo "| $n | $(orb ip $n) | $([ $n = k3s-w1 ] || [ $n = k3s-w2 ] && echo 'agent (worker)' || echo 'server (etcd)') |"; done)

| Nama      | IP             | Peran           |
|-----------|----------------|-----------------|
(tempel tabel di atas, rapihkan)

## Range IP
- Node: (daftar IP node di atas)
- ClusterIP (Service): 10.43.0.0/16 (default k3s)
- Pod: 10.42.0.0/16 (default k3s)
- LoadBalancer pool (MetalLB, Modul 2.3): (pilih range bebas, mis. 192.168.97.200–.250)

## Komponen
- servicelb: DISABLED (siap MetalLB)
- traefik: aktif (bawaan) — atau ganti nanti
- etcd: 3 server, quorum 2/3 (tahan 1 server mati)

## Akses dari Mac
- kubectl: context k3s-ha → CP1:6443
- Service type=LoadBalancer: <pending> (MetalLB belum dipasang)

## Bukti Chaos
- Stop cp3 (1 server): cluster tetap jalan, Pod reschedule. Waktu pulih: ... detik
- Stop cp2+cp3 (2 server): cluster ... (mati/pulih?) — pelajaran quorum
EOF
```

### 9. Bersihkan (Pilih Salah Satu)

**Opsi A — simpan untuk Modul 2.3 (MetalLB):**
```bash
# Jangan uninstall. Pastikan semua VM jalan:
for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do orb start $n; done
kubectl get nodes
# Turunkan k3s agent yang tidak perlu bila Mac lag; atau orb stop k3s-w2
```

**Opsi B — hapus semua (kalau ingin mulai bersih di 2.3):**
```bash
for n in k3s-w1 k3s-w2; do ssh $n 'sudo /usr/local/bin/k3s-agent-uninstall.sh'; done
for n in k3s-cp3 k3s-cp2 k3s-cp1; do ssh $n 'sudo /usr/local/bin/k3s-uninstall.sh'; done
kubectl config delete-context k3s-ha 2>/dev/null
for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do orb stop $n && orb rm $n; done
```

### 10. Commit

```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
git switch -c m2.2-lab02
git add m2.2/lab02/topologi.md
git commit -m "feat(m2.2): k3s multi-node HA + topologi on-prem

- 3 server (etcd quorum) + 2 agent di 5 VM OrbStack
- disable servicelb (siap MetalLB modul 2.3)
- verifikasi quorum: tahan 1 server mati
- dokumentasi topologi (node IP, range, jalur akses)

Closes #<issue-m2.2-lab02>"
git push -u origin m2.2-lab02
```
Buat MR → squash & merge.

## Acceptance Criteria

- [ ] 5 VM OrbStack dibuat (k3s-cp1/2/3 + k3s-w1/2), IP stabil, SSH standar (alias config)
- [ ] k3s HA terinstall: 3 server (`--cluster-init` + join) + 2 agent (join via token)
- [ ] `kubectl get nodes` menunjukkan 3 control-plane + 2 worker, semua Ready
- [ ] `servicelb` disabled — `svclb-*` tidak ada di `kube-system`
- [ ] etcd quorum terverifikasi (3 member; `readyz` etcd healthy)
- [ ] Pod app tersebar di ≥3 node (5 replica)
- [ ] Simulasi stop 1 server: cluster tetap jalan, Pod reschedule, pulih saat start
- [ ] (Opsional) stop 2 server: cluster mati — catat pelajaran quorum
- [ ] `m2.2/lab02/topologi.md` lengkap (tabel node, range IP, komponen, jalur akses)
- [ ] Topologi ter-commit via MR

## Troubleshooting

| Gejala | Solusi |
|---|---|
| Server join gagal (`connection refused`) | token salah; CP1 IP salah; firewall; cek `ssh k3s-cp1 'sudo journalctl -u k3s --no-pager \| tail'` |
| Agent join gagal | token sama dengan server; cek `journalctl -u k3s-agent` |
| `get nodes` hanya 3 (agent tidak muncul) | agent service failed; `ssh k3s-w1 'sudo systemctl status k3s-agent'` |
| etcd ` unhealthy` / quorum error | 1 server mati + 2 hidup harus quorum; kalau 2 mati cluster mati — start kembali |
| Mac lag / OOM | 5 VM berat; naikkan limit OrbStack; `orb stop` worker; atau turunkan ke 3 server saja |
| `kubectl` dari Mac error setelah stop cp1 | kubeconfig only point ke CP1; arahkan ke CP2 IP, atau pakai loadbalancer API (advance) |
| Uninstall tidak bersih | hapus `/var/lib/rancher/k3s` manual: `sudo rm -rf /var/lib/rancher/k3s` lalu reinstall |
| Cert error setelah reinstall | `--tls-san` tidak include IP; reinstall dengan flag |

## Catatan SRE
- **3 server = ganjil & tahan 1.** Jangan 2 server (quorum hilang saat 1 mati). 5 server untuk production besar (tahan 2). Pilih sesuai RTO/RPO.
- **Disable servicelb sejak awal.** Lebih mudah daripada disable pasca-install. Ansible (Fase 4) akan memastikan flag konsisten di semua node. MetalLB (Modul 2.3) butuh ini.
- **Topologi = dokumentasi wajib.** Tabel node IP & range menjadi `inventory` Ansible (Fase 4). Tanpa dokumentasi, multi-node jadi liar.
- **Install manual = painful = kenapa IaC.** 5 VM × perintah berulang. Fase 4 Ansible menyelesaikan ini dalam satu `ansible-playbook`. Rasakan dulu manualnya untuk menghargai otomasi.
- **kubeconfig single-server = risiko.** kubeconfig point ke CP1; CP1 mati = `kubectl` gagal walau cluster (CP2/3) sehat. Production pakai API loadbalancer (HAProxy/konfigurasi) di depan 3 server. (Konsep; bootcamp pakai CP1 cukup.)

## Lanjut
[Evaluasi: Latihan & Kuis](../evaluasi/latihan.md)

---

**Modul 2.2 selesai.** Lanjut ke [Modul 2.3 — MetalLB](../../modul-2.3-metallb/README.md) (menyusul) — pasang LoadBalancer bare-metal di cluster HA ini.