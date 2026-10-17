# 04 — Backup, Health, Evidence, dan Total Cluster Rebuild: Menguasai Pemulihan Bencana On-Premises

> **Ringkasan:** Kualitas seorang *Site Reliability Engineer* (SRE) sejati tidak diukur saat sistem berjalan normal di hari yang cerah, melainkan bagaimana ia merespons ketika bencana katastropik terjadi: ketiga server master tersambar petir, basis data etcd mengalami korupsi data massal, dan aplikasi perusahaan mati total. Di lingkungan *on-premises*, tidak ada tombol ajaib "Restore Snapshot" seperti di cloud publik. SRE harus merancang strategi pencadangan berlapis (*defense-in-depth backups*), mengotomasi pengiriman snapshot ke penyimpanan dingin (*off-site MinIO S3*), menyusun rantai bukti audit (*evidence chain*), serta menguasai prosedur rekonstruksi total klaster dari nol (*Total Cluster Rebuild*) dengan target waktu pemulihan (*RTO*) terukur.

---

## 1. Analogi Pemula: Polis Asuransi Kebakaran & Latihan Gempa Bumi Nyata

Bagi pemula, ada kesalahpahaman umum seputar konsep cadangan (*backup*):
> *"Saya sudah membuat snapshot etcd di dalam folder `/var/lib/rancher/k3s/server/db/snapshots/`. Jadi klaster saya sudah 100% aman dari bencana, kan?"*

Mari gunakan analogi proteksi kebakaran gedung perkantoran:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI ASURANSI BRANKAS vs LATIHAN EVAKUASI BENCANA        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KESALAHAN FATAL: POLIS ASURANSI DI DALAM GEDUNG YANG TERBAKAR ]          │
│  - Anda membeli polis asuransi gedung senilai 100 miliar rupiah.            │
│  - Anda menyimpan dokumen polis tersebut di dalam laci meja kerja Anda di   │
│    lantai 3 gedung yang sama.                                               │
│  - Suatu malam, gedung tersebut terbakar habis rata dengan tanah.           │
│  - Hasilnya: Dokumen asuransi ikut hangus terbakar bersama gedung!          │
│    Perusahaan asuransi menolak klaim karena Anda tidak punya bukti fisik!   │
│                                                                             │
│  [ STANDAR SRE: CADANGAN DILUAR LOKASI (OFF-SITE COLD STORAGE) ]            │
│  - Dokumen polis dan salinan sertifikat tanah difotokopi dan disimpan       │
│    di brankas bank berbeda kota (**MinIO S3 Off-Site Object Storage**).      │
│  - Jika gedung fisik terbakar, Anda cukup mengambil salinan dari bank dan    │
│    mulai membangun gedung baru di lahan seberang!                           │
│                                                                             │
│  [ HUKUM KEANDALAN SRE TENTANG PEMULIHAN (RESTORE DRILL) ]                  │
│  "Cadangan yang belum pernah diuji coba proses restore-nya                  │
│   bukanlah sebuah cadangan, melainkan HANYA SEBUAH ANGAN-ANGAN KOSONG!"     │
│                                                                             │
│  SRE profesional secara rutin menguji pembongkaran dan pembangunan kembali   │
│  klaster (*Game Day Rebuild Drill*) di lingkungan simulasi disposable.      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Taksonomi & Batasan Tanggung Jawab Pencadangan (Backup Boundaries)

