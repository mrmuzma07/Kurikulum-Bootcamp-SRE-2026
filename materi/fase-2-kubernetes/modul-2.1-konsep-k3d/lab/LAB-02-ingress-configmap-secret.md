# LAB-02 — Ingress, ConfigMap & Secret

> **Target:** mengekspos dua service via satu Ingress (host/path), memasang konfigurasi via ConfigMap, menyuntik kredensial via Secret, dan melakukan rollout update versi — semuanya deklaratif.

## Latar Belakang
LAB-01 fokus Pod/Deployment/Service (Layer 4). Sekarang tambahkan **Layer 7 (Ingress)**, **config eksternal (ConfigMap)**, dan **data sensitif (Secret)** — tiga objek yang membuat deployment "nyata": app baca config tanpa rebuild image, password tidak di-hardcode, dan banyak service di satu IP via host header. Ini pola yang dipakai Helm (Fase 5) & ArgoCD (Fase 6).

## Prasyarat
- [ ] LAB-01 selesai (deployment.yaml + service.yaml ada; paham Pod/Deployment/Service)
- [ ] Cluster k3d berjalan (`k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer`)
- [ ] Image `app:v1.0.0` & `app:v1.1.0` di GitLab registry (atau build v1.1.0 sekarang; lihat Modul 1.1 LAB-01)

## Waktu
± 90 menit

## Langkah

### 1. Siapkan Cluster & Namespace

```bash
k3d cluster delete lab 2>/dev/null
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
kubectl create ns demo
kubectl config set-context --current --namespace=demo
k3d kubeconfig merge lab --switch
```

### 2. ConfigMap — Konfigurasi Tanpa Rebuild

`configmap.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "lab02"
  LOG_LEVEL: "info"
  ENABLE_FEATURE_X: "true"
  greeting.yaml: |
    message: "hello from configmap"
    version: v1
```

```bash
kubectl apply -f configmap.yaml
kubectl get cm app-config
kubectl describe cm app-config
```

### 3. Secret — Kredensial (Jangan Commit Isinya!)

`secret.yaml` — **perhatian: ini contoh lab. Di production, jangan commit secret plain-text** (pakai SOPS/Sealed-Secret/Vault — Fase 6):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  API_TOKEN: "lab-token-abc123"
  DB_PASSWORD: "s3cr3t!"
```

```bash
kubectl apply -f secret.yaml
kubectl get secret app-secret
kubectl get secret app-secret -o jsonpath='{.data.API_TOKEN}' | base64 -d; echo
```

### 4. Deployment — Pakai ConfigMap + Secret

`deployment.yaml` (ganti `<user>`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector: {matchLabels: {app: app}}
  template:
    metadata:
      labels: {app: app}
    spec:
      containers:
      - name: app
        image: registry.gitlab.com/<user>/sre-bootcamp/app:v1.0.0
        ports: [{containerPort: 8080}]
        envFrom:
        - configMapRef: {name: app-config}        # env dari ConfigMap
        env:
        - name: API_TOKEN
          valueFrom:
            secretKeyRef: {name: app-secret, key: API_TOKEN}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef: {name: app-secret, key: DB_PASSWORD}
        volumeMounts:
        - {name: cfg-file, mountPath: /etc/app, readOnly: true}
        readinessProbe:
          httpGet: {path: /health, port: 8080}
          periodSeconds: 5
      volumes:
      - name: cfg-file
        configMap: {name: app-config}
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deploy/app
kubectl exec deploy/app -- env | grep -E "APP_NAME|API_TOKEN|LOG_LEVEL"
kubectl exec deploy/app -- cat /etc/app/greeting.yaml
```

**Verifikasi:** app baca config dari ConfigMap (env + file) & Secret (env) — tanpa ubah image.

### 5. Service + Ingress (Dua Service, Satu Ingress)

Service untuk app:
```yaml
apiVersion: v1
kind: Service
metadata: {name: app}
spec:
  selector: {app: app}
  ports: [{port: 80, targetPort: 8080}]
```

Deploy service "api" kedua (app sama, label berbeda — simulasi microservice):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: api}
spec:
  replicas: 2
  selector: {matchLabels: {app: api}}
  template:
    metadata: {labels: {app: api}}
    spec:
      containers:
      - name: api
        image: registry.gitlab.com/<user>/sre-bootcamp/app:v1.0.0
        ports: [{containerPort: 8080}]
        env:
        - {name: APP_NAME, value: "api-svc"}
        readinessProbe: {httpGet: {path: /health, port: 8080}, periodSeconds: 5}
---
apiVersion: v1
kind: Service
metadata: {name: api}
spec:
  selector: {app: api}
  ports: [{port: 80, targetPort: 8080}]
```

`ingress.yaml` — routing berbasis host:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gateway
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
  - host: api.k3d.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: {name: api, port: {number: 80}}
```

```bash
echo "127.0.0.1 app.k3d.local api.k3d.local" | sudo tee -a /etc/hosts
kubectl apply -f deployment.yaml      # app + api
kubectl apply -f service.yaml -f api-deploy.yaml
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl get svc,deploy

# Akses dari Mac (8080 → LB → Traefik → Service)
curl -H 'Host: app.k3d.local' http://localhost:8080/        # "hello from lab02 ..."
curl -H 'Host: api.k3d.local' http://localhost:8080/        # "hello from api-svc ..."
curl -H 'Host: app.k3d.local' http://localhost:8080/health  # "ok"
```

**Catat:** satu IP (Mac:8080) melayani dua service berbeda via host header — inilah keunggulan Ingress.

### 6. Rollout Update (Versi Baru)

