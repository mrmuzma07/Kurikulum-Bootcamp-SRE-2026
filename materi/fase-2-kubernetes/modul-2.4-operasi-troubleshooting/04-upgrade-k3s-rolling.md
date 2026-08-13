# 04 — Rolling Maintenance & Upgrade k3s

> Upgrade cluster adalah perubahan terkoordinasi pada control plane, agent, workload, dan akses. Targetnya bukan “tidak ada Pod restart”, tetapi availability, quorum, dan rollback yang terukur.

## Tujuan

- Menyusun preflight upgrade k3s berbasis versi dan compatibility.
- Memahami hubungan `cordon`, `drain`, PDB, readiness, dan replica.
- Meng-upgrade server dan agent satu per satu pada cluster lab.
- Menetapkan stop condition, evidence, dan strategi abort/rollback.
- Membedakan rolling upgrade production-like dari eksperimen cepat di k3d.

## 1. Model Perubahan

Jalur perubahan yang aman:

```text
review release/compatibility
  → backup & health check
  → cordon satu node
  → drain sesuai workload/PDB
  → upgrade satu node
  → validasi node/API/workload
  → uncordon
  → observasi
  → lanjut node berikutnya
```

Pada HA embedded etcd, jangan menghentikan server hingga quorum hilang. Tiga server berarti hanya satu server boleh unavailable pada satu waktu jika ingin tetap memiliki quorum. Worker dapat di-upgrade setelah control plane sehat.

`k3d` menggunakan container dan cocok untuk delete/recreate atau latihan rollout. Prosedur systemd, etcd, drain, dan versi binary lebih realistis dilakukan pada k3s VM OrbStack.

## 2. Preflight dan Change Record

Sebelum menyentuh node, verifikasi context dan simpan baseline:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'
kubectl get pods -A -o wide
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Di setiap node server:

```bash
ssh k3s-cp1 'k3s --version; sudo systemctl is-active k3s; sudo df -h; sudo free -h'
ssh k3s-cp2 'k3s --version; sudo systemctl is-active k3s; sudo df -h; sudo free -h'
ssh k3s-cp3 'k3s --version; sudo systemctl is-active k3s; sudo df -h; sudo free -h'
```

Catat:

```text
change ID:
context/cluster:
versi awal per node:
versi target:
release notes dan compatibility:
maintenance window:
operator/approval:
snapshot name/checksum:
PDB dan replica kritis:
stop condition:
rollback/abort plan:
```

Jangan menampilkan token k3s, kubeconfig, Secret, atau PAT dalam evidence.

## 3. Compatibility dan Backup

Sebelum upgrade:

- baca release notes dan upgrade notes versi target;
- cek dukungan arsitektur `linux/arm64` untuk lab M5;
- pastikan versi server/agent dan komponen addon kompatibel;
- pastikan disk, memory, dan konektivitas antar-server cukup;
- buat dan verifikasi snapshot etcd sesuai [03-etcd-backup-restore-k3s](03-etcd-backup-restore-k3s.md);
- siapkan akses console jika SSH/network gagal;
- pastikan image workload memiliki manifest ARM64;
- bekukan perubahan non-esensial selama window.

Backup etcd bukan rollback binary otomatis. Restore state dan downgrade binary adalah keputusan berbeda yang harus mengikuti prosedur release.

## 4. PDB, Replica, dan Drain

PodDisruptionBudget membatasi voluntary disruption, tetapi tidak menjamin zero downtime. Pastikan:

```bash
kubectl get pdb -A -o wide
kubectl get deploy,statefulset -A
kubectl get pods -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName,READY:.status.containerStatuses[*].ready
```

Untuk workload lab, contoh PDB:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web
  namespace: sre-lab
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

Pastikan selector PDB benar-benar cocok dengan Pod. PDB yang terlalu ketat dapat membuat drain tertahan; PDB yang salah selector tidak melindungi workload.

Drain hanya satu node dengan target eksplisit:

```bash
kubectl cordon <node>
kubectl get node <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

Sebelum menjalankan drain, periksa hasil `kubectl drain --help` dan kebijakan cluster. Jangan menambahkan `--force` atau menghapus PVC secara membabi buta. `emptyDir` bersifat ephemeral dan dapat hilang saat Pod dipindahkan.

Jika drain berhenti pada PDB atau Pod unmanaged:

```bash
kubectl get pods -A --field-selector spec.nodeName=<node> -o wide
kubectl describe pdb <pdb> -n <namespace>
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Hentikan prosedur dan minta keputusan owner bila ada stateful workload, local PV, single replica, atau PDB tidak dapat dipenuhi.

## 5. Upgrade Server Satu per Satu

Urutan umum untuk HA:

1. server non-leader/non-critical terlebih dahulu;
2. validasi API dan quorum;
3. server berikutnya;
4. leader/remaining server terakhir sesuai release guidance.

Pada setiap node:

```bash
kubectl config current-context
kubectl cordon <server-node>
kubectl drain <server-node> --ignore-daemonsets --delete-emptydir-data --timeout=10m
ssh <server-node> 'k3s --version; sudo systemctl status k3s --no-pager'
```

Gunakan metode upgrade yang disetujui organisasi. Installer k3s dapat memakai channel atau `INSTALL_K3S_VERSION`, tetapi jangan menjalankan installer remote tanpa meninjau versi dan source terlebih dahulu. Contoh pola **lab setelah release target disetujui**:

```bash
ssh <server-node> 'curl -sfL https://get.k3s.io -o /tmp/install-k3s.sh && \
  grep -q "^#!/" /tmp/install-k3s.sh && \
  sudo INSTALL_K3S_VERSION=<versi-target> sh /tmp/install-k3s.sh server'
```

> Placeholder `<versi-target>` harus diganti versi yang telah direview. Di production gunakan artifact mirror/checksum dan proses change management, bukan blind `curl | sh`.

Validasi node setelah upgrade:

```bash
ssh <server-node> 'k3s --version; sudo systemctl is-active k3s; sudo journalctl -u k3s --since "10 minutes ago" --no-pager'
kubectl get nodes -o wide
kubectl get --raw='/readyz?verbose'
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Jika sehat:

```bash
kubectl uncordon <server-node>
kubectl get node <server-node>
```

Tunggu observasi sebelum melanjutkan ke server berikutnya.

## 6. Upgrade Agent Satu per Satu

Setelah semua server sehat, upgrade worker dengan pola yang sama:

```bash
kubectl get pods -A -o wide | grep '<worker-node>'
kubectl cordon <worker-node>
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data --timeout=10m
ssh <worker-node> 'sudo systemctl status k3s-agent --no-pager'
```

Jalankan installer dengan role `agent` dan konfigurasi yang sama dengan node tersebut melalui prosedur resmi:

```bash
ssh <worker-node> 'curl -sfL https://get.k3s.io -o /tmp/install-k3s.sh && \
  grep -q "^#!/" /tmp/install-k3s.sh && \
  sudo INSTALL_K3S_VERSION=<versi-target> sh /tmp/install-k3s.sh agent'
```

Jangan menaruh token dalam command yang masuk shell history atau laporan. Pastikan service agent membaca konfigurasi/token dari file permission terbatas yang sudah ada.

Validasi dan kembalikan scheduling:

```bash
ssh <worker-node> 'k3s --version; sudo systemctl is-active k3s-agent; sudo journalctl -u k3s-agent --since "10 minutes ago" --no-pager'
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl uncordon <worker-node>
```

## 7. Observasi Availability dan External Path

Selama rolling upgrade, amati:

```bash
kubectl get nodes -w
kubectl get pods -A -w
kubectl get deploy -A
kubectl get pdb -A -o wide
kubectl get svc -A
kubectl get endpointslice -A
```

Uji aplikasi dengan request berulang yang terukur dari client yang diizinkan:

```bash
for i in $(seq 1 20); do
  date -u +%FT%TZ
  curl --max-time 3 -sk -o /dev/null -w 'http=%{http_code} connect=%{time_connect} total=%{time_total}\n' https://<host-yang-diizinkan>/ || true
  sleep 2