Di lingkungan *on-premises*, sistem penyimpanan dan basis data terpecah ke dalam beberapa lapisan yang memiliki kepemilikan (*ownership*) dan strategi retensi berbeda:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 EMPAT LAPISAN PENCADANGAN ON-PREMISES                       │
├──────────────────────┬──────────────────────────────┬───────────────────────┤
│ Lapisan Pencadangan  │ Objek yang Disimpan          │ Alat & Lokasi Simpan  │
├──────────────────────┼──────────────────────────────┼───────────────────────┤
│ 1. Cluster State     │ Objek k8s (Pods, Services,   │ `k3s etcd-snapshot`   │
│    (Etcd Raft DB)    │ ConfigMaps, Secrets, CRDs)   │ dikirim ke MinIO S3.  │
├──────────────────────┼──────────────────────────────┼───────────────────────┤
│ 2. Persistent Volume │ Data fisik stateful app      │ Velero + CSI Snapshot │
│    (PV Storage)      │ (MySQL, Postgres, MinIO disk)│ ke SAN/NFS on-prem.   │
├──────────────────────┼──────────────────────────────┼───────────────────────┤
│ 3. Host & IaC State  │ Virtual Machine topology,    │ Git (Kode HCL) +      │
│    (OpenTofu/Ansible)│ network vNIC, IP, disk size  │ MinIO (tofu.tfstate). │
├──────────────────────┼──────────────────────────────┼───────────────────────┤
│ 4. App Definition    │ Helm Charts, values.yaml,    │ GitLab Registry       │
│    (GitOps Manifests)│ CI/CD pipelines, container   │ & Git Version Control.│
└──────────────────────┴──────────────────────────────┴───────────────────────┘
```

> [!WARNING]
> **Snapshot Etcd BUKAN Cadangan Basis Data Aplikasi:**
> Snapshot etcd k3s hanya mencatat metadata deklaratif Kubernetes (*dimana Pod berjalan, port berapa yang dibuka, secret apa yang di-mount*). Snapshot etcd **TIDAK MENCATAT** isi baris tabel transaksi di dalam database MySQL atau PostgreSQL yang tersimpan di Persistent Volume! Pemulihan bencana aplikasi database wajib ditangani melalui alat terpisah seperti Velero atau *database native dumping*.

---

## 3. Otomatisasi Snapshot k3s dengan Ansible & Ekspor ke MinIO S3

Berikut adalah playbook Ansible yang dijadwalkan berjalan setiap malam untuk mengambil snapshot etcd dan mengirimkannya ke penyimpanan objek S3 MinIO on-prem secara aman:

```yaml
---
# playbooks/backup-etcd-s3.yml - Otomatisasi Pencadangan Etcd ke MinIO S3
- name: Pengambilan Snapshot Etcd k3s dan Pengiriman Off-Site
  hosts: k3s_servers[0] # HANYA dieksekusi dari master pertama!
  become: true

  vars:
    backup_timestamp: "{{ ansible_date_time.iso8601_basic_short }}"
    snapshot_filename: "k3s-etcd-prod-{{ backup_timestamp }}.db"
    minio_s3_endpoint: "https://minio.internal.corp:9000"
    minio_bucket_name: "k3s-cluster-backups"

  tasks:
    - name: 1. Ambil snapshot etcd secara lokal di host master
      ansible.builtin.command: >
        k3s etcd-snapshot save
        --name "{{ snapshot_filename }}"
      register: snapshot_result
      changed_when: true

    - name: 2. Dapatkan path absolut file snapshot yang baru dibuat
      ansible.builtin.set_fact:
        local_snapshot_path: "/var/lib/rancher/k3s/server/db/snapshots/{{ snapshot_filename }}"

    - name: 3. Kirim file snapshot ke MinIO S3 On-Premises (Cold Storage)
      amazon.aws.s3_object:
        endpoint_url: "{{ minio_s3_endpoint }}"
        aws_access_key: "{{ vault_minio_access_key }}"
        aws_secret_key: "{{ vault_minio_secret_key }}"
        bucket: "{{ minio_bucket_name }}"
        object: "etcd/{{ snapshot_filename }}"
        src: "{{ local_snapshot_path }}"
        mode: put
      no_log: true # MENCEGAH KREDENSIAL MINIO TERCETAK DI LOG!

    - name: 4. Hapus snapshot lokal yang berumur lebih dari 7 hari (Retensi Disk)
      ansible.builtin.command: >
        k3s etcd-snapshot prune
        --snapshot-retention 7
      changed_when: false

    - name: 5. Catat bukti keberhasilan snapshot untuk audit
      ansible.builtin.debug:
        msg: "Snapshot {{ snapshot_filename }} sukses diunggah ke MinIO S3 ({{ minio_bucket_name }})."
