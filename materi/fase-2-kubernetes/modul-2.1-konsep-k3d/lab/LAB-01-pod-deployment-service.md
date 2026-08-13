# LAB-01 — Pod, Deployment & Service di k3d

> **Target:** men-deploy aplikasi multi-replica dari GitLab registry, mengekspos via Service ClusterIP, menskalakan, dan merasakan self-healing (Pod hilang → Deployment buat baru) — semua via manifest YAML deklaratif.

## Latar Belakang
Modul 1.2 LAB-01 sudah membuat Anda `kubectl create deployment` (imperatif) & Ingress cepat. Sekarang kita **tuliskan manifest YAML** (deklaratif, Git-ready) untuk Pod, Deployment, Service — fondasi yang dipakai semua fase berikutnya. Inilah pola kerja SRE: tulis YAML → apply → amati → break → perbaiki.

## Prasyarat
- [ ] Fase 1 selesai: image `registry.gitlab.com/<user>/sre-bootcamp/app:v1.0.0` (multi-arch)
- [ ] k3d & kubectl terpasang
- [ ] Repo `sre-bootcamp` di GitLab

## Waktu
± 75 menit

## Langkah

### 1. Cluster k3d Multi-Node

```bash
k3d cluster delete lab 2>/dev/null
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
kubectl get nodes -o wide
k3d kubeconfig merge lab --switch
```

Pastikan 1 server + 2 agent Ready.

### 2. Namespace

```bash
kubectl create ns demo
kubectl config set-context --current --namespace=demo
```

### 3. Deployment (Manifest YAML)

Buat direktori & file di repo:
```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
mkdir -p m2.1/lab01
cd m2.1/lab01
```

`deployment.yaml` (ganti `<user>` dengan username GitLab Anda):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels: {app: app, env: lab}
spec:
  replicas: 3
  selector:
    matchLabels: {app: app}
  strategy:
    type: RollingUpdate
    rollingUpdate: {maxUnavailable: 1, maxSurge: 1}
  template:
    metadata:
      labels: {app: app, env: lab}
    spec:
      containers:
      - name: app
        image: registry.gitlab.com/<user>/sre-bootcamp/app:v1.0.0
        ports: [{containerPort: 8080}]
        env:
        - {name: APP_NAME, value: "lab01"}
        resources:
          requests: {cpu: 50m, memory: 64Mi}
          limits: {cpu: 200m, memory: 128Mi}
        readinessProbe:
          httpGet: {path: /health, port: 8080}
          periodSeconds: 5
          initialDelaySeconds: 2
        livenessProbe:
          httpGet: {path: /health, port: 8080}
          periodSeconds: 10
          initialDelaySeconds: 5
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deploy/app
kubectl get deploy,rs,pod -l app=app
kubectl get pod -o wide                 # amati Pod tersebar di server + 2 agent
```

**Kalau private registry** (repo tidak publik): buat pull secret dulu & tambahkan `imagePullSecrets` di `template.spec` (lihat topik 02 / Modul 1.2 LAB-01):
```bash
kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.com \
  --docker-username=<gitlab-user> \
  --docker-password=<PAT-read_registry>
```
```yaml
    spec:
      imagePullSecrets:
      - name: gitlab-registry
      containers:
      ...
```

### 4. Service (ClusterIP)

`service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector: {app: app}
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

```bash
kubectl apply -f service.yaml
kubectl get svc app
kubectl get endpoints app              # Pod IP yang dilayani Service
```

Verifikasi resolve DNS internal:
```bash
kubectl run tmp --rm -it --restart=Never --image=alpine -- wget -qO- http://app/
# "hello from lab01 ..."
kubectl run tmp --rm -it --restart=Never --image=alpine -- wget -qO- http://app/health
# "ok"
```

### 5. Skala & Amati Distribusi

```bash
kubectl scale deploy app --replicas=6
kubectl get pod -o wide -l app=app
# Amati Pod tersebar; ada node dapat lebih dari 1 Pod?
```

Ubah replika via manifest (deklaratif — edit `replicas: 4` lalu apply):
```bash
# Edit deployment.yaml: replicas: 4
kubectl apply -f deployment.yaml
kubectl get pod -l app=app
```

### 6. Self-Healing: Hapus Pod

