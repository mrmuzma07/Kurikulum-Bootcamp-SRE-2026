# 9.1.2 — Contract OpenTofu → Ansible → k3s

> **Ringkasan:** Merancang dan menegakkan kontrak data serah terima (**Data Handoff Contract**) dari OpenTofu ke Ansible hingga k3s, memvalidasi kesiapan klaster (*Readiness Handoff Criteria*) secara objektif, serta mengeksekusi protokol pembongkaran dan pembangunan kembali platform secara total (**Total Destroy & Cold-Start Rebuild**) tanpa ketergantungan pada modifikasi manual.

---

## 1. Prinsip Handoff Berbasis Kontrak (Contract-Driven Handoff)

### 1.1 Analogi Dunia Nyata: "Tongkat Estafet Lari Olimpiade & Surat Serah Terima Kunci Gedung"
Bayangkan sebuah proyek pembangunan gedung bertingkat:
- **Kontraktor Fondasi (OpenTofu):** Menggali tanah, mengecor pilar beton, dan memasang pipa air utama di bawah tanah. Setelah selesai, kontraktor tidak boleh hanya berteriak: *"Fondasi sudah beres, silakan lanjut!"*. Mereka wajib menyerahkan **Berita Acara Serah Terima Resmi** yang berisi data presisi: titik koordinat pilar, daya tahan beban beton, dan letak sambungan pipa air.
- **Kontraktor Interior & Utilitas (Ansible):** Menerima dokumen tersebut, memeriksa bahwa semen sudah kering sempurna (*Preflight Readiness*), lalu memasang kabel listrik, panel sirkuit, dan pintu ruangan.
- **Pengelola Gedung (Kubernetes k3s):** Baru boleh mengizinkan penghuni (aplikasi bisnis) masuk setelah seluruh lampu menyala, keran air mengalir, dan alarm kebakaran aktif.

Dalam rekayasa platform SRE, **Handoff** antar alat infrastruktur **BUKANLAH** menyalin-tempel output terminal atau membagikan kredensial melalui obrolan Slack. Handoff adalah **Kontrak Data Terstruktur (*Structured Data Contract*)** yang dapat dibaca mesin (*machine-readable*), dapat divalidasi skemanya, dan sepenuhnya **bersih dari data rahasia (*non-secret*)**.

---

### 1.2 Alur Data Handoff Platform Rebuild

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ALUR DATA KONTRAK HANDOFF TIGA LAPISAN                      │
│                                                                             │
│  [ LAPISAN 1: OPENTOFU ]                                                    │
│   Provisioning VM & Jaringan di OrbStack ──> `tofu apply -out=tfplan`       │
│   Menghasilkan JSON non-secret:                                             │
│   `tofu output -json > /tmp/platform-handoff.json`                          │
│                                  │                                          │
│                                  ▼ (Inventory Consumption)                  │
│  [ LAPISAN 2: ANSIBLE ]                                                     │
│   Dynamic Inventory membaca IP Node & Alamat Bastion                        │
│   ├─ Playbook OS Hardening & Sysctl Tuning (Idempoten)                      │
│   ├─ Instalasi k3s Server (Control-Plane dengan etcd Quorum)                │
│   └─ Instalasi k3s Agents (Worker Nodes) & MetalLB CRDs                     │
│                                  │                                          │
│                                  ▼ (Cluster Readiness Validation)           │
│  [ LAPISAN 3: K3S & METALLB ]                                               │
│   Verifikasi Kesiapan Klaster:                                              │
│   ├─ etcd Quorum Sehat (Minimal 1 Server di Lab / 3 di Prod)                │
│   ├─ Seluruh Node Berstatus `Ready` dengan Arsitektur ARM64                 │
│   ├─ StorageClass Local-Path Aktif (`WaitForFirstConsumer`)                 │
│   └─ MetalLB AddressPool Terverifikasi Tidak Tumpang Tindih                 │
│                                  │                                          │
│                                  ▼ (Persetujuan Gerbang Handoff)            │
│  [ SERAH TERIMA KE MODUL 9.2: GITOPS & OBSERVABILITY ]                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Skema Dokumen Handoff Non-Secret (`platform-handoff.yaml`)

