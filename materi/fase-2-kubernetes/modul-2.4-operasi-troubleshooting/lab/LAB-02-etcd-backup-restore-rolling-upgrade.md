# LAB-02 — Backup/Restore etcd dan Rolling Upgrade k3s

> **Lane:** k3s di OrbStack Machine. Lab ini memiliki dua bagian terpisah: backup/restore pada cluster disposable dan rolling maintenance/upgrade pada cluster lab yang disetujui.

## Tujuan

- Membuktikan snapshot embedded etcd dengan metadata dan checksum.
- Menguji restore hanya pada cluster disposable sesuai prosedur release.
- Menjalankan rolling maintenance satu node pada satu waktu.
- Memahami quorum, PDB, drain, stop condition, dan external Service validation.

## Guardrail Wajib

- Jangan memakai context production.
- Jangan menjalankan `k3s server --cluster-reset` atau restore snapshot pada cluster aktif.
- Jangan menjalankan `kubectl drain` sebelum PDB, replica, local PV, dan maintenance window direview.
- Jangan menyimpan token k3s, kubeconfig, Secret, atau PAT pada report/Git.
- Jika cluster disposable atau prosedur restore tidak tersedia, lakukan walkthrough + evidence plan; jangan improvisasi restore.

## Prasyarat dan Topologi

Ideal:

```text
k3s-cp1  server/etcd
k3s-cp2  server/etcd
k3s-cp3  server/etcd
k3s-w1   agent
k3s-w2   agent
```

Minimum untuk walkthrough backup adalah satu server k3s dengan embedded etcd. Untuk quorum exercise, gunakan tiga server. Pastikan VM memiliki resource dan IP yang terdokumentasi.

```bash
kubectl config current-context
kubectl config get-contexts
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'
```

Di setiap server:

```bash
sudo systemctl is-active k3s
k3s --version
sudo df -h /var/lib/rancher/k3s/server/db
```

## Bagian A — Snapshot dan Verifikasi

### A1. Baseline dan Marker Object

Buat marker non-secret agar dapat diverifikasi setelah restore:

```bash
kubectl create namespace sre24-restore-check
kubectl create configmap before-snapshot -n sre24-restore-check \
  --from-literal=created-by=lab-02 \
  --from-literal=purpose=restore-validation
kubectl get ns sre24-restore-check
kubectl get configmap before-snapshot -n sre24-restore-check -o yaml
```

Simpan timestamp, versi k3s, nodes, dan output API ready. Jangan memasukkan Secret ke marker.

### A2. Preflight dan Snapshot

Dari Mac:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A
```

Di server yang menjalankan embedded etcd:

```bash
sudo systemctl status k3s --no-pager
sudo df -h /var/lib/rancher/k3s/server/db
sudo k3s etcd-snapshot --help
```

Gunakan sintaks release yang tampil pada help. Pola lab umum:

```bash
sudo k3s etcd-snapshot save --name sre24-$(date -u +%Y%m%dT%H%M%SZ)
```

Jangan melanjutkan jika command, versi, atau path berbeda dari dokumentasi release tanpa review instruktur.

### A3. Artifact Evidence

Cari path aktual dan catat metadata:

```bash
sudo find /var/lib/rancher/k3s/server/db -maxdepth 3 \
  \( -type f -name '*.db' -o -type f -name '*.snap' \) -print | sort