```

---

## 4. Rantai Bukti Audit & Telemetri Kesehatan (The Evidence Chain)

Dalam tata kelola rekayasa keandalan (*SRE Governance*), setelah eksekusi perbaikan atau deployment selesai, SRE wajib mempublikasikan **Berkas Ringkasan Bukti Kesehatan (*Health Evidence Summary*)** yang dapat diaudit oleh tim kepatuhan (*compliance*):

```json
{
  "evidence_metadata": {
    "report_id": "REBUILD-VERIFY-2026-10-17-01",
    "timestamp_utc": "2026-10-17T11:45:00Z",
    "environment": "production-onprem",
    "operator_identity": "sre-core-pipeline@gitlab-runner-01",
    "iac_commit_sha": "8f9b2c1743a19e24f706e25776d6ec1438914619",
    "ansible_version": "core-2.16.5"
  },
  "cluster_telemetry": {
    "kubernetes_version": "v1.31.1+k3s1",
    "total_nodes": 5,
    "ready_nodes": 5,
    "etcd_quorum_status": "HEALTHY",
    "etcd_leader": "sre-k3s-cp-01",
    "etcd_members_count": 3
  },
  "readiness_probes": {
    "api_readyz_endpoint": "HTTP 200 OK",
    "api_livez_endpoint": "HTTP 200 OK",
    "coredns_running": true,
    "metallb_speaker_daemonset_ready": true
  },
  "backup_reference": {
    "last_valid_snapshot": "s3://k3s-cluster-backups/etcd/k3s-etcd-prod-20261017T100000.db",
    "sha256_checksum": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
  }
}
```

---

## 5. SOP Prosedur Rekonstruksi Klaster Total (Total Cluster Rebuild Runbook)

Jika terjadi malapetaka di mana seluruh node master korup atau data center mengalami pemadaman fisik total, ikuti **SOP 7 Langkah Rekonstruksi Klaster** berikut:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 RUNBOOK REKONSTRUKSI KLASTER TOTAL DARI NOL                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ LANGKAH 1: BEKUKAN TRAFIK & SIAPKAN LAHAN INFRASTRUKTUR BARU ]           │
│  - Hentikan routing DNS eksternal atau isolasi IP VIP MetalLB.              │
│  - Jalankan OpenTofu untuk mem-provisioning ulang VM bersih di Proxmox:     │
│    $ tofu apply -auto-approve                                               │
│                                                                             │
│  [ LANGKAH 2: EKSEKUSI HOST READINESS GATE ]                                │
│  - Pastikan seluruh VM baru lolos 8 Gerbang Kesiapan via Ansible:           │
│    $ ansible-playbook -i inventories/onprem playbooks/preflight-readiness.yml│
│                                                                             │
│  [ LANGKAH 3: UNDUH SNAPSHOT TERAKHIR DARI MINIO S3 ]                       │
│  - Salin file snapshot sehat ke node master pertama (`sre-k3s-cp-01`):      │
│    $ aws --endpoint-url https://minio.internal.corp:9000 s3 cp \            │
│      s3://k3s-cluster-backups/etcd/k3s-etcd-prod-latest.db /tmp/snapshot.db │
│                                                                             │
│  [ LANGKAH 4: INISIALISASI RESTORE MASTER PERTAMA (CLUSTER RESET) ]         │
│  - Bersihkan sisa data etcd lama: `rm -rf /var/lib/rancher/k3s/server/db/` │
│  - Eksekusi pemulihan data etcd:                                            │
│    $ k3s server \                                                           │
│      --cluster-reset \                                                      │
│      --cluster-reset-restore-path=/tmp/snapshot.db                          │
│                                                                             │
│  [ LANGKAH 5: NYALAKAN LAYANAN MASTER PERTAMA ]                             │
│  - Jalankan daemon k3s: `systemctl start k3s`                               │
│  - Verifikasi bahwa seluruh Deployment dan Secret lama kembali muncul:      │
│    $ kubectl get deployments -A                                             │
│                                                                             │
│  [ LANGKAH 6: GABUNGKAN MASTER 2, MASTER 3, DAN WORKERS DENGAN ANSIBLE ]   │
│  - Jalankan playbook join multi-node deterministik:                         │
│    $ ansible-playbook -i inventories/onprem playbooks/deploy-k3s-ha.yml \   │
│      --limit 'k3s_servers[1:],k3s_agents'                                   │
│                                                                             │
│  [ LANGKAH 7: UJI VALIDASI LALU LINTAS & RILIS TRAFIK ]                     │
│  - Lakukan smoke-test ingress dan pastikan kuorum etcd sehat sempurna.      │
│  - Hubungkan kembali DNS publik. Insiden selesai!                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pola Kegagalan (Failure Modes) & Diagnosa SRE

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PANDUAN DIAGNOSA KEGAGALAN PEMULIHAN BENCANA                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ KASUS 1: "RESTORE AMNESIA" (DIREKTORI DB ETCD LAMA TIDAK DIBERSIHKAN) ]  │
│  Gejala: Perintah `--cluster-reset` sukses, tapi k3s me-load data lama yang │
│          rusak alih-alih menggunakan data dari file snapshot!               │
│  Akar Masalah:                                                              │
│  - Direktori `/var/lib/rancher/k3s/server/db/etcd` masih berisi file state  │
│    database lama yang corrupt. k3s memprioritaskan database di disk jika    │
│    masih ada sisa data wal file.                                            │
│  Solusi SRE:                                                                │
│  Sebelum menjalankan restore, WAJIB matikan k3s dan bersihkan total folder: │
│  systemctl stop k3s                                                         │
│  rm -rf /var/lib/rancher/k3s/server/db/etcd                                 │
│  k3s server --cluster-reset --cluster-reset-restore-path=/tmp/snapshot.db   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 2: VERSI BINER K3S TIDAK COCOK DENGAN FORMAT SNAPSHOT ]            │
│  Gejala: k3s crash-loop pasca restore: "schema version mismatch".           │
│  Akar Masalah:                                                              │
│  - Snapshot diambil saat klaster berjalan di k3s `v1.28`, tetapi server baru│
│    diinstall k3s versi `v1.31`. Skema internal etcd tidak kompatibel!       │
│  Solusi SRE:                                                                │
│  1. Selalu pin versi k3s yang sama persis saat proses disaster recovery.    │
│  2. Lakukan upgrade versi HANYA SETELAH klaster pulih 100% pada versi lama. │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ KASUS 3: MASTER TAMBAHAN GAGAL BERGABUNG PASCA CLUSTER RESET ]           │
│  Gejala: Master 2 dan 3 gagal start: "cluster ID does not match".           │
│  Akar Masalah:                                                              │
│  - Operasi `--cluster-reset` pada master 1 menghasilkan Cluster ID baru.    │
│    Master 2 dan 3 masih menyimpan Cluster ID lama di disk lokal mereka.     │
│  Solusi SRE:                                                                │
│  Pada Master 2 dan Master 3, hapus total instalasi dan database lokal:      │
│  /usr/local/bin/k3s-uninstall.sh                                            │
│  Lalu gabungkan kembali sebagai node fresh join via Ansible.                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Ringkasan Perintah Penting (SRE Cheatsheet)

```bash
# 1. Mengambil snapshot etcd secara manual dengan nama kustom
k3s etcd-snapshot save --name manual-pre-maintenance.db