Berikut adalah contoh kontrak data serah terima resmi yang dihasilkan secara otomatis dan diverifikasi sebelum lapisan aplikasi (GitOps) diizinkan berjalan:

```yaml
schema_version: "sre.bootcamp.capstone/v1.0"
capstone_revision: "git-commit-a8f1b2c3d4e5"
generated_at_utc: "2026-10-18T09:00:00Z"

# 1. Target Lingkungan & Batas Scope Eksekusi
target:
  environment: "disposable-production-simulation"
  workspace: "capstone-orbstack-arm64"
  platform_architecture: "linux/arm64"
  allowlist_vm_ids:
    - "infra-vm"
    - "k3s-master-01"
    - "k3s-worker-01"
    - "k3s-worker-02"

# 2. Pemetaan Node Fisik/VM (Non-Secret Network Identifiers)
nodes:
  - name: "k3s-master-01"
    role: "control-plane"
    ip_address: "192.168.20.11"
    hardware_spec: "2 vCPU, 4 GiB RAM"
  - name: "k3s-worker-01"
    role: "worker"
    ip_address: "192.168.20.12"
    hardware_spec: "4 vCPU, 8 GiB RAM"
  - name: "k3s-worker-02"
    role: "worker"
    ip_address: "192.168.20.13"
    hardware_spec: "4 vCPU, 8 GiB RAM"

# 3. Alokasi Jaringan Terisolasi (No Overlapping CIDR!)
network:
  management_subnet: "192.168.10.0/24"
  node_cidr: "192.168.20.0/24"
  pod_cidr: "10.42.0.0/16"
  service_cidr: "10.43.0.0/16"
  metallb_vip_pool: "192.168.30.200-192.168.30.220" # Dedicated 21 VIP Addresses

# 4. Akses Administratif & Titik Pemulihan
access_governance:
  bastion_host: "192.168.10.5"
  ssh_user: "sre-admin"
  auth_mechanism: "ed25519-cert" # Tanpa password/private key di file ini!
  break_glass_runbook: "RB-ACCESS-001"

# 5. Kriteria Kesiapan Runtime (Readiness Acceptance Criteria)
readiness_criteria:
  etcd_quorum_nodes_required: 1 # Lab single master (3 untuk HA prod)
  expected_ready_nodes: 3
  storage_class_default: "local-path"
  volume_binding_mode: "WaitForFirstConsumer"
  capacity_headroom_percent: 33.3 # N+1 Headroom terpenuhi
  strict_arp_enabled: true
```

> [!CAUTION]
> **Aturan Higienitas Rahasia (*Secrets Hygiene*) pada Dokumen Handoff:**
> Dokumen handoff di atas **DILARANG KERAS** memuat: token k3s (`K3S_TOKEN`), isi file `kubeconfig`, SSH Private Key, password MinIO, atau token GitLab Runner! Seluruh rahasia tersebut wajib disuntikkan secara dinamis saat runtime menggunakan `ansible-vault` atau variabel environment terproteksi.

---

## 3. Daftar Periksa Pra-Penerbangan (Preflight Checklist)

