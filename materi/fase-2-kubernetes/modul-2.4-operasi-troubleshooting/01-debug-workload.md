# 01 — Debug Workload: Dari Symptom ke Root Cause

> Flow dasar SRE Kubernetes: **scope → evidence → hypothesis → perubahan terkecil → validasi → dokumentasi**.

## Tujuan

- Menggunakan flow `get → describe/events → logs → logs --previous → exec`.
- Membedakan status Pod yang sering tertukar.
- Menemukan masalah image, config, probe, resource, selector, dan node.
- Menggunakan evidence untuk membatasi blast radius.

## 1. Mulai dari Context dan Scope

Sebelum membaca atau mengubah resource:

```bash
kubectl config current-context
kubectl get namespaces
kubectl get nodes -o wide
kubectl get pods -n sre-lab -o wide
kubectl get deploy,svc,endpointslice -n sre-lab
```

Jangan mengandalkan nama context saja. Cocokkan node, namespace, dan workload yang diharapkan. Bila incident production, gunakan akun read-only terlebih dahulu.

Tentukan scope:

```text
Apakah satu Pod, satu Deployment, satu node, satu namespace, atau seluruh cluster?
Apakah hanya request external/VIP, atau akses internal juga gagal?
Kapan first known good dan perubahan terakhir?
```

## 2. Flow Debug Standar

### Langkah 1 — `get`

```bash
kubectl get pod <pod> -n sre-lab -o wide
kubectl get pod <pod> -n sre-lab -o jsonpath='{.status.phase}{"\n"}'
kubectl get pod <pod> -n sre-lab \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,READY:.status.containerStatuses[*].ready,RESTARTS:.status.containerStatuses[*].restartCount
```

`get` memberi snapshot cepat: fase, readiness, restart, node, dan umur.

### Langkah 2 — `describe` dan Events

```bash
kubectl describe pod <pod> -n sre-lab
kubectl get events -n sre-lab --sort-by=.lastTimestamp
kubectl describe deployment <deployment> -n sre-lab
```

Baca bagian `Events` dari bawah/terbaru. Di sana sering ada `FailedScheduling`, `Failed`, `BackOff`, `Unhealthy`, atau `Killing`.

### Langkah 3 — `logs`

```bash
kubectl logs <pod> -n sre-lab -c <container> --timestamps
kubectl logs deploy/<deployment> -n sre-lab --all-containers --tail=200
```

Untuk Pod dengan beberapa container, selalu sebut `-c`. Bedakan log aplikasi dengan event kubelet.

### Langkah 4 — `logs --previous`

```bash
kubectl logs <pod> -n sre-lab -c <container> --previous --timestamps
```

`--previous` membaca container instance sebelum restart. Ini penting saat container sudah crash dan log normal hanya menampilkan startup baru.

### Langkah 5 — `exec`

Hanya bila container sedang hidup dan tindakan ini aman:

```bash
kubectl exec -n sre-lab <pod> -c <container> -- env
kubectl exec -n sre-lab <pod> -c <container> -- sh -c 'id; ps; cat /etc/resolv.conf'
```

`exec` adalah evidence aktif; jangan mengedit file production dari dalam container sebagai "fix". Jika image distroless tidak memiliki shell, gunakan ephemeral/debug Pod sesuai kebijakan.

## 3. `CrashLoopBackOff`

### Arti

Container berhasil dibuat/start, lalu exit atau gagal health check berulang. Kubernetes menerapkan back-off sebelum restart berikutnya.

### Flow diagnosis

```bash
kubectl get pod <pod> -n sre-lab
kubectl describe pod <pod> -n sre-lab
kubectl logs <pod> -n sre-lab --all-containers --tail=200
kubectl logs <pod> -n sre-lab --all-containers --previous --tail=200
kubectl get pod <pod> -n sre-lab -o jsonpath='{.status.containerStatuses[*].state}{"\n"}'
```

Kemungkinan umum:

- command/entrypoint salah;
- env wajib tidak ada atau ConfigMap/Secret key salah;
- dependency/database belum reachable;
- liveness probe terlalu agresif atau path/port salah;
- permission filesystem;
- aplikasi exit normal tetapi `restartPolicy`/controller menganggap perlu restart;
- OOMKilled yang berulang.

Perbaiki sumber deklaratifnya (Deployment/ConfigMap/Secret/probe), bukan container yang sedang berjalan. Setelah perubahan:

```bash
# Bandingkan manifest yang akan diterapkan sebelum perubahan (bila file tersedia)
kubectl diff -f <manifest-yang-direview>.yaml
kubectl rollout status deployment/<deployment> -n sre-lab --timeout=120s
kubectl get pods -n sre-lab -w
```

## 4. `ImagePullBackOff` / `ErrImagePull`

### Arti

Kubelet tidak dapat mengambil image atau memulai image. Back-off adalah retry dengan jeda meningkat, bukan root cause.

