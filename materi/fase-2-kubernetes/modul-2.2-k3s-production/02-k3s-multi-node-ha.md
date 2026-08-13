# 02 — k3s Multi-Node & High Availability

> Dari single-node ke cluster HA: embedded etcd, `--cluster-init`, join agent/server, dan quorum.

## Tujuan
- Bisa install k3s multi-node HA (≥3 server) di beberapa VM OrbStack
- Paham embedded etcd HA & quorum (R=3 tahan 1 gagal, R=5 tahan 2)
- Bisa join agent (worker) ke cluster
- Bisa memverifikasi quorum & dampak kehilangan node
- Bisa uninstall k3s multi-node dengan benar

## 1. Kenapa HA? Single-Node vs Multi-Server

Single-node (Modul 2.1, topik sebelumnya): 1 server = control plane + etcd + worker. **Mudah, tapi satu titik gagal** — server mati = cluster mati.

**HA control plane:** ≥3 server, etcd **terdistribusi** (cluster). Satu server mati → 2 lain tetap membentuk quorum → cluster tetap jalan. Worker (agent) bisa banyak terpisah.

| Topologi | Server | Etcd | Tahan gagal | Cocok |
|---|---|---|---|---|
| Single-node | 1 | SQLite | 0 | lab cepat, dev |
| HA minimal | 3 | embedded etcd | 1 server | simulasi production |
| HA besar | 5 | embedded etcd | 2 server | production besar |

**Aturan etcd quorum:** mayoritas node harus hidup. 3 node → butuh 2 (tahan 1 mati). 5 node → butuh 3 (tahan 2). **Jangan 2 server** — etcd quorum tidak ada (butuh mayoritas dari 2 = 2; 1 mati = mayoritas hilang = cluster berhenti). Selalu **ganjil** (3 atau 5).

## 2. Topologi Lab (3 Server + 2 Agent)

```
 ┌─────────────────────── OrbStack (Mac) ───────────────────────┐
 │                                                              │
 │  ┌─ k3s-cp1 ─┐  ┌─ k3s-cp2 ─┐  ┌─ k3s-cp3 ─┐   (server, etcd)│
 │  │  server   │  │  server   │  │  server   │                 │
 │  │  etcd     │◄─┤  etcd     │◄─┤  etcd     │  ← quorum 3     │
 │  └───────────┘  └───────────┘  └───────────┘                 │
 │      ▲              ▲              ▲                         │
 │      └──────────────┼──────────────┘                         │
 │                     │  (API: 6443, etcd peer: 6443/2380)      │
 │  ┌─ k3s-w1 ─┐  ┌─ k3s-w2 ─┐   (agent / worker, Pod jalan)    │
 │  │  agent    │  │  agent   │                                 │
 │  └──────────┘  └──────────┘                                 │
 │                                                              │
 └──────────────────────────────────────────────────────────────┘
        ▲ kubectl dari Mac (kubeconfig → salah satu server:6443)
```

- **3 server** (`k3s-cp1/2/3`): control plane + etcd + worker (bisa juga Pod).
- **2 agent** (`k3s-w1/2`): worker saja (Pod, Service traffic). Hemat: agent tidak ikut etcd.
- **Quorum 2 dari 3** → 1 server mati, cluster tetap jalan.

## 3. Siapkan 5 VM OrbStack

```bash
for n in k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do
  orb create -a ubuntu:24.04 $n 2>/dev/null || orb start $n
done
orb ls
# Catat semua IP (stabil):
for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do echo "$n: $(orb ip $n)"; done
```

SSH setup (Fase 4 meng-automasi ini; sekarang manual):
```bash
for n in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do
  ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@$(orb ip $n) 2>/dev/null
  ssh ubuntu@$(orb ip $n) 'sudo apt update && sudo apt -y upgrade' 2>/dev/null
done
```

> **Catatan resource:** 5 VM + k3s butuh banyak RAM. Naikkan limit OrbStack ke ~12 GB sementara; `orb stop` VM yang tidak terpakai setelah lab. Untuk Mac 16GB, 3 server saja cukup (skip agent); untuk 8GB Mac, gunakan 3 server dengan limit ketat.