sudo ls -lah /var/lib/rancher/k3s/server/db/snapshots/ 2>/dev/null || true
sudo sha256sum /var/lib/rancher/k3s/server/db/snapshots/<snapshot-file>
```

Isi evidence:

```text
cluster/context:
server:
versi k3s:
snapshot name:
path:
size:
permission/owner:
timestamp UTC:
sha256:
remote copy/location:
retention:
```

Pastikan copy remote/encrypted sesuai kebijakan. File lokal VM bukan perlindungan dari kehilangan VM/Mac.

### A4. Scheduled Snapshot Review

Jika schedule sudah dikonfigurasi, verifikasi behavior, bukan hanya flag:

```bash
sudo journalctl -u k3s --since '24 hours ago' --no-pager | grep -i snapshot
sudo find /var/lib/rancher/k3s/server/db/snapshots -maxdepth 1 -type f -printf '%f %s %TY-%Tm-%Td %TH:%TM:%TS\n' | sort
```

Catat interval, retention, alert ketika snapshot gagal, dan lokasi remote. Tidak perlu mengubah schedule pada cluster bersama tanpa change approval.

## Bagian B — Restore pada Cluster Disposable

### B1. Gate Sebelum Restore

Restore hanya boleh diteruskan jika semua kotak tercentang:

```text
[ ] context/nama cluster disposable diverifikasi dua operator
[ ] tidak melayani user atau workload penting
[ ] snapshot checksum cocok
[ ] versi k3s dan prosedur release direview
[ ] console/SSH dan rebuild plan tersedia
[ ] impact, approval, dan stop condition tercatat
[ ] snapshot/current state boleh digantikan
```

Jika satu saja tidak terpenuhi, berhenti dan buat restore runbook tanpa eksekusi.

### B2. Restore Walkthrough/Execution Terisolasi

Mulai dari help dan dokumentasi versi aktual:

```bash
sudo k3s etcd-snapshot --help
sudo k3s server --help | grep -A3 -B3 cluster-reset
```

Ikuti prosedur resmi release untuk:

1. menghentikan k3s pada target disposable;
2. menempatkan snapshot dengan permission yang benar;
3. menjalankan restore/cluster reset dengan flag yang tepat;
4. menyalakan service dan menunggu API ready;
5. mengubah/rejoin membership server bila prosedur memerlukannya.

Command restore tidak ditulis sebagai copy-paste generik karena flag/path dan dampak peer bergantung pada versi k3s. Jangan pernah mengubah cluster aktif hanya untuk menyelesaikan checklist lab.

### B3. Validasi Restore

```bash
kubectl config current-context
kubectl get --raw='/readyz?verbose'
kubectl get nodes -o wide
kubectl get ns sre24-restore-check
kubectl get configmap before-snapshot -n sre24-restore-check -o yaml
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Catat bahwa restore mengembalikan state pada waktu snapshot. Object yang dibuat setelah snapshot dapat hilang; itu bukan bug. Validasi aplikasi, database, dan PV secara terpisah karena tidak dipulihkan otomatis oleh etcd snapshot.

### B4. Restore Drill Record

```text
snapshot dibuat UTC:
restore dimulai UTC:
API ready UTC:
node ready UTC:
marker object valid:
RPO aktual:
RTO aktual:
object yang hilang/expected:
PV/database validation:
blocker/action item:
```

## Bagian C — Rolling Maintenance dan Upgrade

Bagian ini dapat dilakukan pada cluster lab yang melayani workload latihan, tetapi tetap bukan production. Jika tidak ada versi target yang disetujui, lakukan maintenance drill menggunakan cordon/drain/uncordon dan walkthrough upgrade.

### C1. Baseline Workload dan PDB

Deploy minimal workload dua replica atau gunakan workload lab yang sudah ada. Pastikan selector PDB cocok:

```bash
kubectl get deploy -A
kubectl get pods -A -o wide
kubectl get pdb -A -o wide
kubectl get svc,endpointslice -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Contoh PDB untuk `sre-lab/web`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web
  namespace: sre-lab
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: web
```

Sebelum maintenance, pastikan lebih dari satu Pod Ready dan tidak ada migration/stateful operation yang sedang berjalan.

### C2. Preflight Upgrade

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'
kubectl get pdb -A -o wide
kubectl get pods -A -o wide
```

Pada semua node:

```bash
ssh k3s-cp1 'k3s --version; sudo systemctl is-active k3s; sudo df -h; sudo free -h'
ssh k3s-cp2 'k3s --version; sudo systemctl is-active k3s; sudo df -h; sudo free -h'
ssh k3s-cp3 'k3s --version; sudo systemctl is-active k3s; sudo df -h; sudo free -h'
```

Catat release notes, versi target, image ARM64, snapshot, quorum, PDB, stop condition, dan rollback owner.

### C3. Satu Node Server

Pilih satu server yang aman untuk maintenance. Jangan kehilangan quorum.

```bash
kubectl cordon <server-node>
kubectl get node <server-node>
kubectl drain <server-node> --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

Jika drain tertahan, jangan menambahkan `--force` secara otomatis. Periksa:

```bash
kubectl get pods -A --field-selector spec.nodeName=<server-node> -o wide
kubectl get pdb -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Lakukan upgrade k3s menggunakan artifact/channel yang telah disetujui. Untuk latihan, pattern installer harus direview dahulu:

```bash
ssh <server-node> 'curl -sfL https://get.k3s.io -o /tmp/install-k3s.sh && \
  grep -q "^#!/" /tmp/install-k3s.sh && \
  sudo INSTALL_K3S_VERSION=<versi-target> sh /tmp/install-k3s.sh server'
