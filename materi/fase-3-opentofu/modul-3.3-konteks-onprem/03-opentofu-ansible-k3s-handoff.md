# 03 — OpenTofu, Ansible, dan k3s Handoff: Mengatur Estafet Provisioning, Konfigurasi OS, dan Bootstrap Cluster Tanpa Cacat

> **Ringkasan:** Membangun infrastruktur *on-premises* kelas produksi memerlukan pembagian peran yang tegas antara tiga pilar otomatisasi: **OpenTofu** (penyedia wujud komputasi & jaringan), **Ansible** (arsitek konfigurasi sistem operasi & keamanan), dan **k3s** (orkestrator kontainer & beban kerja). Kesalahan paling fatal seorang pemula adalah mencampuradukkan ketiga peran ini atau melakukan transisi tanpa validasi kesiapan (*readiness gate*). Dokumen ini membedah arsitektur "Estafet Tiga Pilar", merancang kontrak metadata non-rahasia (*non-secret handoff contract*), menyusun 8 gerbang kesiapan (*host readiness gates*), serta menangani kegagalan transisi secara idempoten dan aman.

---

## 1. Analogi Pemula: Lari Estafet 4x100m Olimpiade

Bayangkan sebuah tim lari estafet kelas dunia di ajang Olimpiade. Kemenangan bukan hanya ditentukan oleh seberapa cepat masing-masing pelari berlari, melainkan **seberapa mulus dan disiplin tongkat estafet berpindah di zona pertukaran (*passing zone*)**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI ESTAFET TIGA PILAR OTOMASI INFRASTRUKTUR            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PELARI 1: OPENTOFU ]                                                     │
│  - Berlari di lintasan pertama: Provisioning Lahan & Konstruksi Fisik.      │
│  - Tugas: Meminta hypervisor menyalakan 3 VM, memasang vNIC ke VLAN 20,    │
│    dan mengalokasikan IP statis.                                            │
│  - Di Garis Transfer (Handoff Zone):                                        │
│    Pelari 1 TIDAK BOLEH memasang aplikasi atau mengedit konfigurasi daemon. │
│    Pelari 1 HANYA menyerahkan "Tongkat Estafet" berisi daftar alamat IP     │
│    dan nama host (**Metadata Non-Secret**).                                 │
│                                     │                                       │
│                                     ▼ (Handoff Zone: Host Readiness Gate)   │
│                                                                             │
│  [ PELARI 2: ANSIBLE ]                                                      │
│  - Menerima tongkat: Konfigurasi Sistem Operasi & Hardening.                │
│  - Tugas: Memeriksa apakah host bernapas (ping), menyinkronkan jam (Chrony),│
│    mematikan swap memory, memasang firewall, dan menutup celah keamanan.    │
│  - Di Garis Transfer Berikutnya:                                            │
│    Ansible memastikan sistem operasi 100% sehat sebelum memanggil binary    │
│    k3s.                                                                     │
│                                     │                                       │
│                                     ▼ (Cluster Readiness Gate)              │
│                                                                             │
│  [ PELARI 3: K3S CLUSTER ]                                                  │
│  - Membawa tongkat ke garis akhir: Orkestrasi Beban Kerja Produksi.         │
│  - Tugas: Memulai etcd quorum, menjalankan control plane, mengizinkan       │
│    worker node bergabung, dan menyalakan Pod aplikasi pelanggan.            │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│  PELANGGARAN DISKUALIFIKASI (FATAL ERRORS):                                 │
│  1. Pelari 1 melempar kunci brankas (Secret/Private Key) ke penonton        │
│     (Menaruh private key di output OpenTofu yang terekspos di log CI/CD).   │
│  2. Pelari 2 lari sebelum Pelari 1 sampai di zona transfer                  │
│     (Ansible jalan duluan saat VM masih proses booting / SSH belum up).     │
│  3. Pelari 3 menjatuhkan tongkat karena lintasan licin                      │
│     (k3s install dipaksakan padahal waktu server beda 2 detik / disk penuh).│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Jika salah satu pelari mengambil alih tugas pelari lain (misalnya OpenTofu mencoba menginstal Kubernetes menggunakan `local-exec` atau SSH mentah), rantai kontrol akan putus. Saat terjadi *drift* atau kegagalan jaringan, Anda tidak akan tahu lapisan mana yang rusak, dan proses *rollback* menjadi mimpi buruk!

