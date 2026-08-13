# LAB-01 — Cluster k3d di OrbStack + Deploy App

> **Target:** menjalankan cluster k3d multi-node di OrbStack, deploy aplikasi (image dari Modul 1.1), ekspos via Ingress, dan akses dari Mac — sekaligus membandingkan k3d (cepat) dengan persiapan k3s (Fase 2.2).

## Latar Belakang
Ini jembatan: Anda sudah bisa container (Fase 1.1) & kenal rumahnya OrbStack (modul ini). Sekarang jalankan **Kubernetes ringan** (k3d) untuk merasakan objek inti (Pod, Deployment, Service, Ingress) sebelum Fase 2 mendalami. k3d dipakai sebagai "fast lane" sepanjang bootcamp; lab ini menanam kebiasaan `k3d create → deploy → k3d delete`.

## Prasyarat
- [ ] Modul 1.1 selesai (image app di GitLab registry: `registry.gitlab.com/<user>/sre-bootcamp/app:v1.0.0`)
- [ ] OrbStack jalan, resource limit di-set (modul 1.2 topik 02)
- [ ] `k3d` & `kubectl` terpasang (`brew install k3d kubectl`)
- [ ] Repo `sre-bootcamp` di GitLab

## Waktu
± 90 menit

## Langkah

### 1. Persiapan Resource & Bersih-bersih

```bash
# Pastikan tidak ada cluster k3d lama (bersihkan dulu)
k3d cluster list
k3d cluster delete lab 2>/dev/null || true
docker system prune -f                      # ruang untuk image k3s
docker system df
```

Pastikan OrbStack memory limit cukup (≥4GB; cluster multi-node butuh ~2–3GB).

### 2. Buat Cluster Multi-Node

```bash
k3d cluster create lab \
  --servers 1 \
  --agents 2 \
  --port 8080:80@loadbalancer \
  --port 8443:443@loadbalancer

# Tunggu node Ready
kubectl get nodes -w
# Ctrl+C saat semua Ready
kubectl get nodes -o wide
```

Pastikan ada 1 `server` + 2 `agent`, semua `Ready`.

### 3. Verifikasi Komponen Bawaan k3s

```bash
kubectl get pods -A
# Harus ada: kube-system (CoreDNS, metrics-server, Traefik), dll
kubectl get svc -A | grep -E "traefik|kube-dns"
```

### 4. Deploy Aplikasi dari Registry (Modul 1.1)

```bash
REG=registry.gitlab.com/<username>/sre-bootcamp/app   # ganti!

# Pull secret agar k3d bisa pull dari GitLab private registry
# (kalau repo public, skip; kalau private, buat secret):
kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.com \
  --docker-username=<gitlab-username> \
  --docker-password=<PAT-with-read_registry>

# Deploy (manifest minimal; Fase 2 akan pakai YAML file)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels: {app: app}
  template:
    metadata:
      labels: {app: app}
    spec:
      imagePullSecrets:
      - name: gitlab-registry     # hapus baris ini kalau repo public
      containers:
      - name: app
        image: $REG:v1.0.0
        ports: [{containerPort: 8080}]
        env:
        - {name: APP_NAME, value: "k3d-lab"}
        readinessProbe:
          httpGet: {path: /health, port: 8080}
          initialDelaySeconds: 2
          periodSeconds: 5
EOF

kubectl rollout status deployment/app
kubectl get pod -o wide                   # lihat Pod tersebar di 3 node
```

**Catatan:** kalau image registry Anda **public** (visibility publik di GitLab), hapus bagian `imagePullSecrets` & secret. Kalau private, pakai secret di atas.

### 5. Service & Ingress

```bash
# Service ClusterIP
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector: {app: app}
  ports:
  - port: 80
    targetPort: 8080
EOF

# Ingress (Traefik bawaan)
echo "127.0.0.1 app.k3d.local" | sudo tee -a /etc/hosts
cat <<'EOF' | kubectl apply -f -
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
          service:
            name: app
            port: {number: 80}
EOF

# Akses dari Mac (port 8080 di-forward ke LB → Traefik)
curl -H 'Host: app.k3d.local' http://localhost:8080/
# "hello from k3d-lab (uptime: ...)"
curl -H 'Host: app.k3d.local' http://localhost:8080/health
# "ok"
```

### 6. Skala & Amati Distribusi

```bash
kubectl scale deployment app --replicas=6
kubectl get pod -o wide
# Amati Pod tersebar di server + 2 agent (anti-affinity default sekedarnya)
```

### 7. Simulasi Chaos: Matikan 1 Node