```

Validasi:

```bash
ssh <server-node> 'k3s --version; sudo systemctl is-active k3s; sudo journalctl -u k3s --since "10 minutes ago" --no-pager'
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'
kubectl get pods -A -o wide
```

Jika API, node, quorum, dan workload sehat:

```bash
kubectl uncordon <server-node>
kubectl get node <server-node>
```

Tunggu observation window dan catat evidence sebelum memilih server berikutnya. Untuk tiga server, jangan membuat dua server unavailable bersamaan.

### C4. Satu Node Agent

```bash
kubectl cordon <worker-node>
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data --timeout=10m
ssh <worker-node> 'sudo systemctl status k3s-agent --no-pager'
```

Upgrade role agent menggunakan prosedur yang sama dan konfigurasi yang sudah ada, tanpa mencetak token:

```bash
ssh <worker-node> 'curl -sfL https://get.k3s.io -o /tmp/install-k3s.sh && \
  grep -q "^#!/" /tmp/install-k3s.sh && \
  sudo INSTALL_K3S_VERSION=<versi-target> sh /tmp/install-k3s.sh agent'
```

```bash
ssh <worker-node> 'k3s --version; sudo systemctl is-active k3s-agent; sudo journalctl -u k3s-agent --since "10 minutes ago" --no-pager'
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl uncordon <worker-node>
```

### C5. External Service Validation

Jika cluster memiliki Service/MetalLB dari Modul 2.3:

```bash
kubectl get svc -A
kubectl get endpointslice -A -o wide
curl --max-time 5 -v http://<VIP-yang-diizinkan>/
```

Validasi VIP, EndpointSlice, readiness, HTTP status, dan latency. Node `Ready` saja tidak membuktikan jalur traffic sehat.

## Bagian D — Stop Condition dan Evidence

Berhenti bila:

- API tidak ready atau quorum terancam;
- upgraded node tidak `Ready`;
- PDB/replica tidak terpenuhi;
- error rate/latency melewati threshold;
- workload stateful/local PV tidak aman dipindahkan;
- log menunjukkan kerusakan atau disk penuh;
- artifact/backup tidak dapat diverifikasi.

Simpan:

```text
change ID/context:
node dan role:
versi sebelum/sesudah:
cordon/drain time:
PDB/replica evidence:
API/node readiness:
workload/HTTP evidence:
journal/event summary:
stop condition:
keputusan lanjut/abort:
operator/approval:
```

## Cleanup

Setelah semua node dan workload sehat:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pdb -A
kubectl get svc,endpointslice -A
```

Hapus marker hanya pada namespace lab:

```bash
kubectl delete namespace sre24-restore-check
```

Jangan menghapus namespace yang bukan dibuat oleh lab.

## Acceptance Criteria LAB-02

- [ ] Context dan topologi k3s diverifikasi sebelum operasi.
- [ ] Marker object, baseline, versi, quorum, dan PDB dicatat.
- [ ] Snapshot etcd dibuat, dicari pada path aktual, dan diverifikasi checksum/metadata.
- [ ] Restore dieksekusi atau didemokan hanya pada cluster disposable dengan approval.
- [ ] API, nodes, object, dan RPO/RTO divalidasi setelah restore.
- [ ] Cordon/drain/upgrade/uncordon dilakukan satu node pada satu waktu atau dibuat walkthrough yang lengkap.
- [ ] Server tidak kehilangan quorum; worker kembali `Ready`.
- [ ] Service/EndpointSlice/VIP diuji jika tersedia.
- [ ] Stop condition, abort, rollback decision, dan evidence ditulis.
- [ ] Tidak ada credential plain text di report/Git.

## Catatan SRE

Maintenance yang baik membuat perubahan kecil terlihat: satu node, satu versi, satu observation window, satu keputusan. Jika operator tidak bisa menjelaskan kapan harus berhenti, prosedur belum siap dijalankan.

## Kaitan dengan Modul Berikutnya

Runbook dan evidence dari lab ini akan menjadi kandidat automation pada Fase 4 Ansible, values/PodDisruptionBudget di Fase 5 Helm, dan change promotion melalui GitOps pada Fase 6.