---

## 2. Tiga Boundary Kepemilikan & Tanggung Jawab

Di level produksi, batasan tanggung jawab (*boundary of responsibility*) harus dipisahkan secara ketat ke dalam tiga pilar:

| Dimensi Arsitektur | Pilar 1: OpenTofu (IaC) | Pilar 2: Ansible (Config Mgmt) | Pilar 3: Kubernetes k3s (Runtime) |
|---|---|---|---|
| **Domain Utama** | *Infrastructure Existence & Topology* | *OS State, Hardening, & Packages* | *Container Orchestration & Workloads* |
| **Objek yang Dikelola** | VM, Container, Disk Virtual, vNIC, VLAN | Kernel modules, Sysctl, Swap, Chrony, Firewall, User | Pods, Services, Deployments, CRDs, Etcd Raft |
| **Sumber Kebenaran (*State*)** | `terraform.tfstate` di MinIO S3 | *Stateless* (Keadaan aktual host OS) | `etcd` / SQLite cluster state |
| **Protokol Eksekusi** | Provider API (Hypervisor HTTPS / Libvirt RPC) | Agentless SSH (`port 22`) via Python | Kubernetes API (`https://<VIP>:6443`) |
| **Kriteria Keberhasilan** | Host berstatus `RUNNING` & memiliki IP | Seluruh task Ansible berstatus `ok` / `changed=0` | Semua Node berstatus `Ready` & Quorum Etcd sehat |
| **Aksi saat Gagal (*Rollback*)** | `tofu destroy` atau replace VM spesifik | Re-run playbook idempoten dengan `--limit` | `kubectl cordon/drain` atau k3s snapshot restore |

### Diagram Alur Alih Tugas (Handoff Flow)

```text
┌─────────────────┐
│   Developer /   │
│    GitLab CI    │
└────────┬────────┘
         │
         │ 1. tofu apply
         ▼
┌─────────────────────────────────┐
│   OpenTofu Provisioning Layer   │
│   - Buat 3 VM di Hypervisor     │
│   - Tanam SSH Public Key (Seed) │
│   - Catat IP di State           │
└────────┬────────────────────────┘
         │
         │ 2. Emit Output: ansible_inventory_metadata (NON-SECRET!)
         ▼
┌─────────────────────────────────────────────────────────────┐
│               HOST READINESS GATE (Pre-Ansible)             │
│   [✓] Host bernyawa? (ICMP / Port 22 open)                  │
│   [✓] Hostname & IP tidak bentrok?                          │
│   [✓] Cloud-Init selesai tuntas? (`cloud-init status --wait`)│
└────────┬────────────────────────────────────────────────────┘
         │
         │ 3. Jalankan Ansible Playbook (inventory generated)
         ▼
┌─────────────────────────────────┐
│     Ansible OS Hardening Layer  │
│   - Set Chrony (NTP Sync)       │
│   - swapoff -a & modprobe br_   │
│   - UFW firewall rules          │
│   - Siapkan direktori /var/lib/ │
└────────┬────────────────────────┘
         │
         │ 4. Verifikasi OS Baseline Lulus
         ▼
┌─────────────────────────────────────────────────────────────┐
│              CLUSTER READINESS GATE (Pre-k3s)               │
│   [✓] Clock skew < 100ms?                                   │
│   [✓] Port 6443 & 2379 tidak terblokir?                     │
│   [✓] Disk I/O /var/lib/rancher/k3s sehat?                  │
└────────┬────────────────────────────────────────────────────┘
         │
         │ 5. Bootstrap k3s Server-1 -> Join Server-2 & Worker-1
         ▼
┌─────────────────────────────────┐
│    k3s Cluster Runtime Layer    │
│   - Inisialisasi Etcd Quorum    │
│   - Pasang CoreDNS & Flannel    │
│   - Siap terima MetalLB & Helm  │
└─────────────────────────────────┘
```

---

## 3. Kontrak Metadata Non-Secret: Apa yang Boleh dan Dilarang Diteruskan

Setelah OpenTofu selesai menjalankan tugasnya, ia harus mengekspor informasi infrastruktur agar dapat dikonsumsi oleh Ansible. Di sinilah aturan **Strict Secrets Hygiene** berlaku mutlak.

### 3.1 Kode HCL Kontrak Output (`outputs.tf`)

Perhatikan bahwa output di bawah ini **HANYA** memuat metadata operasional non-sensitif:

```hcl
# outputs.tf pada root module OpenTofu

output "ansible_inventory_metadata" {
  description = "Kontrak metadata non-secret untuk diserahkan ke tahap Configuration Management (Ansible)."

  value = {
    cluster_name = var.cluster_name
    environment  = var.environment
    generated_at = timestamp()
    
    # Metadata per host yang berhasil diprovisioning
    hosts = {
      for key, node in module.compute_hosts : key => {
        hostname         = node.hostname
        ip_address       = node.ip_address
        role             = node.role            # 'server' (control-plane) atau 'agent' (worker)
        mac_address      = node.mac_address
        failure_domain   = node.availability_zone
        provisioning_ref = node.vm_id
      }
    }
  }
}
```

### 3.2 Matriks Filter Keamanan: Apa yang Boleh vs Dilarang

```text
┌──────────────────────────────────────┬──────────────────────────────────────┐
│  DIIZINKAN DI OUTPUT OPENTOFU (AMAN) │  DILARANG KERAS DI OUTPUT (BAHAYA!)  │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ [✓] Alamat IP Statis (Private IP)    │ [✗] SSH Private Key pembuat VM       │
│ [✓] Fully Qualified Domain Name (FQDN)│ [✗] Password root/admin sistem operasi│
│ [✓] Peran Node (server vs agent)     │ [✗] k3s Cluster Join Token (`K3S_TOKEN`)│
│ [✓] MAC Address kartu jaringan virtual│ [✗] Admin Kubeconfig (`k3s.yaml`)     │
│ [✓] ID Objek Hypervisor / UUID VM    │ [✗] Token Rahasia Proxmox / vSphere  │
│ [✓] Versi OS dan Arsitektur CPU      │ [✗] Kunci Enkripsi Disk / Hash Password│
└──────────────────────────────────────┴──────────────────────────────────────┘
```

> [!CAUTION]
> **Mengapa Rahasia Tidak Boleh Ada di Output OpenTofu?**
> Setiap nilai yang dicetak pada blok `output` akan disimpan dalam format **plaintext** di dalam file state (`terraform.tfstate`). Jika Anda menggunakan MinIO S3 backend, secret tersebut tersimpan di bucket. Lebih buruk lagi, jika pipeline CI/CD Anda mengeksekusi `tofu output`, seluruh token dan private key akan tercetak di layar konsol pipeline, terbaca oleh siapa pun yang memiliki hak akses baca ke repositori Git!

---

## 4. Host Readiness Gate: 8 Gerbang Pemeriksaan Wajib

Ansible **TIDAK BOLEH** langsung dipanggil hanya karena perintah `tofu apply` selesai dengan status sukses. Di lingkungan bare-metal/on-prem, hypervisor mungkin melaporkan VM sudah "dibuat", tetapi di dalam guest OS, inisialisasi jaringan atau Cloud-Init masih berjalan.