Build & push `app:v1.1.0` dulu (atau pakai yang sudah ada; lihat Modul 1.1 LAB-01):
```bash
# (di direktori app dari Modul 1.1)
docker buildx build --platform linux/arm64,linux/amd64 \
  -t registry.gitlab.com/<user>/sre-bootcamp/app:v1.1.0 --push .
```

Update Deployment app ke v1.1.0:
```bash
kubectl set image deploy/app app=registry.gitlab.com/<user>/sre-bootcamp/app:v1.1.0
kubectl rollout status deploy/app
kubectl rollout history deploy/app
```

Amati rolling update (Pod baru dibuat bertahap, Pod lama mati):
```bash
kubectl get pod -l app=app -o wide -w      # Ctrl+C setelah selesai
curl -H 'Host: app.k3d.local' http://localhost:8080/   # versi baru?
```

**Rollback** (simulasi v1.1.0 bermasalah):
```bash
kubectl rollout undo deploy/app
kubectl rollout status deploy/app
curl -H 'Host: app.k3d.local' http://localhost:8080/   # kembali v1.0.0
```

### 7. Ubah ConfigMap & Amati (Tanpa Rebuild Image)

```bash
# Edit configmap.yaml: APP_NAME: "lab02-v2", LOG_LEVEL: "debug"
kubectl apply -f configmap.yaml
# ConfigMap update tidak otomatis restart Pod. Restart Deployment:
kubectl rollout restart deploy/app
kubectl rollout status deploy/app
kubectl exec deploy/app -- env | grep APP_NAME   # "lab02-v2"
curl -H 'Host: app.k3d.local' http://localhost:8080/   # "hello from lab02-v2 ..."
```

**Pelajaran:** ConfigMap di-mount sebagai env **tidak auto-reload** (Pod harus restart). ConfigMap di-mount sebagai **file** (volumes) auto-reload oleh kubelet (tapi app harus watch file). Pilih sesuai kebutuhan.

### 8. Bersihkan & Commit

```bash
k3d cluster delete lab
docker system prune -f
sudo sed -i '' '/k3d.local/d' /etc/hosts

cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
git switch -c m2.1-lab02
mkdir -p m2.1/lab02
# pindahkan semua YAML ke m2.1/lab02/
git add m2.1/lab02/
git commit -m "feat(m2.1): ingress multi-host + configmap + secret + rollout

- ingress: app.k3d.local & api.k3d.local via satu IP (traefik)
- configmap (env+file) + secret (env) tanpa rebuild image
- rolling update v1.0.0 → v1.1.0 + rollback
- restart untuk reload configmap env

Closes #<issue-m2.1-lab02>"
git push -u origin m2.1-lab02
```
Buat MR → squash & merge.

## Acceptance Criteria

- [ ] ConfigMap `app-config` ter-apply; app baca `APP_NAME` & `greeting.yaml` dari ConfigMap
- [ ] Secret `app-secret` ter-apply; `API_TOKEN` & `DB_PASSWORD` jadi env di Pod (tidak di-hardcode di image)
- [ ] Dua Deployment (`app` + `api`) jalan; dua Service
- [ ] Ingress `gateway` route `app.k3d.local` → app & `api.k3d.local` → api (satu IP:8080)
- [ ] `curl -H 'Host: ...'` balas berbeda untuk app vs api
- [ ] Rollout update v1.0.0 → v1.1.0 berhasil; rollback kembali ke v1.0.0
- [ ] Ubah ConfigMap + `rollout restart` → app baca config baru
- [ ] Semua manifest YAML ter-commit via MR (SECRET: jangan commit `secret.yaml` berisi plain-text — pakai `.gitignore` atau commit template kosong)

## Troubleshooting

| Gejala | Solusi |
|---|---|
| Ingress 404 / default backend | host header salah; `curl -H 'Host: app.k3d.local'`; cek `kubectl get ingress` |
| `curl` connection refused 8080 | port forward `--port 8080:80@loadbalancer` lupa; buat ulang cluster |
| Service endpoints kosong | label selector Service ≠ Pod; `kubectl get pod --show-labels` |
| ConfigMap env tidak berubah setelah apply | ConfigMap env tidak auto-reload; `kubectl rollout restart deploy/app` |
| Secret `base64 -d` salah | cek `stringData` vs `data`; `stringData` auto-encode, `data` harus base64 manual |
| Rollout stuck (Pod baru tidak Ready) | `kubectl rollout status`; cek readinessProbe; `kubectl describe pod` |
| `ImagePullBackOff` setelah `set image` | tag v1.1.0 tidak ada di registry; build & push dulu |
| `k3d.local` tidak resolve | `/etc/hosts` belum ditambah; `sudo tee -a /etc/hosts` |

## Catatan SRE
- **Ingress = virtual hosting.** Banyak service, satu IP. Hemat LoadBalancer di on-prem (Modul 2.3). Tanpa Ingress, tiap service butuh nodePort/MetalLB sendiri.
- **ConfigMap/Secret = pisahkan config & code.** Image tidak rebuild saat ganti config/credential — hanya Pod restart. Ini prinsip twelve-factor.
- **Secret bukan keamanan penuh.** base64 ≠ encryption. Di GitOps (Fase 6): Sealed-Secret (enkripsi di Git, dekrip di cluster) atau SOPS/Vault. Untuk lab, jangan commit `secret.yaml` berisi nilai asli.
- **RollingUpdate + readinessProbe = zero-downtime.** Tanpa readinessProbe benar, traffic bisa ke Pod belum siap → error. Uji dengan `curl` berulang saat rollout.

## Lanjut
[Evaluasi: Latihan & Kuis](../evaluasi/latihan.md)