Sebelum menjalankan perintah `tofu apply` atau Ansible playbook bootstrap, SRE wajib melakukan serangkaian pemeriksaan **Read-Only** untuk memastikan kondisi awal memenuhi syarat:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DAFTAR PERIKSA READ-ONLY PRA-PENERBANGAN                    │
├──────────────────────────┬──────────────────────────────────────────────────┤
│ Komponen yang Diperiksa  │ Perintah Verifikasi & Syarat Kelulusan           │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 1. Arsitektur CPU Host   │ `uname -m` ➔ Wajib mengembalikan `arm64` (Apple  │
│                          │ Silicon) dan OrbStack dalam status running.      │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 2. Budget RAM OrbStack   │ `orb info` ➔ Pastikan RAM fisik host yang di-    │
│                          │ alokasikan ke OrbStack minimal 12 GiB – 16 GiB.  │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 3. Remote State Locking  │ `tofu state list` ➔ Pastikan bucket MinIO dapat  │
│                          │ diakses dan tidak ada state lock yang menggantung│
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 4. Subnet Non-Overlap    │ Periksa agar `metallb_vip_pool` tidak berada di  │
│                          │ subnet yang sama dengan DHCP Wi-Fi rumah/kantor. │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ 5. Tool Versions         │ `tofu version`, `ansible --version`, dan         │
│                          │ `kubectl version --client` sesuai standar repo.  │
└──────────────────────────┴──────────────────────────────────────────────────┘
```

---

## 4. Protokol Cold-Start Destroy & Rebuild (Disaster Recovery Simulation)

Dalam ujian Capstone SRE, pembuktian kemampuan sesungguhnya terjadi saat platform dihancurkan secara total (*Total Destruction*) lalu dibangun kembali dari nol (*Cold-Start Rebuild*) hanya dengan menggunakan repositori Git!

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 PROTOKOL 7 TAHAP COLD-START TOTAL REBUILD                   │
│                                                                             │
│  [1. Backup State]     ──> Buat etcd snapshot & Velero backup terakhir.     │
│  [2. Scoped Destroy]   ──> Eksekusi pembongkaran VM: `tofu destroy`         │
│  [3. Verify Cleanup]   ──> Pastikan seluruh VM di OrbStack musnah total.    │
│  [4. IaC Provision]    ──> Bangun VM baru: `tofu apply -out=tfplan`         │
│  [5. OS & k3s Bootstrap]─> Jalankan Ansible Playbook: `ansible-playbook`    │
│  [6. Handoff Check]    ──> Validasi etcd Quorum, Node Ready, StorageClass.  │
│  [7. GitOps Rehydrate] ──> Hubungkan ArgoCD ➔ Workload & Telemetri pulih!   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Aturan Keselamatan Eksekusi Destroy
1. **Verifikasi Target Scope:** Pastikan workspace yang sedang aktif adalah workspace simulasi disposable (`tofu workspace show`), bukan cluster produksi fisik lain!
2. **Review Destroy Plan:** Jalankan `tofu plan -destroy` dan periksa daftar ID resource yang akan dihapus. **Dilarang keras menggunakan flag `-auto-approve` secara ceroboh!**
3. **Penyimpanan Snapshot Terpisah:** Pastikan file snapshot etcd dan arsip backup Velero disimpan di bucket MinIO yang berada di luar VM yang sedang dihancurkan.

---

## 5. Pola Kegagalan (Failure Modes) & Diagnosa Cepat

| Gejala Masalah | Penyebab Utama | Langkah Diagnosa & Solusi SRE |
|---|---|---|
| **Ansible Gagal Menghubungi VM Baru (`SSH Connection Refused`)** | VM baru yang dibuat oleh OpenTofu belum selesai melakukan proses inisialisasi cloud-init atau kunci host SSH berubah (*Host key verification failed*). | 1. Tambahkan task `wait_for` port 22 di awal playbook.<br>2. Gunakan opsi `StrictHostKeyChecking=accept-new` pada file `ansible.cfg` khusus lingkungan disposable lab. |
| **Node Worker Menolak Bergabung ke Master k3s (`Unauthorized 401`)** | K3s node token yang di-generate saat master bootstrap tidak diteruskan secara benar ke variabel role worker node. | Pastikan playbook Ansible master menyimpan token ke variabel terenkripsi (`register: k3s_token`) dan menyuntikkannya ke file `/etc/rancher/k3s/config.yaml` di worker node. |
| **Pod Mengalami Status `Pending` Pasca-Rebuild** | StorageClass `local-path` kehilangan path direktori fisik di worker node baru. | Periksa apakah direktori `/opt/local-path-provisioner` sudah dibuat oleh Ansible Playbook dengan izin kepemilikan yang sesuai sebelum pod dideploy. |

---

## 6. Cheatsheet & Ringkasan Perintah Rebuild

```bash
# ==============================================================================
# 1. PREFLIGHT & EKSEKUSI OPENTOFU REBUILD
# ==============================================================================

# Memeriksa rencana perubahan infrastruktur secara deterministik
tofu plan -out=rebuild.tfplan

# Menerapkan rencana yang telah disetujui
tofu apply rebuild.tfplan