Berikut adalah **8 Gerbang Kesiapan (*Host Readiness Gate*)** yang wajib divalidasi oleh skrip orkestrator atau tahap awal pipeline sebelum Ansible menyentuh konfigurasi sistem operasi:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 8 GERBANG KESIAPAN HOST (HOST READINESS GATE)               │
├─────┬──────────────┬──────────────────────────────┬─────────────────────────┤
│ No  │ Gerbang      │ Kondisi yang Wajib Terpenuhi │ Perintah Diagnosa SRE   │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 1   │ Identity     │ Hostname unik, FQDN valid,   │ hostnamectl status      │
│     │              │ role jelas (server/agent).   │                         │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 2   │ Network      │ Tidak ada IP collision, ARP  │ arping -c 2 <IP>        │
│     │              │ stabil, gateway reachable.   │ ip route get 1.1.1.1    │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 3   │ Time Sync    │ NTP/Chrony aktif, clock skew │ chronyc tracking        │
│     │              │ antar node < 100 milidetik.  │ (Max error < 0.1s)      │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 4   │ SSH Access   │ Key-based auth sukses, no    │ ssh -o BatchMode=yes    │
│     │              │ password prompt, sudo tanpa  │ -i id_ed25519 user@host │
│     │              │ hambatan interaktif.         │ 'echo READY'            │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 5   │ OS Baseline  │ Ubuntu 22.04 LTS ARM64/AMD64,│ uname -r && swapoff -a  │
│     │              │ SWAP memory MATI TOTAL!      │ free -m (Swap: 0B)      │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 6   │ Storage      │ Direktori `/var/lib/rancher` │ df -hT /var/lib/rancher │
│     │              │ sisa kapasitas > 20GB, IOPS  │ dd test latency disk    │
│     │              │ cukup untuk etcd write.      │                         │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 7   │ Firewall     │ Port matrix k3s dibuka antar │ nc -zvw3 <IP> 6443      │
│     │              │ node: 6443, 2379-2380, 8472. │ nc -zvw3 -u <IP> 8472   │
├─────┼──────────────┼──────────────────────────────┼─────────────────────────┤
│ 8   │ Cloud-Init   │ Status Cloud-Init `done`,    │ cloud-init status       │
│     │              │ apt/dpkg lock sudah dilepas! │ --wait                  │
└─────┴──────────────┴──────────────────────────────┴─────────────────────────┘
```

> [!IMPORTANT]
> **Mengapa Clock Skew & Swap Begitu Mematikan Bagi k3s?**
> 1. **Etcd Quorum & TLS:** Etcd menggunakan algoritma konsensus Raft yang sangat sensitif terhadap waktu. Jika perbedaan jam (*clock skew*) antar-server k3s melebihi 100ms hingga 1 detik, pemilihan ketua (*leader election*) akan berulang tanpa henti (*flapping*), menyebabkan cluster terpecah (*split-brain*). Selain itu, sertifikat TLS internal k3s akan ditolak jika waktu sistem melompat ke masa lalu.
> 2. **Swap Memory:** Kubelet Kubernetes secara default menolak berjalan jika swap aktif. Alokator memori Kubernetes mengasumsikan Pod memiliki batas resource yang presisi. Jika OS memindahkan memori Pod ke swap disk, jaminan SLA performa dan OOM-killer Kubernetes akan hancur total.

---

## 5. Dynamic Inventory Handoff: Menghubungkan Output ke Ansible

Bagaimana Ansible membaca metadata yang dihasilkan oleh OpenTofu tanpa perlu menulis ulang file inventaris secara manual?

### 5.1 Menghasilkan Inventory dari Output OpenTofu

Gunakan parser sederhana (atau jq / template Terraform) untuk menerjemahkan output OpenTofu ke dalam format YAML inventaris Ansible:

```bash
# Mengekstrak metadata JSON dari OpenTofu
tofu output -json ansible_inventory_metadata > /tmp/tofu_metadata.json
```

Hasil file inventaris yang dikonsumsi oleh Ansible (`inventory/hosts.yaml`):

```yaml
---
all:
  vars:
    ansible_user: "sre-admin"
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
    environment_name: "production"
    k3s_version: "v1.31.1+k3s1"

  children:
    k3s_servers:
      hosts:
        sre-k3s-cp-01:
          ansible_host: 192.168.10.11
          node_role: server
          is_bootstrap_server: true
        sre-k3s-cp-02:
          ansible_host: 192.168.10.12
          node_role: server
          is_bootstrap_server: false
        sre-k3s-cp-03:
          ansible_host: 192.168.10.13
          node_role: server
          is_bootstrap_server: false

    k3s_agents:
      hosts:
        sre-k3s-worker-01:
          ansible_host: 192.168.10.21
          node_role: agent
        sre-k3s-worker-02:
          ansible_host: 192.168.10.22
          node_role: agent

    k3s_cluster:
      children:
        k3s_servers:
        k3s_agents:
```

### 5.2 Manajemen Rahasia di Sisi Ansible

Di manakah private key SSH dan join token k3s disimpan jika tidak ada di OpenTofu?
- **SSH Private Key:** Disuntikkan langsung oleh runner CI/CD melalui SSH Agent (`ssh-add <(echo "$CI_SSH_PRIVATE_KEY")`) atau file terproteksi berizin `chmod 600`.
- **k3s Join Token:** Dihasilkan secara dinamis oleh server pertama (`/var/lib/rancher/k3s/server/node-token`) lalu disalin secara aman ke server/worker berikutnya via memori Ansible (variabel register sementara), atau disimpan di brankas terenkripsi (**Ansible Vault**).

---

## 6. Urutan Eksekusi Produksi Langkah Demi Langkah (13-Step Sequence)

Berikut adalah panduan standar operasional (SOP) pelaksanaan transisi dari nol hingga cluster aktif:

```text
[ TAHAP A: OPENTOFU PROVISIONING ]
  1. Operator / CI mencheckout Git commit SHA yang telah disetujui tim lead.
  2. Inisialisasi OpenTofu dan validasi integritas provider:
     $ tofu init -backend-config="key=prod/k3s.tfstate"
  3. Verifikasi rencana eksekusi:
     $ tofu plan -out=tfplan.binary
  4. Eksekusi pembuatan VM (Hanya provisioner komputasi/jaringan):
     $ tofu apply tfplan.binary
  5. Ekstrak metadata inventaris non-secret:
     $ tofu output -json ansible_inventory_metadata > hosts.json

