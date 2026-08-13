# 01 — Arsitektur Kubernetes

> Control plane, kubelet, container runtime, etcd — apa yang sebenarnya berjalan di balik `kubectl get pods`.

## Tujuan
- Bisa menjelaskan komponen control plane & fungsinya
- Mengerti peran kubelet & container runtime di node worker
- Paham apa itu etcd & kenapa ia "sumber kebenaran"
- Bisa memetakan jalur permintaan: `kubectl` → API server → etcd → kubelet → container

## 1. Dua Jenis Node

Cluster Kubernetes = kumpulan **node**. Dua peran:

```
 ┌─ control plane (otak) ──┐   ┌─ worker (otot) ───┐
 │ - menjalankan API server│   │ - menjalankan Pod  │
 │ - menyimpan state (etcd)│   │ - kubelet + CRI   │
 │ - scheduling, kontrol   │   │ - melayani traffic│
 └─────────────────────────┘   └───────────────────┘
```

- **Control plane** = otak: menerima perintah (`kubectl`), simpan state, pilih node mana yang jalankan Pod (scheduler), pastikan kondisi nyata = kondisi diinginkan (controller).
- **Worker** = otot: menjalankan container (Pod) sesuai perintah control plane, melayani traffic jaringan.

Di k3s/k3d, satu node bisa jadi **keduanya** (server = control plane + worker) — hemat untuk lab. Production besar pisahkan agar control plane tidak terganggu beban Pod.

```bash
# Lihat peran node
kubectl get nodes -o wide
# ROLES: control-plane,master  → control plane
# ROLES: <none>                 → worker (agent di k3d)
```

## 2. Komponen Control Plane

| Komponen | Fungsi | Dijalankan oleh |
|---|---|---|
| **kube-apiserver** | satu-satunya yang bicara dengan etcd; semua `kubectl`/client lewat sini (REST API) | k3s server |
| **etcd** | database key-value; simpan **semua** state cluster (objek, secret, konfig) — sumber kebenaran | embedded di k3s |
| **kube-scheduler** | pilih node mana untuk Pod baru (resource, affinity, taint) | k3s server |
| **kube-controller-manager** | loop kontrol: pastikan replica sesuai, node sehat, dsb. (Deployment controller, ReplicaSet, Node controller) | k3s server |
| **cloud-controller-manager** | integrasi cloud (on-prem umumnya tidak aktif) | — |

**Inti:** `kubectl get pods` tidak "melihat Pod di node" — ia bertanya ke **API server**, yang membaca dari **etcd**. Kalau etcd hilang, cluster kehilangan ingatan (itulah kenapa backup etcd penting — Modul 2.4).

## 3. Komponen di Node (Worker)

| Komponen | Fungsi |
|---|---|
| **kubelet** | agen di tiap node; pantau Pod, pastikan container jalan sesuai spec, laporkan status ke API server |
| **kube-proxy** | aturan jaringan (iptables/IPVS) untuk Service: route traffic ke Pod |
| **container runtime** | yang benar-benar menjalankan container (containerd; k3s pakai containerd bawaan) |

**Alur kerja kubelet:** API server bilang "Pod X harus jalan di node ini" → kubelet terjemahkan ke container runtime → `containerd` start container → kubelet pantau & laporkan status (Running/CrashLoopBackOff).

```bash
# Di k3d node = container; lihat kubelet & containerd
docker exec k3d-lab-server-0 ps aux 2>/dev/null | grep -E "kubelet|containerd" | head
```

## 4. etcd — Sumber Kebenaran

etcd = database key-value konsisten. Semua objek K8s (Pod, Service, Secret, ...) disimpan di sini. **Tanpa etcd, tidak ada cluster** (control plane tidak tahu apa yang harus ada).

```
 kubectl apply -f pod.yaml
   │
   ▼
 kube-apiserver ── tulis ──▶ etcd   (state diinginkan: "3 replica app")
                                   │
                kubelet baca ◀──────┘  (pastikan 3 Pod jalan di node)
```

- k3s **embed** etcd (jalankan di server) — tanpa setup terpisah. k3d memakai SQLite default (1 server) atau etcd (`--servers 3`).
- **Kenapa backup etcd penting:** kalau etcd rusak dan tidak ada backup, seluruh definisi cluster hilang (Pod masih jalan, tapi "ingatan" hilang). Backup = snapshot etcd (Modul 2.4).

