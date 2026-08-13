# 02 — Objek Inti Kubernetes

> Pod, Deployment, Service, Ingress, ConfigMap, Secret, PV/PVC — unit-unit yang Anda operasikan sehari-hari.

## Tujuan
- Bisa menjelaskan & membuat Pod, Deployment (replica, rollout)
- Bisa menjelaskan Service (ClusterIP, NodePort, LoadBalancer) & kapan pakai masing-masing
- Bisa membuat Ingress & memahami host/path routing
- Bisa memakai ConfigMap & Secret (env vs mount)
- Bisa menjelaskan PV/PVC & kenapa Pod butuh volume persisten
- Bisa menulis manifest YAML dari nol (deklaratif, version-controlled)

## 0. Filosofi: Deklaratif

Kubernetes = **deklaratif**: Anda tulis *apa yang diinginkan* (manifest YAML), bukan *cara mencapainya* (imperatif).

```yaml
# "Saya ingin 3 replica app:v1.0.0" — bukan "jalankan 3 container"
apiVersion: apps/v1
kind: Deployment
metadata: {name: app}
spec:
  replicas: 3
  selector: {matchLabels: {app: app}}
  template:
    metadata: {labels: {app: app}}
    spec:
      containers:
      - name: app
        image: registry.gitlab.com/user/sre-bootcamp/app:v1.0.0
```

Kontrol loop (controller) yang menjaga *desired state* = *actual state*. Anda `kubectl apply`, controller bekerja. Ini beda dengan `docker run` (imperatif). Manifest disimpan di repo (GitOps — Fase 6).

## 1. Pod — Unit Terkecil

**Pod** = satu atau beberapa container berbagi namespace (jaringan, storage) — satu IP, satu volume share. Bukan container tunggal: Pod = "tempat container hidup bersama".

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels: {app: web, tier: frontend}
spec:
  containers:
  - name: web
    image: nginx:alpine
    ports: [{containerPort: 80}]
    resources:
      requests: {cpu: 50m, memory: 64Mi}
      limits: {cpu: 200m, memory: 128Mi}
```

- **Kenapa Pod, bukan container langsung?** Pod memberi abstraksi: container bisa mati-restart di dalam Pod tanpa Pod hilang; Pod bisa punya >1 container (sidecar — Fase 5/7).
- **Label** = cara tag Pod agar dicari (selector). Ini inti bagaimana Service menemukan Pod.
- **resources requests/limits** = cgroup di level orkestrator (lanjutan Modul 1.1). Melampaui memory limit → `OOMKilled` (exit 137).

```bash
kubectl apply -f pod.yaml
kubectl get pod web -o wide
kubectl describe pod web          # events di bawah = riwayat masalah
kubectl logs web                  # stdout container
kubectl exec -it web -- sh        # masuk container
```

**Aturan:** jarang membuat Pod langsung di production. Pakai **Deployment** (yang kelola Pod replica & restart). Pod telanjang dipakai untuk debug/one-off.

## 2. Deployment — Mengelola Replica Pod

Deployment = kontrol Pod supaya selalu ada N replica, bisa rollout versi baru, bisa rollback. Di atasnya ada **ReplicaSet** (yang benar-benar jaga jumlah Pod) — Deployment mengelola ReplicaSet.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: app}
spec:
  replicas: 3
  selector: {matchLabels: {app: app}}       # Pod mana yang milikku
  strategy:
    type: RollingUpdate
    rollingUpdate: {maxUnavailable: 1, maxSurge: 1}
  template:                                  # cetakan Pod
    metadata: {labels: {app: app}}
    spec:
      containers:
      - name: app
        image: registry.gitlab.com/user/sre-bootcamp/app:v1.0.0
        ports: [{containerPort: 8080}]
        readinessProbe:
          httpGet: {path: /health, port: 8080}
          periodSeconds: 5
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/app
kubectl get deploy,rs,pod -l app=app         # Deployment → ReplicaSet → Pod
kubectl scale deployment app --replicas=6

# Rollout versi baru
kubectl set image deployment/app app=registry.gitlab.com/user/sre-bootcamp/app:v1.1.0
kubectl rollout status deployment/app
kubectl rollout history deployment/app       # riwayat revisi
kubectl rollout undo deployment/app          # rollback ke v1.0.0
```

**RollingUpdate** = ganti Pod bertahap (maxUnavailable/maxSurge) — zero downtime kalau readinessProbe benar. **Recreate** = matikan semua dulu, baru buat baru — ada downtime (jarang dipakai).

