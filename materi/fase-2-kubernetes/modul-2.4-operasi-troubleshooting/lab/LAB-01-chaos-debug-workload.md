# LAB-01 — Chaos Workload & Debug Berlapis

> **Lane:** k3d fast lane. Semua failure dibuat di namespace `sre-lab` dan hanya pada cluster latihan yang sudah disetujui.

## Tujuan

Peserta menghasilkan incident report yang membuktikan diagnosis berbasis evidence untuk:

- `ImagePullBackOff`;
- `CrashLoopBackOff` karena konfigurasi/probe;
- `Pending` karena request terlalu besar;
- `OOMKilled` karena memory limit kecil;
- Service tanpa endpoint;
- reschedule setelah satu node k3d dihentikan.

## Guardrail dan Prasyarat

- Jangan jalankan lab pada context production.
- Pastikan k3d hanya berisi workload latihan.
- Jangan memakai `kubectl delete -A`.
- Jangan memakai image `latest`.
- Semua perubahan dibatasi `-n sre-lab`.
- Jika resource Mac tidak cukup, kecilkan jumlah node/replica; jangan menghapus guardrail diagnosis.

Buat cluster baru bila perlu:

```bash
k3d cluster create sre24-lab --servers 1 --agents 2 --wait
kubectl config current-context
kubectl get nodes -o wide
```

Pastikan context sesuai nama cluster sebelum setiap skenario.

## 1. Bootstrap Namespace dan Baseline

```bash
kubectl create namespace sre-lab
kubectl get ns sre-lab
kubectl get nodes -o wide
kubectl get pods -A
```

Deploy workload sehat:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: sre-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.27.1
        ports:
        - name: http
          containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: sre-lab
spec:
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: http
```

Simpan sebagai `/tmp/sre24-web.yaml` atau file lokal yang tidak berisi secret, lalu:

```bash
kubectl apply -f /tmp/sre24-web.yaml
kubectl rollout status deployment/web -n sre-lab --timeout=120s
kubectl get pods,svc,endpointslice -n sre-lab -o wide
kubectl port-forward -n sre-lab svc/web 18080:80
```

Terminal lain:

```bash
curl --max-time 5 -I http://127.0.0.1:18080/
```

Catat baseline: replica desired/ready, node placement, response code, dan timestamp.

## 2. Flow Evidence Wajib

Untuk setiap skenario, jalankan urutan berikut dan masukkan output relevan ke report:

```bash
kubectl config current-context
kubectl get pods -n sre-lab -o wide
kubectl get deploy,svc,endpointslice -n sre-lab
kubectl describe pod <pod> -n sre-lab
kubectl get events -n sre-lab --sort-by=.lastTimestamp
kubectl logs <pod> -n sre-lab --all-containers --tail=200 --timestamps
kubectl logs <pod> -n sre-lab --all-containers --previous --tail=200 --timestamps
```

Gunakan `exec` hanya pada Pod yang sedang hidup:

```bash
kubectl exec -n sre-lab <pod> -- sh -c 'id; cat /etc/resolv.conf'
```

Pisahkan evidence, hypothesis, perubahan, dan validasi. Jangan langsung menghapus Pod yang sedang diselidiki.

## 3. Skenario A — ImagePullBackOff

Buat Deployment dengan tag yang sengaja tidak ada:

```bash
kubectl create deployment broken-image -n sre-lab \
  --image=nginx:tag-yang-tidak-ada-untuk-lab
kubectl get pods -n sre-lab -l app=broken-image -w
```

Diagnosis:

```bash
kubectl describe pod -n sre-lab -l app=broken-image
kubectl get deploy broken-image -n sre-lab -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

Perbaikan deklaratif dengan tag immutable yang tersedia:

```bash
kubectl set image deployment/broken-image \
  -n sre-lab '*=nginx:1.27.1'
kubectl rollout status deployment/broken-image -n sre-lab --timeout=120s
```

Acceptance skenario:

- event menunjukkan tag/manifest gagal;
- image diperbaiki tanpa `latest`;
- Pod menjadi `Ready`;
- tidak ada Secret registry yang masuk report.