```bash
# Hentikan 1 agent (node worker)
docker ps | grep k3d-lab-agent
docker stop k3d-lab-agent-0
sleep 5
kubectl get nodes                     # 1 node NotReady
kubectl get pod -o wide                # Pod di node itu jadi Pending/Terminating

# k3s/k3d akan reschedule? (Deployment akan buat Pod baru di node sehat)
sleep 10
kubectl get pod -o wide | grep -c Running

# Pulihkan
docker start k3d-lab-agent-0
sleep 10
kubectl get nodes                     # kembali Ready
```

### 8. Bersihkan

```bash
k3d cluster delete lab
docker system prune -f
sudo sed -i '' '/app.k3d.local/d' /etc/hosts   # macOS
```

### 9. Catat & Commit

```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
git switch -c m1.2-lab01
mkdir -p m1.2/lab
cat > m1.2/lab/k3d-report.md <<'EOF'
# m1.2 LAB-01 — k3d Cluster Report

## Bukti
- Output `kubectl get nodes`:
  ```
  (tempel di sini)
  ```
- Output `kubectl get pod -o wide` (sebaran Pod):
  ```
  (tempel)
  ```
- `curl -H 'Host: app.k3d.local' http://localhost:8080/`:
  ```
  (tempel)
  ```
- Setelah `docker stop k3d-lab-agent-0`: berapa Pod Running lagi? berapa detik pulih?

## Catatan
- OrbStack memory limit: ... GB
- k3d vs k3s: kapan pakai masing-masing (dari topik 03)
- Pelajaran: (1 hal paling berguna)
EOF

git add m1.2/lab/k3d-report.md
git commit -m "feat(m1.2): lab k3d cluster multi-node + deploy app via ingress

- k3d cluster 1 server + 2 agents di OrbStack
- deploy app dari GitLab registry (Modul 1.1 image)
- Service + Ingress (Traefik), akses dari Mac via host
- simulasi chaos: stop 1 agent, amati reschedule

Closes #<issue-m1.2-lab01>"
git push -u origin m1.2-lab01
```

Buat MR → squash & merge.

## Acceptance Criteria

- [ ] Cluster k3d 1 server + 2 agent, semua `Ready` (`kubectl get nodes`)
- [ ] Pod `app` jalan & tersebar di node (≥2 node punya Pod)
- [ ] `/health` Ingress balas `ok` dari Mac (`curl`)
- [ ] `app.k3d.local` resolve via `/etc/hosts` & route lewat Traefik
- [ ] Simulasi `docker stop` 1 agent → Pod reschedule ke node sehat, lalu pulih saat `start`
- [ ] `k3d cluster delete lab` bersih (tidak nyangkut; `docker ps | grep k3d` kosong)
- [ ] `m1.2/lab/k3d-report.md` ter-commit via MR

## Troubleshooting

| Gejala | Solusi |
|---|---|
| Node `NotReady` lama | Tunggu 30–60s; cek `kubectl describe node`; resource OrbStack cukup? |
| Pod `ImagePullBackOff` | Image tag/path salah; kalau private registry, cek `imagePullSecrets` + PAT scope `read_registry` |
| `curl` connection refused di 8080 | Port forward `--port 8080:80@loadbalancer` lupa saat create; buat ulang cluster |
| Ingress 404 / default backend | host header salah; `curl -H 'Host: app.k3d.local'`; cek Ingress `kubectl get ingress` |
| `kubectl` context salah | `k3d kubeconfig merge lab --switch`; `kubectl config use-context k3d-lab` |
| Mac lag | OrbStack memory limit kecil; kurangi `--agents`; `k3d cluster delete` yang tidak terpakai |
| Chaos test: Pod tidak reschedule | Tunggu lebih lama; cek `kubectl describe pod`; node `NotReady` perlu waktu agar di-taint |

## Catatan SRE
- **k3d = fast lane.** Fase 2.1 akan pakai k3d intensif untuk belajar objek Kubernetes. Sekarang Anda sudah pernah merasakannya.
- **k3s di VM = production lane.** Fase 2.2 akan install k3s di VM OrbStack (systemd, static IP) — lebih berat tapi seperti server sungguhan, dan MetalLB L2 bisa jalan di sana.
- Kebiasaan `k3d create → delete` menjaga laptop tetap bersih. Jangan biarkan cluster tertinggal jalan.
- Ingress + host header di sini = pola yang sama dipakai di Fase 2 & GitOps (ArgoCD deploy Ingress).

## Lanjut
[Evaluasi: Latihan & Kuis](../evaluasi/latihan.md)

---

**Fase 1 (Container & OrbStack) selesai!** Lanjut ke [Fase 2 — Kubernetes](../../fase-2-kubernetes/README.md) (menyusul).