# 2. Melihat daftar snapshot yang tersimpan di disk lokal master
k3s etcd-snapshot list

# 3. Menguji konektivitas dan kesehatan API server secara mendalam
kubectl get --raw='/readyz?verbose'

# 4. Melakukan reset klaster dan restore dari berkas snapshot terverifikasi
k3s server --cluster-reset --cluster-reset-restore-path=/tmp/manual-pre-maintenance.db

# 5. Menghapus instalasi k3s secara tuntas untuk persiapan rebuild fresh
/usr/local/bin/k3s-uninstall.sh        # Pada Server Control-Plane
/usr/local/bin/k3s-agent-uninstall.sh  # Pada Worker Node
```

---

## 8. Pertanyaan Reflektif & Uji Pemahaman Mandiri

### Soal 1 (Pilihan Ganda)
Mengapa menyimpan berkas snapshot etcd hanya di dalam direktori lokal master (`/var/lib/rancher/k3s/...`) merupakan pelanggaran fatal terhadap prinsip *Disaster Recovery* (DR) SRE?
- A. Karena harddisk server akan cepat penuh dalam waktu 1 jam.
- B. Karena jika server fisik atau VM master tersebut mengalami kehancuran perangkat keras (misal: storage SAN terbakar), file cadangan tersebut akan ikut musnah bersama sistem yang dilindunginya (*Single Point of Failure*).
- C. Karena k3s otomatis menghapus file snapshot setiap 24 jam.
- D. Agar file snapshot tidak terbaca oleh Kubelet.

*Kunci Jawaban:* **B**. Prinsip dasar pencadangan mewajibkan salinan data dikirimkan ke lokasi fisik yang berbeda (*Off-site storage / MinIO S3*) untuk mengantisipasi kehancuran total pada lokasi utama.

### Soal 2 (Pilihan Ganda)
Sebelum mengeksekusi perintah pemulihan darurat `k3s server --cluster-reset --cluster-reset-restore-path=...`, tindakan krusial apa yang wajib dilakukan pada direktori `/var/lib/rancher/k3s/server/db/etcd`?
- A. Mengubah izin berkas menjadi chmod 777.
- B. Menghapus atau memindahkan seluruh isi folder etcd lama yang korup agar k3s menginisialisasi basis data konsensus baru yang bersih dari file snapshot.
- C. Menyalin file kubeconfig ke dalam folder tersebut.
- D. Mengompresi folder tersebut ke format zip.

*Kunci Jawaban:* **B**. Menghapus residu file etcd lama adalah syarat mutlak agar k3s tidak terjebak dalam kondisi *amnesia* atau konflik *write-ahead log* (WAL).

### Soal 3 (Pilihan Ganda)
Setelah master pertama (`cp-01`) berhasil dipulihkan dari snapshot menggunakan opsi `--cluster-reset`, mengapa master kedua (`cp-02`) dan master ketiga (`cp-03`) yang lama tidak bisa langsung dinyalakan begitu saja?
- A. Karena kabel jaringan master kedua otomatis dicabut oleh sistem operasi.
- B. Karena operasi cluster-reset menerbitkan ID klaster kriptografis yang baru; master kedua dan ketiga masih membawa identitas keanggotaan klaster lama dan harus di-reset bersih sebelum bergabung kembali.
- C. Karena master kedua otomatis berubah menjadi server DHCP.
- D. Karena k3s hanya mengizinkan satu master setelah proses restore.

*Kunci Jawaban:* **B**. `--cluster-reset` melahirkan generasi identitas klaster baru (*new cluster epoch*), sehingga anggota lama harus dibersihkan secara tuntas agar bisa masuk kembali sebagai anggota baru.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Pada pukul 02.00 dini hari, terjadi lonjakan arus listrik di data center on-premise yang mengakibatkan seluruh 3 server master k3s mati mendadak. Setelah listrik kembali menyala, teknisi mendapati bahwa master 1 dan master 2 mengalami *unrecoverable disk failure*, sementara master 3 masih hidup namun status etcd melaporkan `corrupted wal file`. Anda memiliki snapshot sehat di MinIO S3 yang diambil pada pukul 00.00. Susun rencana aksi (*Action Plan*) 5 tahap untuk memulihkan klaster ke status operasional!

*Panduan Solusi:*
1. **Penyelamatan & Provisioning (OpenTofu):**
   Minta hypervisor membuat 2 VM baru untuk menggantikan master 1 dan 2 yang rusak fisik, serta perbaiki disk pada master 3 (atau buat 3 VM baru bersih sekaligus).
2. **Pengunduhan Snapshot Off-site:**
   Unduh file snapshot `k3s-etcd-prod-20261017T000000.db` dari MinIO S3 ke server yang ditetapkan sebagai master bootstrap baru (`sre-k3s-cp-01`).
3. **Inisialisasi Pemulihan (Cluster Reset):**
   Pastikan direktori `/var/lib/rancher/k3s/server/db/etcd` bersih total, lalu jalankan:
   ```bash
   k3s server --cluster-reset --cluster-reset-restore-path=/tmp/k3s-etcd-prod-20261017T000000.db
   systemctl start k3s
   ```
4. **Verifikasi Keutuhan State:**
   Gunakan `kubectl get nodes` dan `kubectl get pods -A` pada master 1 untuk memastikan seluruh namespace, secret, dan konfigurasi aplikasi per pukul 00.00 berhasil dipulihkan.
5. **Rekonstruksi Multi-Node HA via Ansible:**
   Jalankan playbook `deploy-k3s-ha.yml --limit 'sre-k3s-cp-02,sre-k3s-cp-03'` untuk menggabungkan master 2 dan master 3 ke dalam kuorum etcd baru, lalu kembalikan seluruh worker node ke klaster.

---

## 9. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [02 — Role Instalasi dan Upgrade k3s Multi-Node HA](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/02-k3s-install-upgrade-role.md) — Memahami arsitektur kuorum etcd dan konfigurasi multi-master.
  - [03 — Hardening, Patching, dan Rolling Maintenance](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/03-hardening-patching-rolling.md) — Menjalankan siklus pemeliharaan node secara berkala.
- **Praktik Laboratorium:**
  - [LAB-01 — Handoff k3s Multi-node](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/lab/LAB-01-handoff-k3s-multinode.md) — Menguji deployment k3s multi-server secara penuh.
  - [LAB-02 — Rolling Patching dan Readiness](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/modul-4.3-ansible-k3s-onprem/lab/LAB-02-rolling-patching-readiness.md) — Melatih penambalan rolling dan pengujian readiness di terminal.
- **Fase Berikutnya:**
  - [Fase 5 — Helm: Packaging Aplikasi Kubernetes](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-5-helm/) — Mengemas dan mendistribusikan aplikasi ke atas klaster k3s yang telah tangguh.
