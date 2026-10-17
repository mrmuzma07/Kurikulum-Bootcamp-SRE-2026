# 01 — Provider On-Premises dan Boundary Ownership: Mengorkestrasi Virtualisasi Tanpa Mengaburkan Tanggung Jawab

> **Ringkasan:** Di lingkungan cloud publik (AWS/GCP), abstraksi perangkat keras disembunyikan di balik antarmuka serba otomatis. Namun di lingkungan *on-premises* (bare-metal atau private data center), seorang Site Reliability Engineer (SRE) memegang kendali penuh atas seluruh lapisan tumpukan teknologi: mulai dari hypervisor, segmentasi VLAN, alokasi penyimpanan SAN/NFS, inisialisasi *first-boot* Cloud-Init, hingga konfigurasi sistem operasi. Dokumen ini membedah arsitektur provider on-premise (Proxmox, vSphere, libvirt, Cloud-Init), menetapkan matriks batasan tanggung jawab (*Boundary Ownership*), serta memitigasi risiko kegagalan provisioning di infrastruktur fisik.

---

## 1. Analogi Pemula: Kontraktor Pembetonan vs Desainer Interior vs Pemilik Toko

Salah satu kebingungan terbesar bagi engineer pemula saat beralih dari cloud publik ke bare-metal adalah mencampuradukkan peran alat:
> *"Kenapa saya tidak menggunakan OpenTofu saja untuk menginstal package NGINX, mengedit file konfigurasi `/etc/nginx/nginx.conf`, dan menyetel firewall sekalian? Kenapa harus repot-repot memanggil Ansible?"*

Mari kita gunakan analogi pembangunan gedung pertokoan:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI PEMISAHAN TANGGUNG JAWAB KONSTRUKSI                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TAHAP 1: KONTRAKTOR PEMBETONAN BANGUNAN (OPENTOFU / PROVIDER) ]          │
│  Tugas Mandor Pembetonan:                                                   │
│  - Meratakan lahan tanah (Alokasi Resource Hypervisor).                     │
│  - Mengecor pilar beton, dinding luar, dan lantai (Membuat Virtual Machine).│
│  - Menyambungkan pipa air dan kabel listrik utama ke gardu kota             │
│    (Memasang Kartu Jaringan Virtual NIC ke VLAN Bridge fisik).              │
│                                                                             │
│  [ TAHAP 2: JURU KUNCI GERBANG UTAMA (CLOUD-INIT / GUEST SEED) ]            │
│  Tugas Petugas First-Boot:                                                  │
│  - Memberikan nomor rumah sementara di plang depan (Set Hostname).          │
│  - Menyerahkan kunci gembok pintu kepada pemilik sah (Inject SSH Key).      │
│  - Memastikan listrik mengalir saat sakelar pertama kali ditekan            │
│    (Inisialisasi DHCP / Static IP dasar).                                   │
│                                                                             │
│  [ TAHAP 3: DESAINER INTERIOR & TUKANG KAYU (ANSIBLE CONFIGURATION) ]       │
│  Tugas Tukang Interior:                                                     │
│  - Memasang wallpaper, karpet, dan rak display barang (Install Package OS). │
│  - Mengatur tata letak meja kasir dan kunci brankas (Hardening & Config).   │
│  - Memasang alarm anti-maling toko (Firewall UFW & Sysctl Security).        │
│                                                                             │
│  [ TAHAP 4: OPERASIONAL TOKO HARIAN (KUBERNETES K3S & DOCKER) ]             │
│  Toko resmi dibuka untuk melayani transaksi ribuan pembeli harian!          │
│                                                                             │
│  BAHAYA FATAL JIKA TANGGUNG JAWAB DICAMPURADUKKAN:                          │
│  Jika Anda menyuruh Mandor Pembetonan (OpenTofu) memasang wallpaper toko,   │
│  ketika ada renovasi kecil pada pilar gedung, mandor akan MEROBOHKAN TEMBOK │
│  beserta seluruh wallpaper dan barang dagangan di dalamnya (**ACCIDENTAL    │
│  DESTRUCTION & DATA LOSS**)!                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Realitas Data Center On-Premises: The Full Bare-Metal Stack

Di lingkungan cloud publik (seperti AWS EKS), Anda dimanjakan oleh otomatisasi tak terlihat: VPC sudah memiliki gateway internet, EBS volume otomatis terpasang dengan satu baris kode, dan Load Balancer otomatis memiliki IP publik.