# Mengekstrak metadata hasil apply ke format JSON non-secret
tofu output -json > /tmp/platform-handoff.json

# ==============================================================================
# 2. EKSEKUSI BOOTSTRAP ANSIBLE DENGAN LIMIT & CHECK MODE
# ==============================================================================

# Menguji konektivitas SSH ke seluruh node hasil provisioning OpenTofu
ansible -i inventory/hosts.ini all -m ping

# Menjalankan playbook instalasi k3s cluster dengan tampilan diff
ansible-playbook -i inventory/hosts.ini playbooks/site.yml --diff

# ==============================================================================
# 3. VERIFIKASI GERBANG READINESS K3S SEBELUM HANDOFF GITOPS
# ==============================================================================

# Memeriksa status kesehatan seluruh node k3s
kubectl get nodes -o wide

# Memeriksa ketersediaan StorageClass default
kubectl get storageclass

# Memeriksa apakah pod speaker MetalLB berjalan di seluruh node
kubectl get pods -n metallb-system -l component=speaker
```

---

## 7. Evaluasi Pemahaman Mandiri

### 7.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa dalam alur rekayasa platform modern, data output dari **OpenTofu** yang akan diserahterimakan ke **Ansible** wajib didefinisikan sebagai **Kontrak Data Terstruktur Non-Secret (*Structured Non-Secret Contract*)**?
- A. Agar ukuran file output selalu di bawah 1 Kilobyte.
- B. Untuk memisahkan tanggung jawab secara tegas (*Separation of Concerns*), mencegah kebocoran kredensial rahasia ke file log, serta memungkinkan proses otomatisasi membaca data identitas server (IP, hostname, CIDR) secara deterministik tanpa intervensi manual.
- C. Karena Ansible tidak mendukung format file selain file teks biasa.
- D. Agar proses instalasi sistem operasi bisa dilewati secara langsung.

#### Soal 2
Saat melakukan verifikasi kesiapan klaster (*Readiness Handoff*) setelah k3s selesai diinstal oleh Ansible, mengapa status node yang menunjukkan **`Ready` saja BELUM CUKUP** untuk menyatakan klaster siap menerima deployment aplikasi bisnis?
- A. Karena status Ready hanya berlaku untuk node berbasis Windows.
- B. Karena status `Ready` hanya membuktikan bahwa Kubelet lokal terhubung ke API server; SRE masih wajib memverifikasi kesiapan komponen lain seperti: keabsahan kuorum etcd, ketersediaan StorageClass (`WaitForFirstConsumer`), alokasi pool IP MetalLB, dan CNI CoreDNS.
- C. Karena k3s otomatis menghapus node setelah 10 menit.
- D. Karena kubectl get nodes tidak membaca status dari memori server.

#### Soal 3
Dalam simulasi pemulihan bencana (*Cold-Start Total Rebuild Drill*), tindakan preventif apakah yang **WAJIB** dipastikan sebelum menjalankan perintah pembongkaran total `tofu destroy`?
- A. Memastikan seluruh aplikasi sudah dihapus satu per satu menggunakan mouse.
- B. Memastikan bahwa workspace yang aktif adalah lingkungan lab disposable yang diizinkan (*allowlisted*), dan salinan pencadangan (*snapshot etcd & backup Velero*) telah tersimpan aman di penyimpanan eksternal yang tidak ikut terhapus.
- C. Mematikan sambungan listrik rumah secara mendadak.
- D. Mengubah alamat email penanggung jawab sistem.

---

### 7.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Dalam ujian sertifikasi Capstone, seorang peserta diminta melakukan simulasi rekonstruksi total (*Total Platform Rebuild*) setelah skenario server hancur.
Peserta tersebut menjalankan `tofu apply` dan berhasil membuat 3 VM baru di OrbStack dalam waktu 2 menit.
Selanjutnya, peserta menjalankan Ansible Playbook untuk menginstal k3s cluster. Playbook selesai dengan status `changed=14 failed=0`.

Namun, ketika peserta langsung menyerahkan sistem ke dewan penguji dengan berkata: *"Klaster k3s sudah selesai dibangun, silakan pasang ArgoCD!"*, dewan penguji menemukan bahwa:
1. Pod `coredns` berstatus `CrashLoopBackOff` karena bentrok dengan `systemd-resolved` di host Linux.
2. MetalLB pool yang dikonfigurasi menggunakan subnet `192.168.1.0/24` yang bertabrakan langsung dengan router Wi-Fi ruang ujian.
3. StorageClass `local-path` belum terpasang.

Dewan penguji menyatakan penyerahan (*handoff*) **GAGAL (*REJECTED*)**.

**Pertanyaan & Tugas Anda sebagai Lead Evaluator:**
1. Mengapa output Ansible `failed=0` dan status VM `Running` tidak otomatis membuktikan bahwa gerbang serah terima (*Readiness Handoff*) telah tercapai?
2. Sebutkan **3 parameter kritis dalam skema `platform-handoff.yaml`** yang diabaikan oleh peserta tersebut!
3. Tuliskan 3 perintah pengujian otomatis (*automated readiness validation script*) yang seharusnya dijalankan peserta tersebut untuk memvalidasi klaster sebelum mengajukan handoff!

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Kontrak data non-secret menjaga arsitektur tetap bersih dari ketergantungan manual (*loose coupling*) sekaligus memastikan tidak ada kredensial sensitif yang tersimpan di file sementara.
2. **B.** Status node `Ready` hanyalah sinyal dasar Kubelet. Kesiapan operasional platform membutuhkan fungsi jaringan (MetalLB/DNS) dan penyimpanan (StorageClass) yang bekerja serasi.
3. **B.** Menghancurkan infrastruktur tanpa memverifikasi target workspace dan ketersediaan backup off-site adalah pelanggaran prosedur operasional fatal yang dapat memicu kehilangan data permanen.

#### Solusi Soal Kasus Nyata:
1. **Mengapa Ansible `failed=0` Belum Membuktikan Kesiapan:**
   - Ansible hanya membuktikan bahwa paket biner k3s berhasil diinstal dan service systemd aktif. Ansible tidak secara otomatis memvalidasi apakah CoreDNS di dalam Kubernetes dapat melakukan resolusi domain internal atau apakah ada tabrakan IP di jaringan fisik Layer 2.
2. **Tiga Parameter Kritis yang Diabaikan:**
   - **`network.metallb_vip_pool`:** Peserta tidak memeriksa *non-overlapping CIDR*, menyebabkan tabrakan ARP dengan router lokal.
   - **`readiness_criteria.storage_class_default`:** Peserta lupa memvalidasi ketersediaan penyedia volume persisten (`local-path-provisioner`).
   - **`readiness_criteria.cni_dns_health`:** Peserta mengabaikan resolusi port 53 DNS lokal yang terbentur proteksi `systemd-resolved`.
3. **Tiga Perintah Skrip Validasi Otomatis Sebelum Handoff:**
   ```bash
   # 1. Uji resolusi DNS internal Kubernetes di dalam klaster
   kubectl run test-dns --image=busybox:1.36 --restart=Never --rm -it -- nslookup kubernetes.default.svc.cluster.local

   # 2. Uji alokasi IP MetalLB dan ARP broadcast
   arping -c 2 -I eth0 $(kubectl get ipaddresspool -n metallb-system -o jsonpath='{.items[0].spec.addresses[0]}' | cut -d'-' -f1)

   # 3. Uji fungsionalitas provisi StorageClass default
   kubectl get storageclass local-path -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}'
   ```

---

## 8. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai kontrak serah terima infrastruktur dan protokol cold-start rebuild:
- Lanjutkan ke [Modul 9.2 — Delivery, Observability, dan Recovery Evidence](../modul-9.2-delivery-observability/README.md) untuk mempelajari deployment pipeline GitOps, integrasi observabilitas telemetri penuh, dan penyusunan bukti kepatuhan (*evidence packet*).
- Praktikkan simulasi serah terima di [LAB-02 — k3s Bootstrap dan Handoff Lab](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-9-capstone/modul-9.1-platform-rebuild/lab/LAB-02-k3s-bootstrap-handoff.md).