[ TAHAP B: VALIDASI KESIAPAN (READINESS GATE) ]
  6. Eksekusi skrip validasi 8 Gerbang Kesiapan:
     $ ./scripts/check-host-readiness.sh --inventory hosts.json
     (Pastikan Cloud-Init 'done', Chrony tersinkron, SSH terbuka).
     BILA GAGAL: STOP! Jangan lanjutkan ke Ansible!

[ TAHAP C: ANSIBLE OS CONFIGURATION & HARDENING ]
  7. Jalankan ping konektivitas Ansible:
     $ ansible -i inventory/hosts.yaml k3s_cluster -m ping
  8. Jalankan playbook common hardening OS dalam mode dry-run:
     $ ansible-playbook -i inventory/hosts.yaml playbooks/os-hardening.yaml --check
  9. Eksekusi hardening OS nyata (Matikan swap, pasang Chrony, set firewall):
     $ ansible-playbook -i inventory/hosts.yaml playbooks/os-hardening.yaml

[ TAHAP D: K3S CLUSTER BOOTSTRAP ]
 10. Bootstrap server pertama (Pemimpin inisial Etcd):
     $ ansible-playbook -i inventory/hosts.yaml playbooks/k3s-install.yaml --limit sre-k3s-cp-01
 11. Gabungkan server control-plane tambahan (Membentuk 3-node Quorum):
     $ ansible-playbook -i inventory/hosts.yaml playbooks/k3s-install.yaml --limit sre-k3s-cp-02,sre-k3s-cp-03
 12. Gabungkan worker nodes (Agents):
     $ ansible-playbook -i inventory/hosts.yaml playbooks/k3s-install.yaml --limit k3s_agents

[ TAHAP E: VERIFIKASI AKHIR & EVIDENCE ]
 13. Lakukan smoke-test kesehatan cluster Kubernetes:
     $ kubectl get nodes -o wide
     $ kubectl get pods -A
     Arsipkan log eksekusi dan checksum ke tiket persetujuan produksi.
