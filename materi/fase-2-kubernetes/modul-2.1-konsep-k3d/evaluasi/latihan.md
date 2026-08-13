# Latihan — Modul 2.1 Konsep & k3d untuk Latihan

Latihan ini memperkuat pemahaman **arsitektur K8s**, **objek inti**, **k3d**, dan **kubectl survival kit**. Kerjakan di terminal Mac (OrbStack + k3d).

> **Aturan:** kerjakan di terminal. Catat output penting di `m2.1/lab/log-latihan.md` di repo `sre-bootcamp`.

---

## Bagian 1 — Arsitektur

### 1.1 Komponen Control Plane
```bash
k3d cluster delete lab 2>/dev/null
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
kubectl get pods -n kube-system
kubectl get pods -n kube-system -o wide
```
Catat: di node mana CoreDNS, Traefik, metrics-server jalan? (server atau agent) — kenapa?

### 1.2 Lihat Komponen di Node
```bash
docker exec k3d-lab-server-0 ps aux 2>/dev/null | grep -E "apiserver|etcd|kubelet|containerd|scheduler|controller" | grep -v grep | wc -l
```
Catat: berapa proses k3s yang merangkap komponen control plane? Ini bukti k3s menggabungkan komponen.

### 1.3 etcd / State
```bash
kubectl get --raw='/readyz?verbose' 2>/dev/null | head -20
kubectl get pods -A
```
Catat: dari mana `kubectl get pods` mendapat data? Apa yang terjadi kalau etcd hilang tanpa backup?

---

## Bagian 2 — Objek Inti

### 2.1 Pod & Resource Limits
Kerjakan [LAB-01](../lab/LAB-01-pod-deployment-service.md) dan catat di laporan:
1. Set `replicas: 3`, di node mana Pod tersebar? Merata atau di satu node?
2. Ubah `resources.limits.memory: 10Mi` (terlalu kecil), apply. Apa status Pod? Exit code berapa?
3. Hapus 1 Pod — berapa detik Deployment buat yang baru?

### 2.2 Service & Selector
```bash
kubectl apply -f https://k8s.io/examples/service/networking/load-balancer-example.yaml 2>/dev/null || true
# atau pakai deployment dari LAB-01
kubectl get svc,deploy,endpoints -l app=app
```
Catat: bagaimana Service tahu Pod mana yang dilayani? Apa yang terjadi kalau label Pod diubah (mismatch)?

### 2.3 Ingress Multi-Host
Kerjakan [LAB-02](../lab/LAB-02-ingress-configmap-secret.md) dan catat:
1. Satu IP (Mac:8080) melayani dua host — bagaimana Traefik membedakan `app.k3d.local` vs `api.k3d.local`?
2. Tanpa Ingress, berapa NodePort/LoadBalancer dibutuhkan untuk dua service? (bandingkan)

### 2.4 ConfigMap & Secret
```bash
kubectl create configmap cfg --from-literal=KEY=val
kubectl get cm cfg -o yaml | head
kubectl create secret generic sec --from-literal=TOKEN=abc
kubectl get secret sec -o jsonpath='{.data.TOKEN}' | base64 -d; echo
```
Catat: kenapa `kubectl get cm` menampilkan nilai langsung, tapi `kubectl get secret` menampilkan base64? Apakah base64 = aman untuk commit?

