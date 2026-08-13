# Kuis & Jawaban Modul 2.4

> Target kelulusan: **minimal 80%**. Pilih satu jawaban terbaik untuk pilihan ganda, lalu jawab esai dengan alasan dan evidence yang dapat diverifikasi.

## A. Pilihan Ganda

### 1. Urutan awal troubleshooting yang paling tepat adalah ...

A. Hapus semua Pod lalu apply ulang
B. `get → describe/events → logs → logs --previous → exec`
C. Naikkan memory limit lalu restart node
D. Ganti image ke `latest`

### 2. `logs --previous` berguna ketika ...

A. Pod belum pernah start
B. Container sebelumnya sudah terminated dan restart
C. Service tidak memiliki VIP
D. Node tidak memiliki label

### 3. Event `manifest unknown` paling mungkin menunjukkan ...

A. PVC penuh
B. Tag/image tidak ada di registry
C. PDB terlalu ketat
D. Node cordoned

### 4. `Pending` dengan event `Insufficient memory` terutama terkait ...

A. CPU usage aktual saja
B. Pod request dan allocatable node
C. HTTP status aplikasi
D. ARP VIP

### 5. Memory limit terlewati biasanya dapat menghasilkan ...

A. `OOMKilled`
B. `ImagePullBackOff`
C. `NoSchedule` taint otomatis
D. `EXTERNAL-IP` baru

### 6. CPU utilization HPA berbasis `Utilization` dibandingkan terhadap ...

A. seluruh RAM host
B. CPU request Pod target
C. jumlah node control plane
D. memory limit container

### 7. Jika `kubectl top` gagal, pemeriksaan awal yang tepat adalah ...

A. Hapus namespace kube-system
B. Periksa APIService metrics dan metrics-server
C. Ubah semua image ke amd64
D. Jalankan `kubectl delete -A`

### 8. QoS `Guaranteed` memerlukan ...

A. tidak ada resource stanza
B. setiap container memiliki CPU/memory request yang sama dengan limit
C. HPA selalu aktif
D. Pod berjalan di control plane

### 9. Snapshot embedded etcd terutama melindungi ...

A. state object Kubernetes
B. isi seluruh PV dan database eksternal
C. image registry
D. konfigurasi switch fisik

### 10. Cluster embedded etcd tiga server umumnya dapat mentoleransi ...

A. kehilangan nol server saja
B. kehilangan satu server tanpa kehilangan quorum
C. kehilangan dua server dan tetap write normal
D. semua server hilang

### 11. Restore etcd seharusnya pertama kali diuji pada ...

A. production aktif tanpa approval
B. cluster disposable/terisolasi dengan snapshot terverifikasi
C. node worker acak
D. namespace system production

### 12. Sebelum `kubectl drain`, operator perlu memeriksa ...

A. PDB, replica readiness, local PV, dan scope node
B. hanya warna prompt shell
C. hanya image tag
D. hanya DNS Mac

### 13. Tujuan `cordon` adalah ...

A. menghentikan semua API server
B. mencegah Pod baru dijadwalkan ke node
C. menghapus PV
D. mengiklankan ARP

### 14. Setelah upgrade node berhasil divalidasi, tindakan normal adalah ...

A. `uncordon` node tersebut
B. `delete -A`
C. reset seluruh etcd
D. menghapus PDB

### 15. Pada tiga server etcd, dua server tidak boleh dibuat unavailable bersamaan karena ...

A. Service kehilangan selector
B. quorum hilang
C. HPA scale down
D. image menjadi amd64

### 16. Semua node `Ready` tetapi VIP timeout. Lapisan berikutnya yang paling tepat diperiksa adalah ...

A. allocation/speaker/ARP lalu Service/EndpointSlice
B. langsung upgrade image
C. hapus semua endpoint
D. ubah kubeconfig menjadi production

### 17. Mengapa `latest` tidak ideal untuk incident reproduction?

A. selalu tidak dapat dipull
B. tag dapat berubah sehingga hasil tidak reproducible
C. hanya bekerja di amd64
D. otomatis membuat PDB

### 18. Stop condition rolling upgrade yang valid adalah ...

A. node upgraded tetap `NotReady`
B. semua command sudah diketik
C. operator ingin cepat selesai
D. log dikosongkan

### 19. Etcd snapshot bukan pengganti backup database karena ...

A. etcd tidak menyimpan data aplikasi pada PV/database eksternal
B. etcd hanya menyimpan HTTP log
C. snapshot hanya berisi image
D. database selalu stateless

### 20. Praktik context safety yang benar sebelum operasi berisiko adalah ...

A. cek `kubectl config current-context`, nodes, dan target scope
B. gunakan context terakhir yang aktif tanpa cek
C. pakai `kubectl delete -A`
D. cetak token ke laporan

## B. Esai