```bash
kubectl describe pod <pod> -n sre-lab
kubectl get deploy <deployment> -n sre-lab -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

Cari event seperti:

| Evidence | Kemungkinan |
|---|---|
| `manifest unknown` | tag/image salah atau sudah dihapus |
| `unauthorized` | registry private, `imagePullSecret`/PAT salah |
| `no matching manifest` | image tidak menyediakan arsitektur ARM64 |
| `x509`/timeout | CA, DNS, proxy, atau network registry |
| `exec format error` | binary/image arsitektur tidak cocok |

Koreksi dengan tag yang valid dan image multi-arch. Jangan langsung memakai `:latest` karena menyulitkan reproduksi. Pastikan Secret registry tidak dimasukkan ke Git plain text.

```bash
kubectl get serviceaccount default -n sre-lab -o yaml
kubectl get secret -n sre-lab
kubectl rollout restart deployment/<deployment> -n sre-lab
```

## 5. `Pending`

### Arti

Pod belum mendapatkan node atau belum dapat menyelesaikan kebutuhan scheduling/volume. `Pending` bukan sinonim image error.

```bash
kubectl describe pod <pod> -n sre-lab
kubectl get nodes
kubectl describe node <node>
kubectl get pvc -n sre-lab
```

Evidence umum:

- `Insufficient cpu`/`Insufficient memory`: requests tidak muat pada node.
- `node(s) didn't match node selector/affinity`: constraint terlalu ketat.
- `untolerated taint`: Pod tidak memiliki toleration.
- PVC `Pending`: StorageClass/PV tidak tersedia.
- quota/limit range: namespace menolak atau membatasi resource.
- semua node `NotReady`/cordoned: tidak ada target schedule.

Perbaikan aman dapat berupa mengurangi request untuk workload lab, menambah kapasitas, memperbaiki label/affinity, atau menyediakan storage. Jangan menghapus Pod berulang-ulang; scheduler akan tetap gagal jika constraint belum berubah.

## 6. `OOMKilled` dan Exit Code 137

```bash
kubectl describe pod <pod> -n sre-lab
kubectl get pod <pod> -n sre-lab -o jsonpath='{range .status.containerStatuses[*]}{.name}{" reason="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{"\n"}{end}'
kubectl top pod <pod> -n sre-lab 2>/dev/null || true
```

`137 = 128 + SIGKILL (9)`. Sering berarti kernel/cgroup menghentikan proses karena melewati memory limit. Kemungkinan lain adalah node-level OOM; bedakan dengan melihat events node dan metrics.

Tindakan:

1. Cek tren penggunaan dan perubahan beban.
2. Cek memory request/limit serta QoS.
3. Pastikan bukan memory leak atau konfigurasi cache berlebihan.
4. Naikkan limit hanya jika kapasitas node dan biaya sudah dipahami.
5. Uji ulang dengan beban terukur.

Jangan menyimpulkan "naikkan limit" sebagai root fix otomatis.

## 7. Probe dan Readiness

- **Readiness** menentukan apakah Pod masuk Endpoints Service.
- **Liveness** menentukan apakah kubelet perlu me-restart container.
- **Startup probe** memberi waktu aplikasi lambat start sebelum liveness aktif.

```bash
kubectl describe pod <pod> -n sre-lab | sed -n '/Conditions:/,/Events:/p'
kubectl get endpointslice -n sre-lab -l kubernetes.io/service-name=<service>
```

Kesalahan umum: `port` probe bukan container port, path belum ada, aplikasi bind ke `127.0.0.1` alih-alih `0.0.0.0`, atau liveness digunakan untuk dependency eksternal yang sedang down. Readiness sebaiknya mengeluarkan Pod dari traffic; liveness harus menunjukkan proses yang memang perlu restart.

## 8. Service, Selector, dan Endpoint

Jika Pod `Running` tetapi request Service gagal:

```bash
kubectl get svc <service> -n sre-lab -o yaml
kubectl get pods -n sre-lab --show-labels
kubectl get endpointslice -n sre-lab -l kubernetes.io/service-name=<service> -o wide
kubectl describe svc <service> -n sre-lab
```

Periksa:

- label Pod cocok dengan `spec.selector` Service;
- `port` Service menuju `targetPort` yang benar;
- endpoint hanya berisi Pod Ready;
- NetworkPolicy tidak memblokir;
- untuk MetalLB, bedakan VIP/ARP dari jalur Service/Pod (Modul 2.3).

## 9. Scheduling, Node, dan ARM64

```bash
kubectl get nodes -L kubernetes.io/arch,kubernetes.io/os
kubectl describe node <node> | sed -n '/Taints:/,/Conditions:/p'
kubectl get pod <pod> -n sre-lab -o yaml | grep -A8 -E 'nodeSelector|affinity|tolerations|resources:'
```

Pada Mac M5 dan k3s ARM64, image harus memiliki manifest `linux/arm64`. `amd64` dapat gagal atau berjalan melalui emulasi yang lambat. Requests memengaruhi penjadwalan; limits membatasi runtime.

## 10. Incident Checklist

```text
[ ] current-context dan namespace diverifikasi
[ ] impact/scope/first known good dicatat
[ ] get + describe + events disimpan
[ ] logs saat ini dan --previous dikumpulkan
[ ] Service/EndpointSlice/Node diperiksa bila relevan
[ ] hypothesis dipisahkan dari evidence
[ ] perubahan terkecil dilakukan dengan approval
[ ] rollout/health divalidasi setelah perubahan
[ ] rollback/cleanup dilakukan bila lab
[ ] root cause, action item, dan pencegahan dicatat
```

## Catatan SRE

`kubectl describe` bukan root cause; ia adalah bukti. Root cause adalah penjelasan yang menghubungkan symptom dengan perubahan/keadaan sistem dan dapat diuji dengan validasi setelah perbaikan.

## Kaitan dengan Modul Berikutnya

- Requests, limits, QoS, dan HPA dibahas di [02-resource-hpa](02-resource-hpa.md).
- Snapshot etcd dibutuhkan agar perubahan cluster dapat dipulihkan di [03-etcd-backup-restore-k3s](03-etcd-backup-restore-k3s.md).
- Flow evidence ini dipakai saat drain dan upgrade pada [04-upgrade-k3s-rolling](04-upgrade-k3s-rolling.md).