### 2.5 PVC Persisten
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data}
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 100Mi}}
EOF
kubectl get pvc data
kubectl delete pvc data
```
Catat: siapa yang membuat PV otomatis di k3s? (provisioner) Apa yang terjadi pada data Pod DB tanpa PVC vs dengan PVC saat Pod dihapus?

---

## Bagian 3 — k3d

### 3.1 Lifecycle & Kubeconfig
```bash
k3d cluster create sandbox --port 8081:80@loadbalancer
k3d kubeconfig merge sandbox --switch
kubectl config get-contexts
k3d cluster stop sandbox
kubectl get nodes         # error? kenapa?
k3d cluster start sandbox
kubectl get nodes
k3d cluster delete sandbox
kubectl config delete-context k3d-sandbox 2>/dev/null
```
Catat: beda `stop` vs `delete` pada state/Pod. Kenapa `kubectl get nodes` gagal saat `stop`?

### 3.2 Multi-Cluster Context
```bash
k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer
k3d cluster create ci --port 8082:80@loadbalancer
kubectl config get-contexts
kubectl config use-context k3d-ci
kubectl get nodes          # cluster ci
kubectl config use-context k3d-lab
kubectl get nodes          # cluster lab
k3d cluster delete ci lab
```
Catat: risiko operasi `kubectl delete` saat salah context. Bagaimana mencegah?

---

## Bagian 4 — kubectl Survival Kit

### 4.1 Output Format
```bash
kubectl create deployment web --image=nginx:alpine --replicas=3 -n default
kubectl get pod -o wide
kubectl get pod -o name
kubectl get pod -o jsonpath='{.items[*].metadata.name}'
kubectl get pod -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```
Catat: kapan `-o jsonpath` berguna dibanding `-o wide`? (sebut 1 kasus: automation/CI)

### 4.2 Debug Flow Skenario
Buat Pod error dan jalankan debug flow:
```bash
kubectl run bad --image=nginx:notexist      # ImagePullBackOff
kubectl get pod bad
kubectl describe pod bad | tail -15
kubectl logs bad 2>&1 || true
kubectl delete pod bad

kubectl run crash --image=alpine --restart=Always -- /bin/false   # CrashLoopBackOff (exit 1)
kubectl get pod crash -w    # amati beberapa saat, Ctrl+C
kubectl logs crash --previous
kubectl delete pod crash
```
Catat: untuk `ImagePullBackOff`, apa pesan di Events? Untuk `CrashLoopBackOff`, apa exit code & kenapa `--previous` penting?

### 4.3 Top & Port-Forward
```bash
kubectl top nodes
kubectl top pods -A
kubectl port-forward deploy/web 9090:80 &
sleep 2
curl -s http://localhost:9090/ | head -1
kill %1 2>/dev/null
kubectl delete deployment web
```

---

## Bagian 5 — Soal Refleksi

Tulis jawaban singkat di `m2.1/lab/log-latihan.md`:
1. Seorang rekan bilang "Kubernetes itu seperti Docker Compose tapi banyak node." Setujukah Anda? Jelaskan apa yang benar & apa yang kurang.
2. Kenapa `kubectl apply -f` (deklaratif) lebih baik dari `kubectl create` (imperatif) untuk production? (sebut 2 alasan)
3. Service `type: LoadBalancer` di cloud otomatis dapat IP dari provider. Di on-prem (tanpa MetalLB), apa yang terjadi? (pengantar Modul 2.3)
4. Anda punya context `k3d-lab` (latihan) & `k3s-prod` (production). Sebut 2 kebiasaan agar tidak salah `kubectl delete` di production.
5. Jelaskan jalur lengkap `kubectl apply -f pod.yaml` sampai Pod `Running` (sebut 4 komponen yang dilalui).

---

## Catatan Performa

- [ ] Semua latihan di terminal OrbStack + k3d
- [ ] Output penting disimpan di `m2.1/lab/log-latihan.md` di repo
- [ ] Bisa menjelaskan arsitektur K8s (control plane, kubelet, etcd) & jalur kubectl → Pod
- [ ] Bisa menulis manifest Deployment + Service + Ingress + ConfigMap + Secret dari nol
- [ ] Bisa membuat/hapus cluster k3d multi-node & mengelola kubeconfig multi-context
- [ ] Bisa menjalankan debug flow (get → describe → logs → exec) & mengenali status Pod buruk
- [ ] Bisa menjelaskan k3d (fast lane) vs k3s (production lane) & kapan pakai masing-masing