## 5. Jalur Lengkap: `kubectl` → Pod Jalan

```
 Anda: kubectl apply -f deployment.yaml (ingin 3 replica)
   │
   ▼  (HTTPS, kubeconfig)
 kube-apiserver ── validasi + tulis ──▶ etcd
   │
   ▼  (watch)
 kube-controller-manager (Deployment controller) ── buat ReplicaSet ──▶ 3 Pod
   │
   ▼  (watch)
 kube-scheduler ── pilih node tiap Pod ──▶ tulis nodeName ke etcd
   │
   ▼  (watch di node)
 kubelet (di node terpilih) ── terjemah Pod spec ──▶ containerd ──▶ container jalan
   │
   ▼
 kubelet laporkan status Pod ──▶ API server ──▶ etcd (Running)
```

Ini menjelaskan kenapa ada **delay** antara `kubectl apply` dan Pod `Running` — informasi mengalir lewat banyak komponen (watch-based, bukan synchronous).

## 6. k3s & k3d — Bagaimana Mereka Menyederhanakan?

| Komponen standar K8s | k3s | k3d |
|---|---|---|
| kube-apiserver, etcd, scheduler, controller | 1 proses `k3s server` (gabungan) | 1 container k3s server |
| kubelet, kube-proxy, containerd | 1 proses `k3s agent`/server | 1 container k3s agent |
| Cloud provider integration | dihapus (on-prem) | dihapus |
| etcd | SQLite default / etcd embedded | SQLite default / etcd (`--servers 3`) |

k3s **menggabungkan** semua komponen control plane jadi satu biner ringan. Itu sebabnya k3s cocok on-prem/IoT: install 1 biner, jadi cluster. k3d membungkus k3s sebagai container agar cepat berulang di laptop.

```bash
# Bukti: k3s server gabungkan komponen
docker exec k3d-lab-server-0 k3s --version 2>/dev/null
kubectl get componentstatuses 2>/dev/null || kubectl get --raw='/readyz?verbose' | head
```

## 7. High Availability: Kapan Butuh Banyak Control Plane?

- **Lab/harian:** 1 server (control plane + worker) cukup — etcd = SQLite. k3d default.
- **Simulasi production:** 3 server (etcd clustered) — HA control plane. k3s `--cluster-init` + join 2 server lagi (Modul 2.2).
- **Trade-off:** HA = kompleks, butuh 3 VM, resource besar. Tapi control plane tidak jadi single point of failure.

## Latihan Cepat (15 menit)

```bash
# 1. Buat cluster k3d 1 server + 2 agent (atau pakai yang ada)
k3d cluster delete lab 2>/dev/null
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
kubectl get nodes -o wide

# 2. Lihat komponen control plane (Pod di kube-system)
kubectl get pods -n kube-system
kubectl get pods -n kube-system -o wide   # di node mana CoreDNS/Traefik jalan?

# 3. API server & etcd (di k3d node = container)
docker exec k3d-lab-server-0 ps aux 2>/dev/null | grep -E "apiserver|etcd|kubelet|containerd" | grep -v grep | head

# 4. Jalur: apply Pod, amati muncul
kubectl run tmp --image=alpine -- sleep 3600
kubectl get pod tmp -o wide   # node mana?
kubectl delete pod tmp
```

## Ringkasan

| Mau paham... | Lihat di... |
|---|---|
| Apa yang simpan state cluster | **etcd** (sumber kebenaran; backup penting) |
| Yang menerima kubectl | **kube-apiserver** |
| Yang pilih node untuk Pod | **kube-scheduler** |
| Yang pastikan replica benar | **kube-controller-manager** (Deployment controller) |
| Agen di node yang jalan container | **kubelet** + container runtime (containerd) |
| Aturan jaringan Service | **kube-proxy** |
| Kenapa k3s ringan | semua komponen digabung 1 biner |

## Cek Pemahaman

1. Saat `kubectl get pods`, dari mana informasi itu datang? (komponen apa yang dibaca)
2. Apa yang terjadi pada cluster jika etcd hilang & tidak ada backup?
3. Jelaskan peran kubelet vs container runtime (containerd) — siapa perintah siapa?
4. Kenapa ada delay antara `kubectl apply` dan Pod `Running`? (sebut 2 langkah di jalur)
5. Di k3s, komponen mana yang "dihapus" karena tidak relevan on-prem? (sebut 1)