## 4. Skenario B — CrashLoopBackOff karena Probe

Buat workload yang menjalankan nginx tetapi readiness/liveness mengarah ke port salah:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-probe
  namespace: sre-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: broken-probe
  template:
    metadata:
      labels:
        app: broken-probe
    spec:
      containers:
      - name: web
        image: nginx:1.27.1
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /does-not-exist
            port: 80
          initialDelaySeconds: 1
          periodSeconds: 3
          failureThreshold: 2
```

```bash
kubectl apply -f /tmp/broken-probe.yaml
kubectl get pods -n sre-lab -l app=broken-probe -w
kubectl describe pod -n sre-lab -l app=broken-probe
kubectl logs -n sre-lab -l app=broken-probe --all-containers --previous --tail=100
```

Perbaiki sumber manifest dengan path `/` dan rollout:

```bash
kubectl patch deployment broken-probe -n sre-lab --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/httpGet/path","value":"/"}]'
kubectl rollout status deployment/broken-probe -n sre-lab --timeout=120s
```

Acceptance:

- membedakan `Running` dengan `Ready`;
- event probe failure dan restart tercatat;
- setelah perbaikan Pod stabil tanpa restart baru;
- readiness/liveness tidak digunakan untuk menyamarkan dependency eksternal.

## 5. Skenario C — Pending karena Request

Buat Pod dengan request memory yang lebih besar dari allocatable node k3d:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-request
  namespace: sre-lab
spec:
  containers:
  - name: pause
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "1000"
        memory: "999Gi"
      limits:
        cpu: "1000"
        memory: "999Gi"
```

```bash
kubectl apply -f /tmp/pending-request.yaml
kubectl get pod pending-request -n sre-lab -w
kubectl describe pod pending-request -n sre-lab
kubectl get nodes
kubectl describe node <node>
```

Perbaiki hanya pada object lab dengan request yang realistis:

```bash
kubectl patch pod pending-request -n sre-lab --type='json' \
  -p='[{"op":"replace","path":"/spec/containers/0/resources/requests/memory","value":"32Mi"},{"op":"replace","path":"/spec/containers/0/resources/limits/memory","value":"64Mi"}]'
```

Jika Pod tidak dapat diubah karena field immutable, hapus hanya object yang dibuat lab setelah evidence tersimpan lalu apply manifest versi benar:

```bash
kubectl delete pod pending-request -n sre-lab
kubectl apply -f /tmp/pending-request-fixed.yaml
```

Acceptance:

- event `Insufficient memory/cpu` atau alasan scheduling lain dicatat;
- `Pending` dibedakan dari image pull error;
- request diperbaiki melalui manifest lab;
- Pod menjadi `Running` tanpa mengubah node production.

## 6. Skenario D — OOMKilled

Gunakan Pod dengan memory limit kecil dan proses yang meminta lebih banyak memory:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-lab
  namespace: sre-lab
spec:
  restartPolicy: Always
  containers:
  - name: allocator
    image: python:3.12-alpine
    command: ["python", "-c"]
    args:
    - "x=bytearray(128*1024*1024); import time; time.sleep(60)"
    resources:
      requests:
        memory: 16Mi
      limits:
        memory: 32Mi
```

```bash
kubectl apply -f /tmp/oom-lab.yaml
kubectl get pod oom-lab -n sre-lab -w
kubectl describe pod oom-lab -n sre-lab
kubectl get pod oom-lab -n sre-lab -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}{" exit="}{.status.containerStatuses[*].lastState.terminated.exitCode}{"\n"}'
kubectl logs pod/oom-lab -n sre-lab --previous --timestamps
```

Diagnosis harus menyebutkan `OOMKilled`, exit code 137 bila muncul, limit, dan apakah ini cgroup atau node-level OOM. Jangan sekadar menaikkan limit tanpa capacity check. Setelah evidence:

```bash
kubectl delete pod oom-lab -n sre-lab
```

Acceptance:

- status termination dan resource limit dicatat;
- limit container dibedakan dari tekanan memory node;
- cleanup dibatasi ke `oom-lab`;
- action item mencakup tuning, capacity, dan kemungkinan memory leak.

## 7. Skenario E — Service tanpa Endpoint

Buat Service selector yang sengaja tidak cocok:

```bash
kubectl expose deployment web -n sre-lab --name web-broken \
  --port=80 --target-port=80 --selector='app=label-yang-salah'
