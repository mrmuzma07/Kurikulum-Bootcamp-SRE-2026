# Kuis & Kunci Jawaban — Modul 2.1 Konsep & k3d untuk Latihan

> **Cara kerja:** jawab sendiri tanpa buka catatan. Cek di bagian bawah. Target ≥ 80% (16 dari 20).

---

## Bagian A — Pilihan Ganda (12 soal)

**1. Komponen yang menyimpan semua state cluster (sumber kebenaran) adalah...
- A. kube-apiserver
- B. kubelet
- C. etcd
- D. kube-scheduler

**2. Saat `kubectl get pods`, dari mana informasi itu datang?
- A. kubelet di tiap node
- B. etcd, dibaca via kube-apiserver
- C. container runtime
- D. kube-proxy

**3. Peran kubelet adalah...
- A. menyimpan state cluster
- B. agen di node yang memastikan container jalan sesuai Pod spec
- C. route traffic Service ke Pod
- D. pilih node untuk Pod baru

**4. Kenapa jarang membuat Pod langsung di production?
- A. Pod tidak bisa di-restart
- B. Deployment mengelola replica, rollout, self-healing — Pod telanjang tidak
- C. Pod lebih berat dari container
- D. Pod tidak punya IP

**5. Service `type: ClusterIP` dapat diakses dari...
- A. luar cluster via nodeIP:port
- B. dalam cluster saja (IP+DNS internal)
- C. internet publik
- D. Mac langsung

**6. Bagaimana Service menemukan Pod yang dilayani?
- A. via IP Pod yang hardcode
- B. via label selector (Pod dengan label cocok)
- C. via nama container
- D. via etcd query manual

**7. Ingress lebih baik dari NodePort untuk banyak service HTTP karena...
- A. Ingress otomatis TLS
- B. satu IP/port melayani banyak service via host/path (virtual hosting)
- C. Ingress tidak butuh Service
- D. NodePort hanya untuk TCP

**8. Beda ConfigMap dan Secret adalah...
- A. ConfigMap untuk env, Secret untuk file
- B. Secret base64-encoded (bukan encrypted default), ConfigMap plain text
- C. ConfigMap hanya untuk angka, Secret untuk string
- D. tidak ada beda

**9. Kenapa Secret (base64) TIDAK aman untuk commit ke Git?
- A. base64 bisa di-decode siapa saja; bukan encryption
- B. Secret hilang saat push
- C. Git menolak file Secret
- D. Secret tidak bisa version-controlled

**10. PVC (PersistentVolumeClaim) berguna untuk...
- A. membatasi CPU Pod
- B. storage persisten — Pod mati ≠ data hilang
- C. ekspos Service
- D. routing Ingress

**11. k3d menjalankan k3s sebagai...
- A. VM di OrbStack
- B. container Docker
- C. bare-metal proses
- D. pod di cluster lain

**12. Status Pod `CrashLoopBackOff` berarti...
- A. image tidak ada
- B. container start lalu crash, berulang (app error di startup / config salah)
- C. Pod menunggu resource
- D. Pod selesai normal

## Bagian B — Esai (8 soal)

**13.** Jelaskan jalur lengkap `kubectl apply -f pod.yaml` sampai Pod `Running` (sebut 4 komponen/control yang dilalui).

**14.** Pod status `ImagePullBackOff`. Sebut 3 kemungkinan penyebab & cara cek masing-masing.

**15.** Kenapa k3d cocok untuk latihan harian tetapi **tidak ideal** untuk simulasi MetalLB L2 mode? (aspek teknis)

**16.** Sebut urutan debug flow saat Pod `CrashLoopBackOff` (5 langkah). Kenapa `kubectl logs --previous` penting?

**17.** Anda punya context `k3d-lab` (latihan) & `k3s-prod` (production). Sebut 2 kebiasaan agar tidak salah `kubectl delete` di production.

**18.** ConfigMap di-mount sebagai **env** vs sebagai **file** (volume) — beda perilakunya saat ConfigMap diubah? Mana yang auto-reload?

**19.** Exit code 137 (OOMKilled) terjadi pada Pod. Jelaskan apa itu, penyebab umum, & hubungannya dengan konsep cgroup di Modul 1.1.