## 3. Service — Stabil di Atas Pod Berubah

Pod IP berubah-ubah (Pod mati/restart = IP baru). **Service** = IP + DNS stabil yang route ke Pod (via label selector). Client bicara ke Service, bukan Pod langsung.

```yaml
apiVersion: v1
kind: Service
metadata: {name: app}
spec:
  selector: {app: app}        # Pod mana yang dilayani
  ports:
  - port: 80                  # port Service
    targetPort: 8080          # port Pod/container
  type: ClusterIP             # default: hanya dalam cluster
```

Tiga `type`:

| type | IP | Akses | Kapan |
|---|---|---|---|
| **ClusterIP** (default) | IP internal cluster | hanya dari dalam cluster | app-to-app (db, internal API) |
| **NodePort** | ClusterIP + port di semua node (30000–32767) | dari luar via `nodeIP:nodePort` | debug, atau Ingress backend |
| **LoadBalancer** | minta IP eksternal dari provider/MetalLB | dari luar via IP publik/LB | ekspos service ke user; **on-prem = MetalLB** (Modul 2.3) |

```bash
kubectl apply -f service.yaml
kubectl get svc app
kubectl get endpoints app          # Pod IP yang dilayani Service
# Dalam cluster:
kubectl run tmp --rm -it --image=alpine -- wget -qO- http://app   # resolve via DNS
```

**DNS internal:** Service `app` di namespace `prod` → `app.prod.svc.cluster.local`. Pod dalam cluster bisa panggil `app` langsung (namespace sama). Ini ganti "nama service compose" (Modul 1.1) tapi di level cluster.

## 4. Ingress — HTTP Routing ke Service

Ingress = HTTP/HTTPS layer 7 router (host + path) ke Service. Di k3s/k3d, **Traefik** jadi Ingress controller bawaan.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: app.k3d.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: app, port: {number: 80}}
```

```bash
echo "127.0.0.1 app.k3d.local api.k3d.local" | sudo tee -a /etc/hosts
kubectl apply -f ingress.yaml
curl -H 'Host: app.k3d.local' http://localhost:8080/   # 8080 di-forward ke LB → Traefik
```

Ingress ideal untuk banyak service di satu IP/port (virtual hosting via host header): `app.k3d.local` → service app; `api.k3d.local` → service api. Tanpa Ingress, tiap service butuh NodePort/LoadBalancer sendiri.

## 5. ConfigMap — Konfigurasi Tanpa Image Rebuild

ConfigMap = pasang konfigurasi (env / file) tanpa ubah image. Pisahkan config dari code.

```yaml
apiVersion: v1
kind: ConfigMap
metadata: {name: app-config}
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

Dipakai dua cara:
```yaml
# (a) sebagai env
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef: {name: app-config}

# (b) sebagai file (mount)
spec:
  containers:
  - name: app
    volumeMounts:
    - {name: cfg, mountPath: /etc/app}
  volumes:
  - name: cfg
    configMap: {name: app-config}
```

```bash
kubectl apply -f configmap.yaml
kubectl exec deploy/app -- cat /etc/app/config.yaml
kubectl exec deploy/app -- env | grep APP_ENV
```

## 6. Secret — Sensitif (Tapi Bukan Keamanan Penuh)

Secret = seperti ConfigMap tapi untuk data sensitif (password, token, TLS cert). **Default base64-encoded, bukan encrypted** (encryption-at-rest harus diaktifkan terpisah).

```yaml
apiVersion: v1
kind: Secret
metadata: {name: db-cred}
type: Opaque
stringData:                # stringData = plain text (auto base64)
  DB_PASSWORD: "s3cr3t!"
```

```yaml
# Pakai sebagai env
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef: {name: db-cred, key: DB_PASSWORD}
```

```bash
kubectl apply -f secret.yaml
kubectl get secret db-cred -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```

**⚠️ Aturan SRE:** **jangan commit Secret ke Git** (base64 bukan encryption, siapa saja bisa decode). Solusi: Sealed-Secret / SOPS / Vault (Fase 6 GitOps) — enkripsi saat commit, dekrip di cluster. Untuk lab, pakai Secret tapi jangan push isinya ke repo publik.

## 7. PV/PVC — Storage Persisten