```bash
kubectl delete pod -l app=app --field-selector=status.phase=Running | head -1
# atau spesifik:
kubectl get pod -l app=app
kubectl delete pod <nama-pod>
kubectl get pod -l app=app              # Deployment langsung buat Pod baru (replica kembali 4)
```

**Catat:** berapa detik sampai replica kembali? Inilah "desired state reconciliation" — controller menjaga replica.

### 7. Simulasi Chaos: Stop 1 Agent

```bash
docker stop k3d-lab-agent-0
sleep 15
kubectl get nodes                        # 1 NotReady
kubectl get pod -o wide -l app=app        # Pod di node itu → Terminating/Pending → reschedule
sleep 10
kubectl get pod -o wide -l app=app
docker start k3d-lab-agent-0
sleep 10
kubectl get nodes
```

**Catat:** berapa Pod Running sebelum/saat/ setelah? Berapa detik pulih?

### 8. Port-Forward (Akses Debug Tanpa Ingress)

```bash
kubectl port-forward svc/app 9090:80 &
sleep 2
curl -s http://localhost:9090/
kill %1 2>/dev/null
```

### 9. Bersihkan & Commit

```bash
# Bersihkan (simpan file YAML!)
k3d cluster delete lab
docker system prune -f

# Commit manifest ke repo
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
git switch -c m2.1-lab01
git add m2.1/lab01/
git commit -m "feat(m2.1): deployment + service multi-replica di k3d

- deployment 3 replica (rollingUpdate, probes, resource limits)
- service ClusterIP + endpoints verifikasi
- self-healing: hapus pod, stop agent, amati reschedule
- image dari gitlab registry (fase 1)

Closes #<issue-m2.1-lab01>"
git push -u origin m2.1-lab01
```
Buat MR → squash & merge.

## Acceptance Criteria

- [ ] Cluster k3d 1 server + 2 agent, namespace `demo`, context aktif di `demo`
- [ ] Deployment `app` 3 replica Running, tersebar di ≥2 node
- [ ] Service `app` ClusterIP, endpoints sesuai Pod IP
- [ ] `wget -qO- http://app/` dari Pod alpine balas hello (DNS internal jalan)
- [ ] Skala ke 6 lalu 4 via manifest; replica sesuai
- [ ] Hapus 1 Pod → Deployment buat baru (self-healing); catat waktu
- [ ] `docker stop` 1 agent → Pod reschedule, lalu pulih saat `start`
- [ ] `port-forward` 9090:80 bisa di-curl dari Mac
- [ ] `deployment.yaml` + `service.yaml` ter-commit via MR

## Troubleshooting

| Gejala | Solusi |
|---|---|
| Pod `ImagePullBackOff` | image tag/path salah; kalau private registry cek `imagePullSecrets` + PAT scope `read_registry` |
| Pod `Pending` | resource request terlalu besar untuk node; turunkan `requests` atau tambah agent |
| Pod `CrashLoopBackOff` | lihat `kubectl logs <pod> --previous`; env salah? app error di startup? |
| Service `endpoints` kosong | selector Service ≠ label Pod; `kubectl get pod --show-labels` bandingkan |
| `wget app` dari alpine gagal | Service tidak ada atau salah port; cek `kubectl get svc app` |
| Pod semua di 1 node | normal untuk cluster kecil tanpa anti-affinity; buat `podAntiAffinity` (topik 02 lanjut) jika perlu |
| Node `NotReady` lama setelah `docker stop` | tunggu 30–60s; `kubectl describe node`; taint `node.kubernetes.io/not-ready` |

## Catatan SRE
- **Deklaratif > imperatif.** `kubectl apply -f deployment.yaml` (manifest di Git) = reproducible; `kubectl create deployment` (imperatif) = cepat tapi hilang. GitOps (Fase 6) bekerja pada manifest seperti ini.
- **Resource requests/limits** di sini = cgroup level orkestrator (lanjutan Modul 1.1). Tanpa requests, scheduler tidak tahu node mana yang muat; tanpa limits, satu Pod bisa habiskan node.
- **Self-healing** adalah inti K8s: controller menjaga desired state. SRE tidak "restart Pod manual" — biarkan controller.
- **Selector/label** = "bahasa" yang menghubungkan Service ke Pod. Kesalahan label = Service tidak menemukan Pod (endpoints kosong). Konsisten label `app`, `env`.

## Lanjut
[LAB-02: Ingress, ConfigMap & Secret](LAB-02-ingress-configmap-secret.md)