## 4. Install Server Pertama (`--cluster-init`)

Server pertama **menginisialisasi** cluster etcd:

```bash
ssh k3s-cp1
# di k3s-cp1:
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --tls-san $(hostname -I | awk '{print $1}') \
  --node-external-ip $(hostname -I | awk '{print $1}')
```

| Opsi | Arti |
|---|---|
| `server` | jalankan sebagai server (control plane + etcd) |
| `--cluster-init` | inisialisasi embedded etcd cluster (server pertama) |
| `--tls-san <ip>` | tambahkan IP ke cert API server (agar server lain & Mac bisa reach) |
| `--node-external-ip <ip>` | IP yang dilihat cluster untuk node |

Ambil **token cluster** (dipakai node lain untuk join):
```bash
sudo cat /var/lib/rancher/k3s/server/node-token    # di k3s-cp1
# catat token ini (mis. K10...::server:...)
```

## 5. Join Server Kedua & Ketiga

Server 2 & 3 join ke server 1 (membentuk etcd quorum):

```bash
# Di Mac, dapatkan IP cp1 & token
CP1_IP=$(orb ip k3s-cp1)
TOKEN=$(ssh k3s-cp1 'sudo cat /var/lib/rancher/k3s/server/node-token')

# Install di cp2 & cp3
for n in k3s-cp2 k3s-cp3; do
  ssh $n "curl -sfL https://get.k3s.io | sh -s - server \
    --server https://$CP1_IP:6443 \
    --token $TOKEN \
    --node-external-ip \$(hostname -I | awk '{print \$1}')"
done
```

## 6. Join Agent (Worker)

Agent join ke API server (lewat load ke server mana saja — k3s handle):

```bash
AGENT_TOKEN=$(ssh k3s-cp1 'sudo cat /var/lib/rancher/k3s/server/agent-node-token 2>/dev/null || sudo cat /var/lib/rancher/k3s/server/node-token')
# (k3s pakai node-token untuk server & agent; agent bisa pakai token yang sama)

for n in k3s-w1 k3s-w2; do
  ssh $n "curl -sfL https://get.k3s.io | sh -s - agent \
    --server https://$CP1_IP:6443 \
    --token $AGENT_TOKEN \
    --node-external-ip \$(hostname -I | awk '{print \$1}')"
done
```

## 7. Verifikasi Cluster

```bash
# Ambil kubeconfig dari cp1 ke Mac (seperti topik 01)
CP1_IP=$(orb ip k3s-cp1)
ssh k3s-cp1 'sudo cat /etc/rancher/k3s/k3s.yaml' | sed "s/127.0.0.1/$CP1_IP/" > /tmp/k3s-ha.yaml
KUBECONFIG=~/.kube/config:/tmp/k3s-ha.yaml kubectl config view --flatten > ~/.kube/config
kubectl config rename-context default k3s-ha 2>/dev/null || true
kubectl config use-context k3s-ha

kubectl get nodes -o wide
# NAME       STATUS   ROLES                  AGE   VERSION
# k3s-cp1    Ready    control-plane,master   ...    v1.xx+k3s1
# k3s-cp2    Ready    control-plane,master   ...    v1.xx+k3s1
# k3s-cp3    Ready    control-plane,master   ...    v1.xx+k3s1
# k3s-w1     Ready    <none>                 ...    v1.xx+k3s1
# k3s-w2     Ready    <none>                 ...    v1.xx+k3s1

# Verifikasi etcd quorum
kubectl get pods -n kube-system | grep etcd      # atau:
ssh k3s-cp1 'sudo k3s etcd-snapshot list 2>/dev/null; sudo k3s kubectl get --raw="/readyz?verbose" 2>/dev/null | grep etcd'
```

Deploy app & amati distribusi Pod di 5 node:
```bash
kubectl create ns demo && kubectl config set-context --current --namespace=demo
kubectl create deployment app --image=nginx:alpine --replicas=5
kubectl get pod -o wide        # tersebar di server + agent
```