```

---

## 7. Pola Kegagalan (Failure Modes) di Zona Handoff & Diagnosa

Berikut adalah 4 masalah nyata yang paling sering terjadi saat serah terima dari OpenTofu ke Ansible dan k3s, beserta langkah mitigasi seorang SRE:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                   DIAGNOSTIC WORKFLOW: ZONA TRANSISI GAGAL                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ MASALAH 1: PROVISIONING SUKSES, TAPI ANSIBLE SSH TIMEOUT / REFUSED ]     │
│  Gejala: `tofu apply` sukses membuat 3 VM, tapi Ansible gagal:              │
│          "fatal: [sre-k3s-cp-01]: UNREACHABLE! => Connection refused"       │
│  Akar Masalah:                                                              │
│  - VM baru saja selesai dibuat hypervisor, namun proses booting OS belum    │
│    mencapai daemon SSH (`sshd.service`).                                     │
│  - Atau Cloud-Init sedang menjalankan `apt-get upgrade` di latar belakang   │
│    sehingga port 22 ditahan atau firewall lokal memblokir koneksi.          │
│  Solusi SRE:                                                                │
│  1. Jangan destroy VM! Tunggu 60-90 detik.                                  │
│  2. Pasang loop retry dengan timeout rasional pada gerbang kesiapan:        │
│     until nc -zvw5 192.168.10.11 22; do sleep 5; done                       │
│  3. Cek konsol VNC hypervisor untuk melihat apakah OS mengalami kernel panic│
│     atau tersangkut di DHCP request.                                        │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ MASALAH 2: SERVER-2 GAGAL JOIN KE CLUSTER K3S (ETCD SPLIT-BRAIN) ]       │
│  Gejala: Server-1 aktif, tapi Server-2 melempar error:                      │
│          "Failed to find existing cluster member: member list does not match"│
│  Akar Masalah:                                                              │
│  - Jam sistem (*clock skew*) antara Server-1 dan Server-2 berbeda > 500ms.  │
│  - Port 2379/2380 (komunikasi Raft internal etcd) diblokir oleh UFW/iptables│
│    pada salah satu server.                                                  │
│  Solusi SRE:                                                                │
│  1. Uji sinkronisasi waktu: `ansible k3s_servers -m shell -a "date +%s%N"`  │
│  2. Uji keterjangkauan port etcd dari Server-2 ke Server-1:                 │
│     nc -zvw3 192.168.10.11 2379                                             │
│  3. Pastikan `server` flag pada Server-2 mengarah ke IP Server-1 atau VIP   │
│     Load Balancer, bukan ke localhost!                                      │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ MASALAH 3: OPENTOFU INGIN MENGHANCURKAN (REPLACE) VM K3S AKTIF ]         │
│  Gejala: Saat operator menjalankan `tofu plan`, muncul tanda:               │
│          "-/+ resource \"proxmox_vm_qemu\" \"cp1\" (must be replaced)"      │
│  Akar Masalah:                                                              │
│  - Seseorang mengubah konfigurasi hardware yang bersifat *immutable*        │
│    (misal: ukuran disk awal, nama VM, atau MAC address).                    │
│  - Jika di-apply, VM akan DIHANCURKAN, menyebabkan etcd kehilangan quorum!  │
│  Solusi SRE:                                                                │
│  1. TOLAK PLAN TERSEBUT (REJECT)!                                           │
│  2. Tambahkan `lifecycle { prevent_destroy = true }` pada kode module.      │
│  3. Jika pergantian VM memang disengaja: lakukan `kubectl cordon` dan       │
│     `kubectl drain` terlebih dahulu, pastikan backup snapshot etcd aman,    │
│     baru lakukan penghancuran terkontrol.                                   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ MASALAH 4: DPACKAGE/APT LOCK TERKUNCI SAAT ANSIBLE BERJALAN ]            │
│  Gejala: Ansible gagal pada task `apt: name=curl state=present`:            │
│          "E: Could not get lock /var/lib/dpkg/lock-frontend"                │
│  Akar Masalah:                                                              │
│  - Layanan `unattended-upgrades` otomatis milik Ubuntu sedang mendownload   │
│    security patch di latar belakang saat Ansible mulai menginstal paket.    │
│  Solusi SRE:                                                                │
│  1. Di dalam Cloud-Init user-data, matikan auto-upgrade bawaan OS:          │
│     package_update: false, package_upgrade: false                           │
│  2. Biarkan Ansible yang mengelola siklus update package secara terencana.  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Ringkasan Perintah Penting (SRE Cheatsheet)

Simpan referensi cepat perintah diagnosa handoff ini di terminal Anda:

```bash
# 1. Ekspor metadata OpenTofu tanpa menampilkan secret
tofu output -json ansible_inventory_metadata | jq .

# 2. Periksa kesiapan SSH dan waktu sistem seluruh node via Ansible
ansible -i inventory/hosts.yaml all -m ping
ansible -i inventory/hosts.yaml all -m shell -a "chronyc tracking"

# 3. Verifikasi apakah swap sudah mati di seluruh node
ansible -i inventory/hosts.yaml all -m shell -a "free -m | grep -i swap"

# 4. Tes konektivitas port control plane Kubernetes (Port 6443)
nc -zvw5 192.168.10.11 6443

# 5. Cek status etcd cluster k3s pada server yang aktif
k3s etcd-ctl endpoint health --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/client.key