**20.** Service `type: LoadBalancer` di cloud otomatis dapat IP dari provider. Di on-prem tanpa MetalLB, apa yang terjadi pada Service tsb? (pengantar Modul 2.3 — apa solusinya?)

---

## Kunci Jawaban

### Bagian A

**1. C** — etcd menyimpan semua state cluster (objek, secret, konfig) — sumber kebenaran. Tanpa etcd, cluster kehilangan ingatan.

**2. B** — `kubectl get pods` bertanya ke kube-apiserver, yang membaca dari etcd. Tidak "melihat Pod di node" langsung.

**3. B** — kubelet = agen di tiap node; pantau Pod, pastikan container jalan sesuai spec, laporkan status ke API server. Bukan simpan state (etcd), bukan route (kube-proxy), bukan schedule (scheduler).

**4. B** — Deployment mengelola replica, rollout (versi baru), rollback, self-healing (Pod mati → buat baru). Pod telanjang tidak punya orkestrasi ini. (Pod bisa restart di dalam, tapi tidak ada jaminan replica.)

**5. B** — ClusterIP (default) hanya dari dalam cluster (IP+DNS internal). NodePort = dari luar via nodeIP:port; LoadBalancer = IP eksternal.

**6. B** — Service pakai label selector untuk menemukan Pod (`selector: {app: app}`). Kalau label Pod tidak cocok, endpoints kosong.

**7. B** — Ingress = HTTP layer 7; satu IP/port melayani banyak service via host header / path. Tanpa Ingress, tiap service butuh NodePort/LoadBalancer sendiri.

**8. B** — Secret = data sensitif, base64-encoded default (bukan encrypted); ConfigMap = plain text. Keduanya bisa env atau file. Beda utama: Secret ditujukan sensitif (tapi base64 ≠ aman).

**9. A** — base64 adalah encoding, bukan encryption; siapa saja bisa decode. Untuk aman di Git, pakai Sealed-Secret/SOPS/Vault (enkripsi saat commit, dekrip di cluster). Fase 6.

**10. B** — PVC = permintaan storage persisten; PV = storage di cluster. Pod mati/restart dengan PVC → data tetap. Tanpa PVC → data hilang (ephemeral).

**11. B** — k3d menjalankan k3s (server+agent) sebagai container Docker — sebabnya cepat & ringan, `delete` bersih total.

**12. B** — CrashLoopBackOff = container start lalu crash, diulang (back-off). Penyebab: app error di startup, env/config salah, livenessProbe gagal. Bukan image-hilang (itu ImagePullBackOff) atau Pending (resource).

### Bagian B

**13.** Jalur:
1. `kubectl apply -f` → (HTTPS via kubeconfig) → **kube-apiserver**
2. apiserver validasi + **tulis ke etcd** (desired state)
3. **kube-controller-manager** (Deployment controller) lihat perubahan → buat ReplicaSet → buat Pod spec
4. **kube-scheduler** pilih node untuk Pod → tulis nodeName ke etcd
5. **kubelet** (di node terpilih) watch → terjemah Pod spec → **container runtime (containerd)** → container jalan
6. kubelet laporkan status → apiserver → etcd (Running)
(Karena watch-based/asynchronous, ada delay antara apply & Running.)

**14.** Tiga penyebab ImagePullBackOff:
- **Image/tag salah** (`nginx:notexist`) — cek `kubectl describe pod`, Events: "Failed to pull image ... not found"; verifikasi tag di registry.
- **Registry private tanpa imagePullSecret** — Events: "no creds"; buat `imagePullSecret` + tambah `imagePullSecrets` di Pod spec; cek PAT scope `read_registry`.
- **Arch mismatch** (image amd64-only di host arm64, atau sebaliknya) — Events: "exec format error" atau "no matching manifest"; cek `docker manifest inspect <img>` / `imagetools inspect`; pakai image multi-arch (Fase 1).