Di lingkungan **On-Premises / Private Cloud**, tim SRE bertanggung jawab mengelola seluruh tumpukan teknologi dari bawah ke atas:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       THE ON-PREMISES SRE TECH STACK                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ LAPISAN 5: APLIKASI & GITOPS ]   ──► ArgoCD, Helm, Traefik, MetalLB      │
│  [ LAPISAN 4: CONTAINER ORCHESTRATION]► Kubernetes (k3s HA Cluster, etcd)   │
│  [ LAPISAN 3: OS CONFIG & HARDENING]──► Ansible (SSH, sysctl, security)     │
│  [ LAPISAN 2: VM & GUEST PROVISIONING]► OpenTofu + Cloud-Init               │
│  [ LAPISAN 1: HYPERVISOR & HARDWARE]──► Proxmox VE / VMware vSphere / KVM   │
│  [ LAPISAN 0: PHYSICAL NETWORK & SAN]─► Switch ToR L2/L3, BGP, NFS/Ceph     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Untuk mengorkestrasi Lapisan 2 (VM Provisioning) di atas perangkat keras Lapisan 1, OpenTofu membutuhkan **Provider Plugin On-Premises**.

---

## 3. Peta Provider On-Premises Utama

Berikut adalah perbandingan 5 adapter provider yang paling umum digunakan untuk mengelola infrastruktur lokal dan data center privat:

| Provider Plugin | Ekosistem Target | Karakteristik & Mekanisme Kerja | Batasan & Pertimbangan SRE |
|---|---|---|---|
| **Proxmox VE Provider** (`bpg/proxmox` / `telmate`) | Proxmox Virtual Environment (KVM + LXC) | Berkomunikasi via Proxmox REST API port 8006. Mengkloning VM dari template Cloud-Init, mengatur disk VirtIO, dan binding ke Linux bridge (`vmbr0`). | Membutuhkan API Token bertaraf PVE Administrator atau custom RBAC role. Sangat populer untuk homelab dan enterprise tier-2. |
| **VMware vSphere Provider** (`hashicorp/vsphere`) | VMware ESXi & vCenter Server | Berkomunikasi via vSphere Web Services API. Mendukung datastore cluster, DRS anti-affinity rules, dan distributed virtual switches (vDS). | Membutuhkan lisensi komersial VMware vCenter. Standar de facto industri perbankan dan enterprise tier-1. |
| **Libvirt Provider** (`dmacvicar/libvirt`) | Linux KVM / QEMU Murni | Berkomunikasi langsung ke daemon `libvirtd` lokal via Unix socket (`/var/run/libvirt/libvirt-sock`) atau remote SSH. Sangat ringan tanpa GUI. | Tidak tersedia secara native di macOS Apple Silicon tanpa lapisan virtualisasi Linux perantara. |
| **Cloud-Init Provider / Template** | Inisialisasi OS Linux Guest (Ubuntu, Debian, RHEL) | Menyuntikkan file konfigurasi `#cloud-config` ke dalam disk virtual CD-ROM virtual (NoCloud) yang dibaca kernel saat *first boot*. | **Bukan alat konfigurasi harian.** Hanya bertugas menyalakan SSH, mengatur hostname, dan static IP awal. |
| **Generic HTTP / REST Provider** | IPAM & DCIM Internal (NetBox, phpIPAM) | Melakukan kueri HTTP GET/POST ke sistem inventaris data center untuk memesan alamat IP kosong sebelum VM dinyalakan. | Menjamin tidak terjadi tabrakan alamat IP (*IP Collision*) di jaringan fisik kantor. |

---

## 4. Matriks Batasan Tanggung Jawab (The Boundary Ownership Matrix)

Prinsip utama SRE dalam merancang arsitektur IaC adalah **Strict Separation of Concerns**. Setiap alat hanya boleh memiliki wewenang pada domainnya sendiri:

```text
┌─────────────────────┬────────────────────────────────────┬──────────────────────────────────┐
│ Komponen / Alat     │ TANGGUNG JAWAB MUTLAK              │ DILARANG KERAS MENGERJAKAN       │
├─────────────────────┼────────────────────────────────────┼──────────────────────────────────┤
│ **OpenTofu**        │ - Alokasi vCPU, RAM, dan Ukuran Disk│ - Menginstal package software OS │
│ (Provisioner)       │ - Memilih Target Storage & Node Host│ - Mengedit file konfigurasi app  │
│                     │ - Memasang Virtual NIC ke VLAN ID  │ - Mengelola user/password lokal  │
│                     │ - Mengeluarkan Metadata IP/Hostname│ - Menjalankan instalasi k3s      │
├─────────────────────┼────────────────────────────────────┼──────────────────────────────────┤
│ **Cloud-Init**      │ - Menyetel Hostname saat boot awal │ - Manajemen konfigurasi jangka   │
│ (Guest Seed)        │ - Menaruh SSH Public Key operator  │   panjang / pembaruan harian     │
│                     │ - Menyetel static IP / DHCP client │ - Menjalankan script berat       │
├─────────────────────┼────────────────────────────────────┼──────────────────────────────────┤
│ **Ansible**         │ - OS Hardening & Security Baseline │ - Mengubah ukuran partisi disk   │
│ (Config Management) │ - Konfigurasi Firewall & Sysctl    │   fisik di tingkat hypervisor    │
│                     │ - Instalasi Prasyarat Containerd   │ - Menggantikan fungsi remote     │
│                     │ - Instalasi & Bootstrap Kluster k3s│   state backend OpenTofu         │
├─────────────────────┼────────────────────────────────────┼──────────────────────────────────┤
│ **Kubernetes k3s**  │ - Manajemen Siklus Hidup Pod & App │ - Mengelola mesin virtual host   │
│ (Orchestrator)      │ - Service Discovery, DNS Internal  │ - Mengonfigurasi interface fisik │
│                     │ - Integrasi VIP MetalLB (L2/BGP)   │   kabel jaringan data center     │
└─────────────────────┴────────────────────────────────────┴──────────────────────────────────┘
```

---

## 5. Menghindari Bencana: Pola Konfigurasi Provider di Root Module

Mengapa kita harus mengonfigurasi provider di *Root Module* dan bukan di *Child Module*? Karena di lingkungan on-premises, kredensial dan endpoint hypervisor berbeda antara kluster lab dan kluster produksi.

### Contoh Pola Arsitektur HCL yang Bersih (`environments/prod/providers.tf`):

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    # Contoh deklarasi provider Proxmox resmi
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.60.0"
    }
  }
}

# Provider dikonfigurasi HANYA di Root Module
provider "proxmox" {
  # Endpoint API Proxmox VE Data Center
  endpoint = var.proxmox_api_endpoint

  # Kredensial dibaca dari Environment Variable (BUKAN HARDCODED!):
  # PROXMOX_VE_USERNAME dan PROXMOX_VE_API_TOKEN
  insecure = false # Wajib true hanya jika menggunakan sertifikat SSL valid

  ssh {
    agent    = true
    username = "root"
  }
}
```

### Memanggil Child Module On-Premises:
```hcl
module "k3s_master_nodes" {
  source = "../../modules/proxmox-vm"

  for_each = var.master_nodes_topology

  vm_name     = each.key
  node_target = each.value.target_pve_node # Node Proxmox fisik (e.g. pve-01, pve-02)
  cores       = each.value.cores
  memory_mb   = each.value.memory
  disk_gb     = each.value.disk_size
  vlan_id     = var.management_vlan_id

  # Meneruskan data Cloud-Init untuk first-boot
  cloud_init_user_data = templatefile("${path.module}/templates/user-data.yaml.tftpl", {
    hostname       = each.key
    ssh_public_key = var.operator_ssh_public_key
  })
}
```

---

## 6. Anatomi Guest Initialization: Cloud-Init Non-Secret

File Cloud-Init adalah lembaran instruksi pertama yang dibaca oleh sistem operasi Linux saat mesin virtual pertama kali dinyalakan.

### Template Bersih: `templates/user-data.yaml.tftpl`
Perhatikan bagaimana template ini dirancang secara higienis tanpa menyimpan password:

```yaml
#cloud-config
# INI ADALAH TEMPLATE NON-SECRET GUEST SEED
hostname: ${hostname}
fqdn: ${hostname}.sre.internal
manage_etc_hosts: true

# Buat user operator non-root untuk Ansible
users:
  - name: sre-operator
    gecos: SRE Automation User
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: true # Kunci password! Login HANYA boleh menggunakan SSH Key!
    ssh_authorized_keys:
      - ${ssh_public_key}

