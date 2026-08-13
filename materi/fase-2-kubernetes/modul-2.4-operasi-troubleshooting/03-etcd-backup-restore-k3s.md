# 03 — Snapshot & Restore Embedded etcd k3s

> etcd menyimpan state control plane. Snapshot membantu pemulihan state Kubernetes, tetapi bukan pengganti backup aplikasi, database, atau PV.

## Tujuan

- Memahami embedded etcd dan quorum k3s.
- Membuat snapshot manual/terjadwal dengan path dan permission yang benar.
- Memverifikasi file snapshot, ukuran, timestamp, dan checksum.
- Memahami restore sebagai prosedur cluster-level yang hanya diuji pada cluster disposable.
- Menentukan RPO/RTO dan evidence backup.

## 1. Apa yang Dilindungi Snapshot?

Embedded etcd menyimpan state Kubernetes seperti objek API, Deployment, Service, Secret, dan metadata. Snapshot **tidak otomatis berisi** data aplikasi di:

- volume lokal/PV;
- database eksternal;
- registry/image;
- konfigurasi router, MetalLB pool reservation, atau DNS eksternal;
- kubeconfig/token yang disimpan di luar state etcd.

Maka disaster recovery perlu mencakup etcd, PV/database, image, credential management, dan prosedur operator.

## 2. Quorum dan Failure Domain

Untuk embedded etcd jumlah server sebaiknya ganjil:

| Server etcd | Toleransi kehilangan server |
|---:|---:|
| 1 | 0 |
| 3 | 1 |
| 5 | 2 |

Quorum `floor(n/2)+1`. Kehilangan quorum membuat cluster tidak dapat menerima perubahan state secara normal. Worker yang hidup tidak berarti control plane sehat.

```bash
kubectl config current-context
kubectl get nodes -o wide
# Jalankan di server k3s bila diperlukan:
sudo systemctl status k3s --no-pager
sudo journalctl -u k3s --since '15 minutes ago' --no-pager
```

## 3. Preflight Snapshot

Snapshot adalah perubahan operasional yang harus memiliki lokasi backup dan retention plan.

```bash
# Dari Mac: pastikan context
kubectl config current-context
kubectl get nodes

# Di k3s server pertama/leader (SSH)
sudo df -h /var/lib/rancher/k3s/server/db
sudo systemctl is-active k3s
sudo test -r /var/lib/rancher/k3s/server/token && echo 'token exists (jangan tampilkan nilainya)'
```

Catat:

```text
cluster/context:
server yang menjalankan snapshot:
waktu UTC:
versi k3s:
status node/server:
path output:
retention:
RPO/RTO:
```

Jangan menyalin token ke laporan.

## 4. Snapshot Manual dengan `k3s etcd-snapshot`

K3s menyediakan subcommand snapshot untuk embedded etcd. Sintaks dapat berbeda antar versi; periksa help dari binary yang terpasang:

```bash
ssh k3s-cp1 'k3s etcd-snapshot --help'
```

Pola umum snapshot manual:

```bash
ssh k3s-cp1 'sudo k3s etcd-snapshot save --name sre-lab-$(date -u +%Y%m%dT%H%M%SZ)'
```

Di beberapa release, snapshot disimpan di:

```text
/var/lib/rancher/k3s/server/db/snapshots/
```

Verifikasi path aktual:

```bash
ssh k3s-cp1 'sudo find /var/lib/rancher/k3s/server/db -maxdepth 3 -type f -name "*.db" -o -name "*.snap" | sort'
ssh k3s-cp1 'sudo ls -lah /var/lib/rancher/k3s/server/db/snapshots/ 2>/dev/null || true'
```

> Jangan mengasumsikan nama flag atau path lintas versi. Output `k3s etcd-snapshot --help` dan dokumentasi release adalah sumber kebenaran.

## 5. Verifikasi Artifact

Contoh pengumpulan metadata tanpa membuka secret:

```bash
ssh k3s-cp1 'sudo find /var/lib/rancher/k3s/server/db/snapshots -maxdepth 1 -type f -printf "%f %s bytes %TY-%Tm-%Td %TH:%TM:%TS\n" | sort'
ssh k3s-cp1 'sudo sha256sum /var/lib/rancher/k3s/server/db/snapshots/<snapshot-file>'
```

Salin snapshot ke lokasi backup terpisah menggunakan kanal yang disetujui, dengan permission terbatas. Contoh hanya untuk lab:

```bash
scp k3s-cp1:/tmp/<snapshot-file> ./backup/  # gunakan path yang benar setelah review
chmod 600 ./backup/<snapshot-file>
sha256sum ./backup/<snapshot-file>
```

Jangan menganggap snapshot lokal pada disk VM sebagai backup yang tahan kehilangan VM/Mac. Untuk production gunakan remote backup, encryption, access control, retention, dan restore drill.

## 6. Scheduled Snapshot

K3s dapat dikonfigurasi untuk snapshot periodik melalui flag server, misalnya:

```text
--etcd-snapshot-schedule-cron="0 */6 * * *"
--etcd-snapshot-retention=5
```

Nama flag dan mekanisme remote upload harus diverifikasi sesuai versi k3s. Jadwal bukan bukti backup berhasil. Monitor:

- file baru sesuai interval;
- ukuran tidak nol;
- checksum/retention;
- error journal;
- salinan remote;
- restore test berkala.

```bash
ssh k3s-cp1 'sudo journalctl -u k3s --since "24 hours ago" --no-pager | grep -i snapshot'
```

## 7. Restore — Hanya Cluster Disposable

Restore etcd mengubah seluruh state control plane dan dapat membuat resource/credential kembali ke waktu snapshot. **Jangan uji restore pada cluster production aktif, cluster yang melayani user, atau cluster yang tidak memiliki approval tertulis.**

Precondition cluster disposable:

```text
[ ] context dan nama cluster diverifikasi disposable
[ ] semua workload/DB penting dihentikan atau tidak ada
[ ] snapshot checksum cocok
[ ] versi k3s kompatibel
[ ] akses console/SSH tersedia
[ ] rencana rollback/rebuild tersedia
[ ] instruktur/owner menyetujui
```

Sintaks restore bergantung versi k3s. Mulai dengan:

```bash
ssh k3s-cp1 'k3s server --help | grep -A3 -B3 cluster-reset'
ssh k3s-cp1 'k3s etcd-snapshot --help'
```

Pola konseptualnya:

1. Hentikan service k3s sesuai prosedur versi.
2. Pastikan snapshot ada di server target dan checksum cocok.
3. Jalankan restore/cluster reset dengan flag resmi release.
4. Validasi data dan membership server.
5. Start k3s dan tunggu API server sehat.
6. Verifikasi nodes, namespaces, Service, dan workload.
7. Rejoin server lain sesuai prosedur jika membership berubah.

Jangan menulis command `cluster-reset` dari ingatan lalu menjalankannya pada cluster nyata. Flag, path, dan tindakan terhadap peer berbeda menurut versi/distribusi.

## 8. Restore Validation

```bash
kubectl config current-context
kubectl get --raw='/readyz?verbose'
kubectl get nodes -o wide
kubectl get ns
kubectl get deploy,svc -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Validasi yang harus dibuktikan:

- API server ready;
- object yang ada sebelum snapshot kembali sesuai expected;
- perubahan setelah snapshot memang tidak diharapkan ada;
- node/agent reconnect;
- MetalLB/Ingress/Service external path diperiksa bila relevan;
- aplikasi dan data PV diverifikasi secara terpisah.

## 9. RPO dan RTO

- **RPO (Recovery Point Objective):** berapa banyak perubahan data/state yang boleh hilang. Snapshot setiap 6 jam berarti secara teori kehilangan hingga 6 jam state etcd.
- **RTO (Recovery Time Objective):** berapa lama sampai service kembali tersedia. Termasuk copy snapshot, restore, rejoin, dan validasi.

Catat hasil drill:

```text
snapshot dibuat:
insiden simulasi:
restore dimulai:
API ready:
workload tervalidasi:
RPO aktual:
RTO aktual:
blocker:
action item:
```

## 10. Troubleshooting Snapshot

| Gejala | Pemeriksaan |
|---|---|
| snapshot command gagal | `k3s etcd-snapshot --help`, service status, disk space, journal |
| file tidak muncul | path/version flag, permission, cron/service logs |
| file 0 byte/kecil tidak wajar | checksum, log error, jangan anggap valid |
| restore API tidak ready | version, path, cluster membership, journal k3s |
| object kembali ke state lama | itu expected restore behavior; catat RPO dan reconcile setelah recovery |
| worker tidak join | token/address/CA, service k3s-agent, node identity |

## Acceptance Criteria

- [ ] Context k3s dan server target diverifikasi.
- [ ] Snapshot manual dibuat menggunakan sintaks release yang terpasang.
- [ ] File, timestamp, ukuran, permission, dan checksum dicatat.
- [ ] Salinan backup terpisah atau rencana remote backup didokumentasikan.
- [ ] Retention/schedule dan monitoring dijelaskan.
- [ ] Restore hanya dilakukan pada cluster disposable dengan approval.
- [ ] API, nodes, object, dan workload divalidasi setelah restore.
- [ ] RPO/RTO aktual dan action item ditulis.

## Catatan SRE

Backup yang belum pernah direstore hanyalah hipotesis. Restore drill harus aman, terisolasi, terdokumentasi, dan cukup sering untuk menemukan perubahan prosedur sebelum incident nyata.

## Kaitan dengan Modul Berikutnya

- Snapshot menjadi precondition [04-upgrade-k3s-rolling](04-upgrade-k3s-rolling.md).
- Jadwal backup dapat diotomasi Ansible pada Fase 4.
- Remote retention/encryption dan alerting akan terhubung ke Observability serta praktik SRE.
