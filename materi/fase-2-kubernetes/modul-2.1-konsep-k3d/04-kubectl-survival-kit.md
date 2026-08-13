# 04 — kubectl Survival Kit

> Perintah yang Anda pakai setiap hari sebagai SRE — dan debug flow saat Pod tidak jalan.

## Tujuan
- Bisa memakai kubectl survival kit: get, describe, logs, exec, port-forward, top
- Bisa memilih output format (`-o wide`, `-o yaml`, `-o jsonpath`, `-o name`)
- Bisa memakai label selector & namespace untuk filter
- Bisa menjalankan debug flow: describe → events → logs → exec → resolve
- Bisa menjelaskan exit code & status Pod umum (Running, CrashLoopBackOff, ImagePullBackOff, Pending)

## 1. Survival Kit — Get, Describe, Logs, Exec

```bash
# GET — lihat objek
kubectl get pods                      # semua Pod di namespace aktif
kubectl get pods -A                   # semua namespace
kubectl get pods -o wide              # + node, IP
kubectl get pods -o yaml             # full spec/status (debug mendalam)
kubectl get pods -o name             # hanya nama (untuk script)
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -l app=web          # filter label
kubectl get pods -n prod             # namespace spesifik
kubectl get all                      # semua objek utama

# DESCRIBE — cerita lengkap (+ events: riwayat masalah)
kubectl describe pod <name>          # lihat bagian Events di bawah!
kubectl describe node <name>         # resource, taint, kondisi

# LOGS — output container
kubectl logs <pod>                   # stdout
kubectl logs <pod> -c <container>     # multi-container Pod
kubectl logs <pod> --previous        # log container yang crash sebelumnya
kubectl logs -f deploy/app            # follow, semua replica
kubectl logs --since=10m deploy/app  # filter waktu

# EXEC — masuk container
kubectl exec -it <pod> -- sh
kubectl exec -it <pod> -c <container> -- sh
kubectl exec <pod> -- env            # one-off, non-interactive

# PORT-FORWARD — akses lokal ke Service/Pod (debug, tanpa Ingress)
kubectl port-forward svc/app 9090:80 # Mac:9090 → Service:80
kubectl port-forward pod/<name> 9090:8080
# (Ctrl+C untuk stop)
```

## 2. Output Format (`-o`)

| Format | Untuk |
|---|---|
| (default) | tabel ringkas |
| `-o wide` | + kolom tambahan (node, IP) |
| `-o yaml` | spec/status penuh (debug, salin manifest) |
| `-o jsonpath=...` | ekstrak field spesifik (script, CI) |
| `-o name` | hanya nama objek |
| `-o json` | JSON penuh (pipe ke `jq`) |

```bash
# jsonpath sering dipakai untuk automation/cheatsheet
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
```

## 3. Label, Selector & Namespace

```bash
# Label di metadata: app=web, env=prod, tier=frontend
kubectl get pods -l app=web                 # equality
kubectl get pods -l 'app in (web,api)'       # set-based
kubectl get pods -l app=web,env=prod        # gabungan (AND)
kubectl get pods --show-labels              # lihat semua label Pod

# Namespace
kubectl get pods -A                         # semua namespace
kubectl get pods -n kube-system
kubectl config set-context --current --namespace=prod   # ganti ns default context
```

**Pattern SRE:** tag konsisten (`app`, `env`, `tier`, `version`) — Service & debug bergantung label benar. Helm (Fase 5) auto-label release.

## 4. Apply, Delete, Scale, Rollout

```bash
# Deklaratif (utama)
kubectl apply -f deploy.yaml               # create/update
kubectl apply -f manifests/               # direktori (semua YAML)
kubectl apply -k overlays/prod             # kustomize (Fase 5)

# Imperatif (cepat, tapi kurang reproducible)
kubectl create deployment web --image=nginx:alpine
kubectl scale deployment app --replicas=5
kubectl delete pod <name>                  # Pod dihapus, Deployment buat baru
kubectl delete -f deploy.yaml              # hapus objek dari manifest

# Rollout
kubectl rollout status deployment/app
kubectl rollout history deployment/app
kubectl rollout undo deployment/app
kubectl rollout undo deployment/app --to-revision=2
kubectl restart deployment/app             # mulai ulang Pod (pull image baru)
```

**Aturan SRE:** `apply` (deklaratif) > imperatif. Manifest di Git = reproducible (GitOps — Fase 6). Imperatif (`create deployment`) cepat tapi hilang dari ingatan. Pakai imperatif hanya untuk eksplorasi/debug, lalu tulis YAML.

## 5. Resource & Top

```bash
kubectl top nodes                          # CPU/mem per node (butuh metrics-server; k3s punya)
kubectl top pods                           # per Pod di ns aktif
kubectl top pods -A                        # semua namespace
kubectl describe node <name> | grep -A5 Allocatable   # resource kapasitas node
```

## 6. Debug Flow — Saat Pod Tidak Jalan

Urutan investigasi yang konsisten menemukan akar masalah:

```
 1. kubectl get pod <name>            → status (Pending? CrashLoopBackOff? ImagePullBackOff?)
 2. kubectl describe pod <name>      → Events di bawah (scheduled? image pull fail? OOMKill?)
 3. kubectl logs <name>               → output app (error runtime)
 4. kubectl logs <name> --previous    → log sebelum crash
 5. kubectl exec -it <name> -- sh     → masuk, cek filesystem/env/koneksi
 6. resolve + kubectl apply lagi
```