kubectl get svc web-broken -n sre-lab
kubectl get endpointslice -n sre-lab -l kubernetes.io/service-name=web-broken
```

Diagnosis dan perbaikan:

```bash
kubectl get pods -n sre-lab --show-labels
kubectl describe svc web-broken -n sre-lab
kubectl patch svc web-broken -n sre-lab --type='merge' \
  -p='{"spec":{"selector":{"app":"web"}}}'
kubectl get endpointslice -n sre-lab -l kubernetes.io/service-name=web-broken -o wide
kubectl delete svc web-broken -n sre-lab
```

Acceptance:

- `Running` Pod tidak dianggap otomatis dapat diakses;
- selector, targetPort, readiness, dan EndpointSlice diperiksa;
- Service lab dibersihkan setelah validasi.

## 8. Skenario F — Stop Satu Node dan Reschedule

Catat placement awal:

```bash
kubectl get pods -n sre-lab -o wide
kubectl get nodes
```

Pada node k3d agent yang **bukan satu-satunya lokasi control plane**, hentikan container k3d dari terminal host:

```bash
docker ps --format '{{.Names}}\t{{.Label "k3d.cluster"}}\t{{.Label "k3d.role"}}'
docker stop k3d-sre24-lab-agent-0
```

Amati dari Mac:

```bash
kubectl get nodes -w
kubectl get pods -n sre-lab -o wide -w
kubectl get events -n sre-lab --sort-by=.lastTimestamp
```

Jangan melakukan stop pada node control-plane jika tidak memahami dampak API. Pulihkan node lab:

```bash
docker start k3d-sre24-lab-agent-0
kubectl get nodes -w
kubectl rollout status deployment/web -n sre-lab --timeout=120s
```

Acceptance:

- node yang dihentikan dan waktu kejadian dicatat;
- Pod terdampak dan reschedule/availability diamati;
- Service tetap diuji setelah node pulih;
- tidak ada tindakan pada container/node di luar cluster lab.

## 9. Incident Report

Gunakan format berikut untuk setiap skenario:

```text
Incident ID:
Context/cluster:
Namespace:
Waktu UTC:
Symptom:
Scope/blast radius:
First known good:
Evidence (get/describe/events/logs/previous/exec):
Hypothesis:
Perubahan terkecil:
Validation before/after:
Root cause:
Mitigasi:
Pencegahan/action item:
Cleanup:
```

## Cleanup

Pastikan context tetap benar sebelum menghapus resource lab:

```bash
kubectl config current-context
kubectl delete namespace sre-lab
kubectl get ns sre-lab
```

Jika cluster dibuat khusus lab dan sudah tidak diperlukan:

```bash
k3d cluster delete sre24-lab
```

## Acceptance Criteria LAB-01

- [ ] Context k3d dan namespace lab diverifikasi.
- [ ] Baseline workload sehat dan Service diuji.
- [ ] Lima status/kegagalan (image, crash/probe, pending, OOM, endpoint) didiagnosis dengan evidence.
- [ ] Satu node dihentikan dan recovery/reschedule diamati.
- [ ] Setiap skenario memiliki before/after dan root cause, bukan hanya command.
- [ ] Perubahan dan cleanup terbatas pada `sre-lab`/cluster lab.
- [ ] Incident report selesai dan tidak berisi token/Secret.

## Catatan SRE

Chaos tanpa observability hanya menghasilkan kerusakan. Nilai lab ini ada pada scope yang aman, evidence yang dapat diulang, perubahan minimal, dan pembelajaran yang masuk ke runbook.

## Kaitan dengan Modul Berikutnya

Hasil diagnosis resource dan kesiapan workload menjadi input [LAB-02 — etcd backup/restore & rolling upgrade](LAB-02-etcd-backup-restore-rolling-upgrade.md). VIP/Service dari Modul 2.3 tetap menjadi jalur validasi eksternal.