done
```

Untuk Service LoadBalancer, validasi VIP MetalLB dan jalur Service/Endpoint, bukan hanya status node:

```bash
kubectl get svc -A
kubectl get endpointslice -A -o wide
# dari Mac/VM yang berada di jaringan lab:
arp -an | grep '<VIP>' || true
curl --max-time 5 -v http://<VIP>/
```

## 8. Stop Condition dan Abort

Hentikan rolling upgrade dan jangan lanjut node berikutnya bila salah satu terjadi:

- API server tidak ready;
- quorum/control plane hilang;
- node upgraded tidak kembali `Ready`;
- error rate/latency melewati batas change record;
- PDB atau readiness tidak dapat dipenuhi;
- workload penting tidak memiliki replica sehat;
- disk/memory/log menunjukkan kerusakan;
- snapshot atau rollback evidence tidak tersedia.

Tindakan abort:

1. Biarkan node yang sudah sehat tetap `Ready` bila aman.
2. Jangan uncordon node yang belum tervalidasi.
3. Kembalikan traffic/replica sesuai runbook.
4. Simpan journal, events, dan versi binary.
5. Eskalasi untuk keputusan rollback/rebuild/restore.

Jangan melakukan downgrade atau restore etcd secara improvisasi. Jika harus restore, kembali ke prosedur cluster disposable/approved pada Modul 03 dan ikuti release documentation.

## 9. Troubleshooting Upgrade

| Gejala | Pemeriksaan awal | Stop condition |
|---|---|---|
| node `NotReady` | `kubectl describe node`, service status, journal, disk | jangan lanjut node berikutnya |
| Pod tidak pindah | PDB, replica, taint, local PV, `describe pod` | jangan memakai `--force` tanpa approval |
| API timeout | quorum, status semua server, network 6443 | lindungi quorum |
| agent tidak join | service agent, server URL, CA/token file permission | jangan cetak token |
| image gagal start | architecture, logs, events, registry | jangan ganti ke `latest` |
| VIP/HTTP gagal | Service, EndpointSlice, MetalLB speaker/ARP | bedakan control-plane dan data-plane |

## Acceptance Criteria

- [ ] Context, topology, versi, compatibility, approval, dan maintenance window dicatat.
- [ ] Snapshot etcd dibuat dan diverifikasi sebelum perubahan.
- [ ] PDB, replica, local PV, dan quorum diperiksa.
- [ ] Node di-cordon dan drain dengan target eksplisit, bukan seluruh cluster.
- [ ] Server di-upgrade satu per satu dan kembali sehat sebelum lanjut.
- [ ] Agent di-upgrade satu per satu dan kembali `Ready`.
- [ ] Node di-uncordon hanya setelah validasi.
- [ ] API, workload, PDB, Service/EndpointSlice, dan jalur VIP diuji.
- [ ] Stop condition, evidence, abort, dan rollback decision dicatat.
- [ ] Tidak ada token, kubeconfig, PAT, atau Secret plain text di laporan/Git.

## Catatan SRE

Rolling upgrade adalah eksperimen dengan blast radius yang dibatasi. “Selesai install binary” bukan acceptance; acceptance adalah cluster tetap memiliki quorum, workload tetap melayani sesuai SLO, dan operator tahu kapan harus berhenti.

## Kaitan dengan Modul Berikutnya

- Patching dan upgrade ini akan diotomasi dengan Ansible pada Fase 4.
- Manifest PDB, resource, dan HPA akan dipaketkan Helm pada Fase 5.
- Change record, evidence, dan rollback menjadi input GitOps serta praktik SRE Fase 6–8.
