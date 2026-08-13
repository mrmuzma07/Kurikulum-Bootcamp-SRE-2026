# 02 — Resource, QoS & HPA

> Kubernetes dapat menjadwalkan dan membatasi workload secara terukur hanya jika resource request/limit dan metrics dipahami.

## Tujuan

- Menghubungkan `requests`/`limits` dengan scheduler dan cgroup.
- Memahami QoS class dan perilaku saat node kehabisan memory.
- Menggunakan `kubectl top` dengan metrics-server.
- Membuat HPA CPU/memory dan menguji scale secara terkontrol.
- Mengenali kapan HPA bukan solusi.

## 1. Requests dan Limits

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

- **Request** = kapasitas yang dijanjikan untuk scheduling. Scheduler mencari node dengan allocatable yang masih cukup.
- **Limit CPU** = ceiling throttling CPU (bergantung runtime/kernel).
- **Limit memory** = ceiling; pemakaian melewati limit dapat berakhir `OOMKilled`.
- Keduanya per container; total Pod adalah jumlah container.

`100m` CPU berarti 0,1 core. Memory memakai unit biner (`Mi`, `Gi`) atau desimal (`M`, `G`); pilih dan dokumentasikan konsisten.

Hubungan ke Modul 1.1:

```text
container cgroup → resources.limits pada container Pod
Docker Compose resource config → requests/limits Kubernetes
```

Request bukan batas pemakaian dan limit bukan jaminan kapasitas node.

## 2. QoS Class

| QoS | Kondisi | Dampak umum saat tekanan node |
|---|---|---|
| `Guaranteed` | Setiap container punya CPU dan memory request = limit | prioritas eviction relatif paling tinggi |
| `Burstable` | Ada request/limit, tetapi tidak semuanya sama | di tengah |
| `BestEffort` | Tidak ada request/limit | paling rentan di-evict |

```bash
kubectl get pod <pod> -n sre-lab -o jsonpath='{.status.qosClass}{"\n"}'
kubectl describe node <node> | sed -n '/Allocated resources:/,/Events:/p'
```

QoS bukan SLA. Workload `Guaranteed` tetap dapat gagal karena aplikasi, node mati, atau dependency down.

## 3. Capacity dan Scheduling

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.capacity.cpu,ALLOCATABLE_CPU:.status.allocatable.cpu,MEM:.status.capacity.memory,ALLOCATABLE_MEM:.status.allocatable.memory
kubectl describe node <node>
kubectl get pods -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory
```

Scheduler memakai request, bukan penggunaan aktual. Karena itu Pod dapat `Pending` walaupun `kubectl top` menunjukkan CPU idle: request seluruh Pod mungkin sudah memenuhi allocatable atau ada taint/affinity constraint.

Perhitungan sederhana:

```text
allocatable memory node = 2 Gi
request Pod A            = 1 Gi
request Pod B            = 1.5 Gi
→ Pod B tidak dapat dijadwalkan pada node yang sama, walaupun penggunaan aktual A hanya 100 Mi
```

## 4. Metrics dan `kubectl top`

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl top pod <pod> -n sre-lab --containers
```

Jika gagal:

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl get pods -n kube-system | grep -i metrics
kubectl describe apiservice v1beta1.metrics.k8s.io
```

`kubectl top` menunjukkan snapshot/resource usage dari metrics pipeline, bukan historical monitoring dan bukan billing-accurate. HPA membutuhkan metrics API yang sehat. Metrics-server perlu TLS/network yang cocok dengan node; jangan mematikan verifikasi TLS secara permanen hanya untuk membuat demo bekerja.

## 5. HPA

Contoh HPA v2 untuk CPU:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
  namespace: sre-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 6
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
    scaleDown:
      stabilizationWindowSeconds: 60
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Agar utilization CPU bermakna, container target harus punya `resources.requests.cpu`:

```bash
kubectl apply -f hpa.yaml
kubectl get hpa -n sre-lab
kubectl describe hpa web -n sre-lab
kubectl get deploy web -n sre-lab -w
```

Arti target 70%: rata-rata penggunaan CPU dibanding CPU request, bukan 70% dari seluruh host.

Memory HPA juga dapat digunakan:

```yaml
metrics:
- type: Resource
  resource:
    name: memory
    target:
      type: Utilization
      averageUtilization: 75
```

Memory tidak selalu kembali turun karena allocator/cache. Scale-down memory HPA dapat lambat atau tidak terjadi; observability dan load test harus memahami karakter aplikasi.

## 6. Load Test Terukur

Gunakan image/tool yang diizinkan. Contoh dari Pod debug:

```bash
kubectl run load-debug -n sre-lab --rm -it --restart=Never \
  --image=busybox:1.36 -- sh
```

Dari shell debug, kirim request ke Service secara terkontrol. Jangan melakukan flood ke cluster atau jaringan bersama. Amati di terminal lain:

```bash
kubectl get hpa web -n sre-lab -w
kubectl get deploy web -n sre-lab -w
kubectl top pods -n sre-lab
kubectl get events -n sre-lab --sort-by=.lastTimestamp
```

Acceptance load test bukan sekadar replica naik; pastikan:

- metrics tersedia;
- replica benar-benar Ready;
- latency/error rate dicatat;
- scale-down tidak memutus traffic;
- maxReplicas dan resource capacity cukup.

## 7. HPA Bukan Solusi Universal

HPA tidak menyelesaikan:

- image gagal pull;
- selector/endpoint salah;
- dependency database down;
- node tidak cukup kapasitas;
- memory leak yang hanya memperbanyak Pod rusak;
- bottleneck storage/network/external API;
- queue workload yang memerlukan KEDA/custom metric.

Autoscaling harus dipasangkan dengan:

```text
workload metrics → capacity planning → limits/requests → PDB → observability → alert/runbook
```

## 8. Tuning dan Safety

- Set request berdasarkan observasi baseline, bukan angka acak.
- Limit CPU terlalu rendah dapat menyebabkan throttling dan probe timeout.
- Limit memory terlalu tinggi dapat menekan node dan mengurangi headroom.
- Jangan menghapus limits untuk "menghilangkan OOM" tanpa memahami blast radius.
- Gunakan `maxReplicas` sebagai guardrail biaya/kapasitas.
- HPA dan Deployment rollout dapat berinteraksi; lihat replica desired/current/available.
- Untuk Pod dengan storage/state, scale-out harus mempertimbangkan konsistensi data.

## 9. Checklist Resource Incident

```text
[ ] request/limit setiap container diketahui
[ ] QoS class dicatat
[ ] allocatable node dan taint diperiksa
[ ] kubectl top tersedia dan timestamp dicatat
[ ] HPA current/desired/conditions diperiksa
[ ] metrics-server/APIService sehat
[ ] replica Ready, error, latency, dan event diamati
[ ] perubahan ditinjau terhadap capacity dan PDB
```

## Catatan SRE

Autoscaling yang tidak memiliki limit, metric, atau capacity plan hanya memindahkan kegagalan dari satu Pod ke banyak Pod. Ukur sebelum mengubah.

## Kaitan dengan Modul Berikutnya

- `resources` dan workload stability digunakan saat [03-etcd-backup-restore-k3s](03-etcd-backup-restore-k3s.md) dan maintenance.
- PDB serta replica readiness penting saat [04-upgrade-k3s-rolling](04-upgrade-k3s-rolling.md).
- Metrics dan alerting akan diperdalam di Fase 7 Observability.
