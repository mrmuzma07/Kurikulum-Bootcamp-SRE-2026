# Latihan Modul 2.4 — Operasi & Troubleshooting Kubernetes

> Kerjakan pada cluster lab yang disetujui. Jawaban harus menyertakan evidence, bukan hanya kesimpulan.

## Instruksi Umum

Untuk setiap latihan:

1. tulis context dan scope;
2. ambil evidence read-only;
3. buat hypothesis yang dapat diuji;
4. lakukan perubahan terkecil bila diminta;
5. validasi before/after;
6. catat cleanup, risiko, dan action item.

Jangan memasukkan token, kubeconfig, PAT, atau Secret plain text ke jawaban.

## Latihan 1 — Evidence Flow

Sebuah Deployment memiliki tiga replica, tetapi satu Pod restart 12 kali dan tidak Ready.

1. Tulis command berurutan dari `current-context` sampai `logs --previous`.
2. Jelaskan kapan `exec` tidak dapat digunakan.
3. Buat template incident timeline lima timestamp.

**Deliverable:** command plan, evidence table, dan timeline.

## Latihan 2 — Bedakan Status Workload

Lengkapi tabel berikut dengan minimal dua kemungkinan root cause dan satu evidence utama:

| Symptom | Root cause kandidat | Evidence |
|---|---|---|
| `CrashLoopBackOff` |  |  |
| `ImagePullBackOff` |  |  |
| `Pending` |  |  |
| `OOMKilled` |  |  |

Tambahkan satu contoh root cause yang **bukan** solusi untuk masing-masing status.

## Latihan 3 — Selector dan Readiness

Pod berstatus `Running`, tetapi Service tidak memiliki EndpointSlice address.

1. Tulis command untuk membandingkan label Pod dengan selector Service.
2. Jelaskan pengaruh readiness probe terhadap endpoint.
3. Sebutkan tiga lapisan yang diperiksa sebelum menyimpulkan MetalLB bermasalah.

## Latihan 4 — Resource dan QoS

Sebuah node memiliki memory allocatable `2Gi`. Pod A meminta `1Gi`, Pod B meminta `1.5Gi`, sedangkan penggunaan aktual Pod A hanya `100Mi`.

1. Mengapa Pod B dapat `Pending`?
2. Tentukan QoS class untuk:
   - semua container memiliki request=limit CPU dan memory;
   - sebagian container memiliki request/limit;
   - tidak ada request/limit.
3. Jelaskan perbedaan request, CPU limit, dan memory limit.

## Latihan 5 — OOM Investigation

Pod berakhir dengan reason `OOMKilled`, exit code 137.

1. Tulis command untuk membaca `lastState.terminated`.
2. Evidence apa yang membedakan cgroup memory limit dari node-level OOM?
3. Tulis tiga action item selain “naikkan limit”.

## Latihan 6 — Metrics dan HPA

HPA menunjukkan `unknown` pada target CPU.

1. Periksa APIService dan Pod metrics-server dengan command.
2. Mengapa CPU utilization HPA memerlukan `requests.cpu`?
3. Sebutkan empat masalah yang tidak diselesaikan HPA.
4. Buat rencana load test terukur tanpa flood cluster.

## Latihan 7 — Snapshot etcd

Susun runbook snapshot manual k3s yang mencakup:

- context dan server target;
- disk/service preflight;
- pemeriksaan `k3s etcd-snapshot --help`;
- path artifact;
- ukuran, timestamp, permission, checksum;
- salinan remote dan retention;
- RPO/RTO.

Jelaskan mengapa snapshot etcd bukan backup PV/database.

## Latihan 8 — Restore Gate

Seorang operator meminta Anda menjalankan `cluster-reset` pada cluster yang melayani user karena “snapshot sudah ada”.

1. Apakah Anda menjalankan command tersebut? Jelaskan.
2. Buat tujuh precondition untuk restore.
3. Buat validasi pasca-restore untuk API, node, object, workload, dan data aplikasi.

## Latihan 9 — Quorum

Bandingkan toleransi failure untuk 1, 3, dan 5 embedded etcd server. Untuk cluster tiga server, apa risiko menghentikan dua server bersamaan? Buat stop condition sebelum maintenance server.

## Latihan 10 — Rolling Upgrade Runbook

Tulis runbook upgrade k3s satu node pada satu waktu yang mencakup:

1. compatibility dan release review;
2. backup;
3. PDB/replica/local PV check;
4. cordon dan drain;
5. upgrade server/agent;
6. API/node/workload validation;
7. uncordon dan observation window;
8. abort/rollback.

Tambahkan command yang memiliki scope `<node>` eksplisit.

## Latihan 11 — Drain dan PDB

Drain tertahan karena PDB `minAvailable: 2`, tetapi hanya ada dua replica dan salah satunya NotReady.

1. Evidence apa yang dikumpulkan?
2. Mengapa menambah `--force` bukan solusi default?
3. Buat dua opsi mitigasi yang memelihara availability.

## Latihan 12 — External Path

Selama upgrade node, semua node `Ready`, tetapi curl ke VIP MetalLB menghasilkan timeout.

Buat matriks pemeriksaan berlapis:

```text
VIP allocation → speaker/ARP → node listener → Service → EndpointSlice → Pod readiness → HTTP application
```

Untuk tiap lapisan, tulis satu command/evidence dan satu kemungkinan failure.

## Latihan 13 — Chaos Incident Report

Gunakan LAB-01 untuk membuat satu report lengkap dari `ImagePullBackOff` atau `OOMKilled`.

**Minimum isi:** symptom, scope, first known good, evidence, hypothesis, perubahan terkecil, validasi, root cause, cleanup, dan pencegahan.

## Latihan 14 — Capacity Reflection

Diberikan:

```text
3 server etcd + 2 worker
worker allocatable: 2 CPU, 4 Gi memory per node
web: 3 replica, request 250m/256Mi
HPA maxReplicas: 12
```

1. Faktor apa yang menentukan apakah maxReplicas dapat dijadwalkan?
2. Mengapa HPA dapat memperburuk incident bila request terlalu kecil atau dependency down?
3. Apa perubahan pada PDB dan capacity plan sebelum upgrade?

## Latihan 15 — Evidence dan Safety Review

Review command berikut dan perbaiki agar aman untuk lab:

```bash
kubectl delete -A
kubectl drain node-1 --force
kubectl config use-context production
k3s server --cluster-reset --cluster-reset-restore-path=/path/snapshot
```

Tulis versi yang memiliki context check, target/scope, approval, dan stop condition. Untuk restore, jawab dengan prosedur gate, bukan command yang dieksekusi pada cluster aktif.

## Rubrik Latihan

| Kriteria | Poin |
|---|---:|
| Context/scope dan guardrail | 20 |
| Evidence teknis tepat | 25 |
| Hypothesis/root cause dapat diuji | 20 |
| Perubahan dan validasi | 20 |
| Incident timeline, cleanup, action item | 15 |
| **Total** | **100** |

Target kelulusan latihan: **80/100**. Lab dapat diulang jika evidence tidak lengkap atau perubahan dilakukan tanpa scope.

## Kaitan dengan Modul Berikutnya

Gunakan hasil latihan sebagai bahan [kuis dan jawaban](kuis-dan-jawaban.md), lalu bandingkan runbook manual dengan automation Ansible pada Fase 4.