## 8. Simulasi Kegagalan & Quorum

```bash
# Matikan 1 server (cp3) — quorum masih 2/3, cluster tetap jalan
orb stop k3s-cp3
sleep 10
kubectl get nodes              # cp3 NotReady, lainnya Ready
kubectl get pod -o wide        # Pod di cp3 reschedule; Pod lain tetap

# Verifikasi API masih merespons (via cp1/cp2)
kubectl get nodes

# Pulihkan
orb start k3s-cp3
sleep 15
kubectl get nodes              # cp3 Ready lagi, rejoin etcd
```

**Bahaya: matikan 2 server** → quorum hilang (1 dari 3) → cluster **stop** (etcd read-only). Ini mengapa 3 server = tahan 1, bukan 2. Jangan dicoba kecuali untuk belajar recovery (advance).

## 9. Uninstall k3s Multi-Node

```bash
# Agent dulu (kalau ada), lalu server. Hapus etcd data total.
for n in k3s-w1 k3s-w2; do
  ssh $n 'sudo /usr/local/bin/k3s-agent-uninstall.sh'
done
for n in k3s-cp3 k3s-cp2 k3s-cp1; do
  ssh $n 'sudo /usr/local/bin/k3s-uninstall.sh'
done
kubectl config delete-context k3s-ha 2>/dev/null
```

> **Urutan penting:** hapus agent sebelum server agar Pod tidak orphan; hapus server dari yang "bukan pertama" ke yang pertama (atau bebas — k3s-uninstall bersihkan etcd lokal). Setelah uninstall, `k3s-uninstall` hapus `/var/lib/rancher/k3s` sepenuhnya.

## Latihan Cepat (30 menit)

```bash
# 1. 3 VM server
for n in k3s-cp1 k3s-cp2 k3s-cp3; do orb create -a ubuntu:24.04 $n 2>/dev/null || orb start $n; done
CP1=$(orb ip k3s-cp1)

# 2. cluster-init di cp1
ssh k3s-cp1 "curl -sfL https://get.k3s.io | sh -s - server --cluster-init --node-external-ip $CP1"
TOKEN=$(ssh k3s-cp1 'sudo cat /var/lib/rancher/k3s/server/node-token')

# 3. join cp2, cp3
for n in k3s-cp2 k3s-cp3; do
  ssh $n "curl -sfL https://get.k3s.io | sh -s - server --server https://$CP1:6443 --token $TOKEN"
done

# 4. kubeconfig ke Mac & verifikasi
ssh k3s-cp1 'sudo cat /etc/rancher/k3s/k3s.yaml' | sed "s/127.0.0.1/$CP1/" > /tmp/k3s.yaml
KUBECONFIG=~/.kube/config:/tmp/k3s.yaml kubectl config view --flatten > ~/.kube/config
kubectl config use-context default
kubectl get nodes

# 5. Chaos: stop cp3, amati quorum
orb stop k3s-cp3; sleep 10; kubectl get nodes; orb start k3s-cp3
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Init cluster HA | server: `curl ... \| sh -s - server --cluster-init` |
| Join server | `curl ... \| sh -s - server --server https://CP1:6443 --token <T>` |
| Join agent | `curl ... \| sh -s - agent --server https://CP1:6443 --token <T>` |
| Token | `/var/lib/rancher/k3s/server/node-token` (di cp1) |
| Quorum 3 server | tahan 1 mati; **jangan 2 server** (ganjil) |
| Uninstall agent/server | `k3s-agent-uninstall.sh` / `k3s-uninstall.sh` |

## Cek Pemahaman

1. Kenapa jumlah server HA harus ganjil (3/5)? Apa yang terjadi pada 2 server saat 1 mati?
2. Apa perbedaan `--cluster-init` (cp1) vs `--server https://CP1:6443` (cp2/cp3)?
3. Quorum 3 server = tahan berapa node gagal? Quorum 5 server?
4. Setelah uninstall semua server, apakah data Pod hilang? Kenapa "bersih" penting sebelum install ulang?