### Status Pod Umum & Akar Masalah

| Status | Gejala | Kemungkinan penyebab |
|---|---|---|
| **Pending** | Pod tidak jalan, tidak ada node | resource request terlalu besar; taint/toleration; node penuh; PVC menunggu |
| **ImagePullBackOff** | container tidak start | image/tag salah; registry private tanpa `imagePullSecret`; arch mismatch (amd64-only di arm64) |
| **CrashLoopBackOff** | container start lalu crash, berulang | app error di startup; env/config salah; livenessProbe gagal |
| **OOMKilled** (exit 137) | container mati | memory limit terlalu kecil / app memory leak |
| **Error** (exit non-0) | container exit | app crash code; lihat `logs --previous` |
| **RunContainerError** | container tidak start | image arch salah (`exec format error`); biner hilang |

```bash
kubectl get pod <name>
# NAME   READY   STATUS             RESTARTS   AGE
# app-0  0/1     CrashLoopBackOff    7          9m

kubectl describe pod app-0 | tail -20    # Events: "Back-off restarting failed container"
kubectl logs app-0 --previous            # error runtime (mis. "cannot connect to db")
```

### Exit Code Cepat

| Code | Arti |
|---|---|
| 0 | selesai normal |
| 1 | error aplikasi umum |
| 137 | **OOMKilled** (SIGKILL, 128+9) — memory limit |
| 139 | SIGSEGV (segfault) — 128+11 |
| 126 | command tidak executable |

## 7. Ephemeral Debug Container (Trik SRE)

Pod production mungkin tidak punya shell (distroless — Modul 1.1). Pakai **debug container** ephemeral:

```bash
kubectl debug -it <pod> --image=alpine --target=<container>
# atau copy Pod (jaga yang asli):
kubectl debug -it <pod> --copy-to=app-debug --image=busybox --share-processes -- /bin/sh
```

Berguna untuk memeriksa Pod distroless tanpa ubah image. Fase 2.4 mendalami ini.

## 8. Aliases & Cheatsheet Pribadi

Buat `m2.1/kubectl-notes.md` di repo — cheatsheet pribadi (muscle memory):

```bash
# Aliases di ~/.zshrc (opsional, hemat ketik)
alias k=kubectl
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kx='kubectl exec -it'
complete -o default -F __start_kubectl k   # completion (brew install kubectl completions)
```

```bash
# Lihat semua objek di namespace
kubectl get all,cm,secret,pvc,ing -n prod

# Cek apa yang dipakai Service (endpoints)
kubectl get endpoints app

# Cek image yang jalan (apakah update?)
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
```

## Latihan Cepat (25 menit)

```bash
# 1. Deploy & debug flow
k3d cluster delete lab 2>/dev/null
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
kubectl create deployment web --image=nginx:alpine --replicas=3
kubectl rollout status deploy/web

# 2. Eksplor -o
kubectl get pods -o wide
kubectl get pods -o name
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# 3. Logs & exec
kubectl logs deploy/web --tail=5
kubectl exec -it deploy/web -- ls /usr/share/nginx/html

# 4. Skenario error (ImagePullBackOff)
kubectl create deployment bad --image=nginx:notexist
kubectl get pod -l app=bad
kubectl describe pod -l app=bad | tail -15     # Events: Failed to pull image
kubectl delete deployment bad

# 5. Skenario CrashLoop (command salah)
kubectl create deployment crash --image=alpine -- sleep 1   # exit 0, Completed → CrashLoop? buat command yang error
kubectl run crash --image=alpine --restart=Always -- /bin/false   # exit 1 → CrashLoopBackOff
kubectl get pod crash
kubectl logs crash --previous
kubectl delete pod crash

# 6. Top
kubectl top nodes
kubectl top pods
```

## Ringkasan

| Mau... | Perintah |
|---|---|
| Lihat Pod | `kubectl get pods` (`-A`, `-o wide`, `-l label`) |
| Cerita + events | `kubectl describe pod <name>` |
| Log | `kubectl logs <pod>` (`--previous`, `-f`) |
| Masuk | `kubectl exec -it <pod> -- sh` |
| Forward port | `kubectl port-forward svc/<x> LOCAL:PORT` |
| Resource | `kubectl top nodes/pods` |
| Debug flow | get → describe (events) → logs → exec |
| Status buruk | Pending / ImagePullBackOff / CrashLoopBackOff / OOMKilled (137) |

## Cek Pemahaman

1. Sebutkan urutan debug flow saat Pod `CrashLoopBackOff` (5 langkah).
2. Pod status `ImagePullBackOff`. Sebut 3 kemungkinan penyebab & cara cek.
3. Kenapa `kubectl logs --previous` penting saat container crash?
4. Exit code 137 artinya apa? Penyebab umum?
5. Beda `kubectl apply -f` (deklaratif) vs `kubectl create deployment` (imperatif) — mana yang seharusnya dipakai di production & kenapa?
6. Pod production pakai image distroless (tanpa shell). Bagaimana Anda masuk untuk debug?