Pod ephemeral: mati = data hilang (sama seperti container). **PersistentVolume (PV)** = storage di cluster; **PersistentVolumeClaim (PVC)** = "permintaan" storage oleh Pod. k3s punya **local path provisioner** bawaan (auto-buat PV).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: db-data}
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 1Gi}}
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: db}
spec:
  replicas: 1
  selector: {matchLabels: {app: db}}
  template:
    metadata: {labels: {app: db}}
    spec:
      containers:
      - name: db
        image: postgres:16-alpine
        env:
        - {name: POSTGRES_PASSWORD, valueFrom: {secretKeyRef: {name: db-cred, key: DB_PASSWORD}}}
        volumeMounts:
        - {name: data, mountPath: /var/lib/postgresql/data}
      volumes:
      - name: data
        persistentVolumeClaim: {claimName: db-data}
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc,pv
kubectl delete pod -l app=db     # Pod mati, tapi data tetap (PVC persisten)
kubectl get pod -l app=db        # Pod baru, data sama
```

| Konsep container (Fase 1) | Padanan K8s |
|---|---|
| volume docker | PVC/PV (persisten, dikelola cluster) |
| bind mount | — (di K8s pakai PVC/hostPath, bukan bind Mac) |
| healthcheck compose | liveness/readiness probe |
| compose service name | Service DNS |

## 8. Label, Selector & Namespace — Organisasi

- **Label** = key/value di metadata Pod/Service/... (`app: web`, `tier: frontend`, `env: prod`).
- **Selector** = "cari objek dengan label ini" — Service pakai selector untuk menemukan Pod.
- **Namespace** = partisi cluster (logis, bukan security): `prod`, `staging`, `kube-system`.

```bash
kubectl get pod -l app=web,env=prod      # filter label
kubectl get pod -A                       # semua namespace
kubectl get pod -n prod                  # namespace spesifik
kubectl create ns prod
kubectl apply -f deploy.yaml -n prod
```

Namespace untuk: isolasi lingkungan (prod/staging), tim, atau lifecycle. **Bukan** untuk security isolation (pod di namespace berbeda masih bisa bicara kalau network policy tidak ada).

## Latihan Cepat (30 menit)

```bash
# 1. Cluster
k3d cluster delete lab 2>/dev/null
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer

# 2. Deployment + Service (imperatif dulu, cepat)
kubectl create deployment web --image=nginx:alpine --replicas=3
kubectl expose deployment web --port=80 --target-port=80
kubectl get deploy,svc,pod -l app=web

# 3. ConfigMap + Secret
kubectl create configmap cfg --from-literal=APP_ENV=lab
kubectl create secret generic sec --from-literal=TOKEN=abc123
kubectl get cm cfg -o yaml | head
kubectl get secret sec -o jsonpath='{.data.TOKEN}' | base64 -d; echo

# 4. PVC
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data}
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 100Mi}}
EOF
kubectl get pvc data
```

## Ringkasan

| Objek | Untuk | Ingat |
|---|---|---|
| Pod | unit terkecil (1+ container, 1 IP) | jarang langsung; pakai Deployment |
| Deployment | replica + rollout + rollback | jaga desired replica |
| Service | IP/DNS stabil ke Pod (label selector) | ClusterIP/NodePort/LoadBalancer |
| Ingress | HTTP routing (host/path) ke Service | Traefik bawaan k3s/k3d |
| ConfigMap | config (env/file) tanpa rebuild | pisahkan config dari image |
| Secret | data sensitif (base64, bukan encrypt) | **jangan commit**; pakai SOPS/Vault |
| PVC/PV | storage persisten | Pod mati ≠ data hilang |
| Label/Selector | temukan & filter objek | Service → Pod via selector |
| Namespace | partisi logis cluster | bukan security isolation |

## Cek Pemahaman

1. Kenapa pakai Deployment, bukan Pod langsung? (sebut 2 alasan)
2. Pod IP berubah saat restart. Bagaimana client tetap bisa menemukannya? (komponen apa)
3. Beda `type: ClusterIP` vs `NodePort` vs `LoadBalancer` — kapan pakai masing-masing? (on-prem, siapa yang beri IP untuk LoadBalancer?)
4. Kenapa Ingress lebih baik dari NodePort untuk banyak service HTTP?
5. Kenapa Secret (base64) **bukan** aman untuk commit ke Git? Solusinya apa?
6. Apa yang terjadi pada data Pod database saat Pod dihapus, jika pakai PVC vs tidak?