# Paket dasar minimal yang dibutuhkan untuk handoff ke Ansible
packages:
  - curl
  - ca-certificates
  - qemu-guest-agent
  - python3

# Pastikan qemu-guest-agent langsung berjalan agar IP VM terlapor ke OpenTofu
runcmd:
  - [ systemctl, enable, --now, qemu-guest-agent ]
```

> [!TIP]
> **Mengapa `qemu-guest-agent` Sangat Krusial?**
> Tanpa *QEMU Guest Agent* di dalam VM, hypervisor Proxmox tidak akan bisa mengetahui berapa alamat IP yang diperoleh VM dari DHCP. Akibatnya, OpenTofu akan mengalami *timeout* saat mencoba membaca atribut `ip_address` untuk diekspor ke Ansible!

---

## 7. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Gejala Masalah | Titik Investigasi | Penyebab Akar (*Root Cause*) | Solusi Pemulihan SRE |
|---|---|---|---|
| `Error: Timeout waiting for VM IP address` | `tofu apply` pada pembuatan VM | Qemu Guest Agent belum terpasang di base image, atau DHCP server lokal lambat membagikan IP. | Pastikan base template memiliki `qemu-guest-agent` atau gunakan konfigurasi IP statis pada Cloud-Init. |
| `Plan ingin me-replace seluruh VM (-/+)` | `tofu plan` pasca edit template | Mengubah nama template induk atau ID storage datastore yang bersifat `ForceNew`. | Teliti output plan. Jangan apply perubahan template induk pada VM yang sedang berjalan di produksi. |
| `Error: 401 Unauthorized / Ticket expired` | Komunikasi provider ke API hypervisor | Token API Proxmox atau vCenter kedaluwarsa atau jam sistem mengalami pergeseran (*clock skew*). | Sinkronkan waktu server dengan NTP/chrony dan terbitkan ulang token API baru. |
| `Ansible gagal SSH setelah VM selesai dibuat` | Port 22 SSH pada target VM | Cloud-Init belum selesai mengeksekusi penambahan user saat Ansible pertama kali mencoba masuk. | Tambahkan *wait-for-ssh readiness gate* di skrip pipeline sebelum memanggil Ansible playbook. |

---

## 8. Ringkasan Praktis & Cheatsheet Provider On-Premises

```bash
# 1. Memeriksa daftar provider yang dibutuhkan dan versinya
tofu providers

# 2. Mengekspor kredensial provider secara aman via environment variable
export PROXMOX_VE_API_TOKEN="USER@REALM!TOKENID=UUID..."

# 3. Memeriksa apakah template Cloud-Init bebas dari sintaks error sebelum apply
cloud-init schema --config-file user-data.yaml

# 4. Melakukan refresh status fisik VM dari hypervisor tanpa mengubah apa pun
tofu apply -refresh-only
```

---

## 9. Cek Pemahaman Mandiri

1. Jelaskan mengapa seorang engineer SRE dilarang keras menggunakan OpenTofu untuk menginstal package sistem operasi (seperti `apt-get install nginx`) atau mengonfigurasi file aplikasi di dalam server produksi!
2. Apa fungsi utama dari teknologi *Cloud-Init* dalam arsitektur provisioning bare-metal on-premises? Di titik manakah tanggung jawab Cloud-Init berakhir dan tanggung jawab Ansible dimulai?
3. Mengapa komponen *QEMU Guest Agent* (atau VMware Tools) mutlak wajib berjalan di dalam sistem operasi mesin virtual saat diorkestrasi oleh OpenTofu? Apa dampaknya terhadap atribut output jika agen ini mati?
4. Mengapa konfigurasi endpoint API dan token autentikasi hypervisor HANYA boleh diletakkan di dalam *Root Module* dan DILARANG KERAS diletakkan di dalam *Child Module*?
5. Jelaskan risiko yang terjadi jika template mesin virtual dasar (*golden template*) di hypervisor diperbarui versinya, sementara OpenTofu dikonfigurasi untuk mereferensikan template tersebut secara dinamis!

---

*Langkah Selanjutnya:* Masuk ke [02 — Simulasi Lokal: Docker, libvirt, dan Mock](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/02-simulasi-lokal-docker-libvirt-mock.md) untuk mempelajari bagaimana melakukan simulasi arsitektur provisioning on-premise secara cepat, aman, dan hemat biaya di atas laptop Mac Apple Silicon Anda.