**15.** MetalLB L2 bekerja via **ARP/NDP broadcast** di jaringan nyata untuk mengiklankan service IP ke node. k3d jalan sebagai **container** dengan IP container di network Docker — ARP di dalam container tidak terpropagasi ke jaringan fisik Mac seperti server sungguhan. VM OrbStack dengan IP stabil di jaringan nyata bisa melakukan ARP/MetalLB L2 dengan benar. Jadi k3d = cepat untuk objek K8s, tapi tidak untuk simulasi bare-metal LB. (Pakai k3s di VM — Modul 2.2/2.3.)

**16.** Debug flow:
1. `kubectl get pod <name>` → status (CrashLoopBackOff? berapa restart?)
2. `kubectl describe pod <name>` → **Events** di bawah (Back-off restarting, livenessProbe failed, OOMKill, dll)
3. `kubectl logs <name>` → output app saat ini
4. `kubectl logs <name> --previous` → **log container sebelum crash** (kenapa mati)
5. `kubectl exec -it <name> -- sh` → masuk, cek env/config/koneksi; resolve & apply lagi
`--previous` penting karena container yang crash sudah restart — `logs` tanpa flag hanya log container **baru** (mungkin kosong/awal); `--previous` beri log container yang **menyebabkan** crash.

**17.** Dua kebiasaan:
- **Cek context sebelum perintah destruktif**: `kubectl config current-context` sebelum `delete`/`edit`; atau buat alias `kubectl-current-ns`/prompt yang tampilkan context+namespace.
- **Namespace & RBAC**: jalankan latihan di namespace `lab`/`demo`, bukan `default`/`kube-system`; production pakai akun/serviceAccount terbatas (tidak admin cluster) di context `k3s-prod`. (Boleh juga: `kubectl auth can-i delete pods -n prod` untuk verifikasi.)
- (Bonus: jangan pakai `--all-namespaces` sembarangan; `-A` pada `delete` sangat berbahaya.)

**18.** ConfigMap:
- **env** (`envFrom`/`env`): Pod baca saat start; **tidak auto-reload** saat ConfigMap diubah. Butuh `kubectl rollout restart deploy/app` agar Pod restart & baca env baru.
- **file** (volumeMount + `configMap` volume): kubelet **auto-reload** file (symlink update periodik), tapi **app harus watch file** sendiri untuk re-load (banyak app tidak; tetap butuh restart untuk apply). Jadi: file lebih dinamis (file berubah), tapi perilaku app tetap tentukan apakah berubah efektif. Untuk config yang sering ganti, mount sebagai file + app hot-reload; untuk env, restart Pod.

**19.** Exit 137 = 128 + 9 (SIGKILL). Pada Pod = **OOMKilled**: container melampaui memory `limits` → cgroup kernel membunuh proses (OOM killer). Penyebab: memory limit terlalu kecil, app memory leak, atau spike beban. Hubungan Modul 1.1: cgroup (container limit memory) sama konsep — sekarang di level orkestrator (`resources.limits.memory` di Pod = cgroup yang dibuat kubelet). `OOMKilled` Pod = `docker` OOM pada container (Fase 1) tapi dikelola K8s (Pod restart sesuai restartPolicy, CrashLoopBackOff kalau terus OOM). Solusi: naikkan limit, perbaiki leak, atau turunkan beban.

**20.** Di cloud, `type: LoadBalancer` memicu cloud-controller-manager minta IP dari provider (ELB, dsb). Di **on-prem tanpa MetalLB**, Service `type: LoadBalancer` akan **Pending** selamanya (tidak ada yang beri IP eksternal) — `<pending>` di `EXTERNAL-IP`. Solusi on-prem: **MetalLB** (Modul 2.3) — ia mengisi kekosongan dengan meng-assign IP dari pool (L2 via ARP, atau BGP) dan mengiklankan ke node, sehingga Service LoadBalancer dapat IP & traffic sampai. (Sebelum MetalLB, harus disable ServiceLB bawaan k3s — klipper — dulu.)

---

## Penilaian

| Benar | Skor |
|---|---|
| 18–20 | Expert — siap Modul 2.2 (k3s production) |
| 16–17 | Lulus — boleh lanjut, perbaiki yang salah |
| 12–15 | Belum lulus — ulang materi, kerjakan ulang lab |
| < 12 | Ulangi semua materi, lanjut mentor |