# 6. Inspeksi status node Kubernetes setelah handoff tuntas
kubectl get nodes -o wide --show-labels
```

---

## 9. Pertanyaan Reflektif & Uji Pemahaman Mandiri

Uji pemahaman arsitektur Anda sebelum melangkah ke modul kesiapan produksi:

### Soal 1 (Pilihan Ganda)
Mengapa sangat dilarang memasukkan Private Key SSH atau `K3S_TOKEN` ke dalam blok `output` modul OpenTofu?
- A. Karena OpenTofu tidak mendukung tipe data string rahasia.
- B. Karena output OpenTofu akan otomatis dienkripsi dengan password root host.
- C. Karena seluruh nilai `output` disimpan secara *plaintext* di dalam file `terraform.tfstate`, berisiko bocor melalui storage backend MinIO atau log CI/CD.
- D. Karena Ansible menolak membaca file JSON yang mengandung karakter alfanumerik panjang.

*Kunci Jawaban:* **C**. File state OpenTofu menyimpan seluruh output secara plaintext. Menyimpan secret di output membatalkan prinsip pertahanan berlapis (*defense-in-depth*).

### Soal 2 (Pilihan Ganda)
Seorang SRE menemukan bahwa saat menjalankan playbook instalasi server k3s kedua, node tersebut gagal bergabung dengan pesan error konsensus. Setelah diperiksa, waktu sistem (*clock*) pada Server 2 terlambat 4 detik dibanding Server 1. Mengapa hal ini merusak pembentukan cluster?
- A. Karena Kubelet menolak sertifikat HTTPS yang diterbitkan lebih awal dari waktu sistem saat ini.
- B. Karena algoritma Raft pada etcd membutuhkan toleransi sinkronisasi waktu yang ketat (< 100ms) untuk menentukan kepemimpinan dan masa berlaku heartbeat.
- C. Karena k3s otomatis mematikan layanan jika NTP server tidak terhubung.
- D. Jawaban A dan B benar.

*Kunci Jawaban:* **D**. Sinkronisasi waktu sangat vital bagi validitas sertifikat x509/TLS dan algoritma konsensus Raft etcd.

### Soal 3 (Pilihan Ganda)
Apa yang harus dilakukan jika `tofu plan` pada lingkungan produksi menunjukkan status `must be replaced` (tanda `-/+`) pada node master Kubernetes k3s yang sedang aktif melayani pengguna?
- A. Langsung jalankan `tofu apply` secepat mungkin agar downtime minimal.
- B. Hentikan eksekusi segera (*STOP*), tolak rencana perubahan, dan selidiki atribut immutable apa yang memicu re-creation pada VM tersebut.
- C. Hapus file state OpenTofu dan biarkan Ansible memperbaikinya.
- D. Jalankan `tofu destroy` manual dari terminal lokal.

*Kunci Jawaban:* **B**. Penghancuran VM master secara tiba-tiba tanpa prosedur *cordon*, *drain*, dan manajemen quorum etcd akan merusak stabilitas cluster produksi.

### Soal 4 (Studi Kasus Esai Singkat)
*Skenario:* Anda adalah SRE yang memimpin peluncuran cluster k3s on-premise baru. Pipeline CI/CD Anda baru saja menyelesaikan `tofu apply` untuk 3 node VM. Sebelum menjalankan Ansible, Anda mendapati bahwa Cloud-Init pada VM masih mengunduh kernel update secara otomatis melalui internet, menyebabkan `dpkg` terkunci. Rancang langkah pencegahan arsitektur agar insiden ini tidak terulang di masa depan!

*Panduan Solusi:*
1. Pada konfigurasi template Cloud-Init (`user-data.yaml`), nonaktifkan pembaharuan otomatis:
   ```yaml
   package_update: false
   package_upgrade: false
   ```
2. Tambahkan pemeriksaan kesiapan gerbang (*readiness gate*) di pipeline sebelum memanggil Ansible:
   ```bash
   ssh user@host "cloud-init status --wait"
   ```
   Perintah ini akan menahan eksekusi pipeline sampai proses inisialisasi awal sistem operasi selesai 100%, sehingga tidak ada bentrokan file lock pada `/var/lib/dpkg/lock`.

---

## 10. Kaitan dengan Modul Lain

- **Fondasi Sebelumnya:**
  - [01 — Provider On-Premises dan Boundary Ownership](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/01-provider-onprem-dan-boundary.md) — Memahami batasan tanggung jawab infrastruktur vs OS.
  - [02 — Simulasi Lokal Docker & Mock Provider](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/02-simulasi-lokal-docker-libvirt-mock.md) — Menguji modul pada workstation ARM64.
- **Tahap Berikutnya:**
  - [04 — Production Readiness dan Evidence](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/04-production-readiness-dan-evidence.md) — Membangun rantai bukti audit dan pemulihan drift tanpa downtime.
  - [Fase 4 — Ansible](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-4-ansible/) — Mengeksekusi playbook OS hardening dan instalasi k3s secara nyata.