### 21. Jelaskan flow diagnosis `CrashLoopBackOff` dan peran `logs --previous`.

Sertakan minimal lima command dan bedakan evidence dari hypothesis.

### 22. Bandingkan request, limit, QoS, dan HPA.

Jelaskan mengapa HPA bukan solusi untuk image gagal, endpoint kosong, dan dependency database down.

### 23. Buat runbook snapshot/restore etcd yang aman.

Wajib mencakup quorum, path/permission, checksum, RPO/RTO, disposable cluster, validasi, dan larangan restore pada production aktif.

### 24. Buat runbook rolling upgrade k3s HA.

Wajib mencakup backup, PDB, cordon/drain, server/agent sequence, quorum, observation window, stop condition, dan abort/rollback.

### 25. Sebuah Service memiliki VIP MetalLB tetapi request timeout setelah node maintenance.

Jelaskan troubleshooting berlapis dari VIP sampai Pod/HTTP dan evidence yang dikumpulkan.

## Kunci Jawaban Pilihan Ganda

| No | Jawaban | Alasan singkat |
|---:|:---:|---|
| 1 | B | Mulai dengan scope dan evidence berlapis. |
| 2 | B | Membaca instance container sebelum restart. |
| 3 | B | Registry tidak menemukan manifest/tag. |
| 4 | B | Scheduler memakai request terhadap allocatable. |
| 5 | A | Limit memory dapat memicu OOMKilled. |
| 6 | B | Utilization CPU dibanding CPU request. |
| 7 | B | HPA/top memerlukan metrics API sehat. |
| 8 | B | CPU dan memory request=limit setiap container. |
| 9 | A | Snapshot berisi state etcd/Kubernetes, bukan PV otomatis. |
| 10 | B | Tiga member memerlukan dua untuk quorum. |
| 11 | B | Restore mengganti state dan harus terisolasi. |
| 12 | A | Drain dapat mengganggu availability dan state. |
| 13 | B | Cordon mencegah scheduling baru. |
| 14 | A | Scheduling dikembalikan setelah validasi. |
| 15 | B | Dua dari tiga membuat quorum hilang. |
| 16 | A | Node Ready tidak membuktikan data plane sehat. |
| 17 | B | Mutable tag merusak reproducibility. |
| 18 | A | Jangan lanjut saat node tidak sehat. |
| 19 | A | Data aplikasi/PV/database berada di luar state etcd. |
| 20 | A | Context dan scope wajib diverifikasi. |

## Pedoman Jawaban Esai

### 21. CrashLoopBackOff

Jawaban baik memuat `current-context`, `get pod -o wide`, `describe pod`, events terurut, `logs`, `logs --previous`, dan bila hidup `exec`. Mereka membedakan crash command/config/dependency/probe/OOM; perubahan dilakukan pada manifest deklaratif; validasi memakai rollout status dan restart/readiness.

### 22. Resource dan HPA

Request dipakai scheduler; CPU limit dapat throttle; memory limit dapat OOM; QoS diturunkan dari request/limit; HPA membaca metrics dan CPU utilization terhadap request. HPA tidak memperbaiki image pull, selector/endpoint, database down, memory leak yang menggandakan Pod rusak, atau bottleneck storage/network.

### 23. Snapshot/restore

Jawaban harus memiliki context gate, quorum, disk/service check, help/version, snapshot path, metadata/checksum, remote retention, RPO/RTO, approval, disposable cluster, validasi API/nodes/object/workload, serta larangan menjalankan cluster-reset pada production aktif. Mereka menyebut PV/database perlu backup terpisah.

### 24. Rolling upgrade

Jawaban harus membatasi satu node, review release/architecture, backup, PDB/replica/local PV, cordon, drain, upgrade role yang benar, journal/version/readiness check, uncordon, observation, quorum safety, stop condition, dan keputusan abort/rollback. Mereka tidak mencetak token dan tidak memakai force tanpa approval.

### 25. VIP timeout

Jawaban baik memeriksa IP allocation/Service status, speaker dan ARP/NDP, node listener, Service selector/targetPort, EndpointSlice, Pod readiness/probe, NetworkPolicy, dan HTTP application. Mereka membedakan control-plane health dari data-plane path dan mencatat before/after.

## Skoring

- Pilihan ganda: 20 soal × 3 poin = 60.
- Esai: 5 soal × 8 poin = 40.
- Total: 100.
- Lulus: **≥80** dan tidak melakukan operasi berbahaya pada context yang salah.

## Catatan SRE

Jawaban teknis yang benar tetapi tidak memiliki context safety dan blast-radius control belum memenuhi standar operator.

## Kaitan dengan Modul Berikutnya

Setelah lulus, lanjutkan ke Fase 3 OpenTofu dan gunakan runbook ini sebagai kandidat resource/change automation, tanpa mengganti pemahaman manual dengan blind automation.
