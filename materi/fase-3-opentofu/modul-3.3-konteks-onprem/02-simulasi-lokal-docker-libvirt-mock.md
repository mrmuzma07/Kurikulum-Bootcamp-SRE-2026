# 02 — Simulasi Provisioning Lokal: Docker, libvirt, dan Mock Contract di Apple Silicon

> **Ringkasan:** Tidak semua engineer SRE memiliki akses langsung ke rak server fisik seharga ratusan juta rupiah saat sedang belajar atau merancang arsitektur. Oleh karena itu, kemampuan membangun lingkungan simulasi lokal (*local simulation*) yang cepat, terisolasi, dan hemat sumber daya di atas laptop Mac Apple Silicon (ARM64) adalah keterampilan esensial. Dokumen ini membedah batas realitas antara simulator kontainer Docker, mesin virtual KVM/libvirt, dan server bare-metal produksi sejati, menyusun *predicted plan* yang bebas kredensial, serta merancang jalur verifikasi statis (*static lane*) saat runtime lokal tidak tersedia.

---

## 1. Analogi Pemula: Simulator Pesawat Komersial vs Menerbangkan Boeing 737 Asli

Bagi pemula, sering muncul pertanyaan skeptis:
> *"Jika di modul ini kita menggunakan kontainer Docker di laptop Mac untuk menyimulasikan server Proxmox on-premises, apakah ilmu yang saya pelajari benar-benar terpakai saat nanti memegang server fisik di data center?"*

Mari kita gunakan analogi pelatihan pilot penerbangan:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI SIMULATOR KOKPIT VS PENERBANGAN NYATA               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ SIMULATOR KOKPIT PENERBANGAN DI DARAT (DOCKER / MOCK SIMULATOR) ]        │
│  Calon pilot duduk di simulator kokpit hidrolik di dalam ruangan kelas:     │
│  - Tombol instrumen, tuas kemudi, layar radar, dan radio persis seperti     │
│    aslinya (**Sintaksis HCL, Struktur Modul, Validasi Variabel, Output**). │
│  - Pilot melatih memori otot (*muscle memory*): Jika alarm berbunyi, tombol │
│    mana yang ditekan (**Workflow init, plan, apply, refactoring state**).   │
│  - Manfaat: Latihan bisa diulang 100 kali sehari, gratis, tidak membuang    │
│    avtur, dan jika simulator jatuh (crash), tidak ada yang celaka!          │
│                                                                             │
│  [ PENERBANGAN ASLI DENGAN 300 PENUMPANG (BARE-METAL ON-PREM DATA CENTER) ] │
│  Ketika pilot menerbangkan pesawat Boeing 737 sungguhan di angkasa:         │
│  - Prosedur dan cara membaca indikator radar PERSIS SAMA dengan simulator.  │
│  - NAMUN ADA HUKUM FISIKA NYATA YANG TIDAK ADA DI SIMULATOR:                │
│    Ada turbulensi angin badai nyata (**Kernel Linux, Systemd, Storage SAN,  │
│    kabel fisik L2 ARP MetalLB, dan latensi jaringan switch**).              │
│                                                                             │
│  KESIMPULAN SRE: Simulator lokal melatih ketepatan logika kode dan kontrak  │
│  arsitektur Anda; namun seorang SRE tidak boleh naif menganggap simulator   │
│  kontainer Docker adalah mesin virtual produksi sejati!                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Dua Jalur Praktik: Fast Lane vs Production On-Prem Design Lane

Untuk mengakomodasi perbedaan spesifikasi hardware laptop peserta, materi ini menyediakan dua jalur pembelajaran:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DUA JALUR PEMBELAJARAN MODUL 3.3                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ JALUR A: LOCAL FAST LANE (ORBSTACK / DOCKER DI MACBOOK ARM64) ]          │
│  - Target: Eksekusi cepat, instan, hemat baterai, dan disposable.           │
│  - Alat: OpenTofu + kreuzwerker/docker provider.                            │
│  - Yang diuji: Pembuatan modul host multi-instance (for_each), pengujian    │
│    port mapping, isolasi network, dan ekstraksi metadata inventory JSON.    │
│                                                                             │
│  [ JALUR B: PRODUCTION ON-PREM DESIGN LANE (STATIC CODE & ARCHITECTURE) ]   │
│  - Target: Desain arsitektur enterprise untuk Data Center On-Premises.      │
│  - Alat: Review kode Proxmox VE / vSphere HCL, Cloud-Init, dan Preflight.   │
│  - Yang diuji: Rantai pembuktian (*evidence chain*), kesiapan jaringan VLAN,│
│    perhitungan quorum etcd, dan prosedur handoff ke Ansible & k3s.          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Matriks Realitas Teknis: Apa yang Bisa Dibuktikan vs Apa yang TIDAK Bisa?

Sebagai calon engineer SRE profesional, Anda wajib memahami batas pembuktian (*boundary of proof*) dari masing-masing teknologi simulator:

| Pertanyaan Teknis Operasional | Docker Container di Mac (Fast Lane) | KVM / libvirt VM (Linux Server) | Server Bare-Metal Fisik (Produksi) |
|---|:---:|:---:|:---:|
| **Validasi Sintaksis HCL & Modul Reusable** | ✅ Terbukti 100% | ✅ Terbukti 100% | ✅ Terbukti 100% |
| **Pengujian `for_each` & Stabilitas Alamat State** | ✅ Terbukti 100% | ✅ Terbukti 100% | ✅ Terbukti 100% |
| **Ekstraksi Metadata JSON untuk Ansible Inventory** | ✅ Terbukti 100% | ✅ Terbukti 100% | ✅ Terbukti 100% |
| **Inisialisasi First-Boot via Cloud-Init** | ❌ **Tidak Bisa** | ✅ Terbukti Penuh | ✅ Terbukti Penuh |
| **Multi-Service Systemd Daemon (`systemctl`)** | ❌ **Tidak Bisa** (Single Process) | ✅ Terbukti Penuh | ✅ Terbukti Penuh |
| **Modul Kernel Linux Kustom (Wireguard/VXLAN)** | ❌ **Terbatas** (Kernel Mac) | ✅ Terbukti Penuh | ✅ Terbukti Penuh |
| **Isolasi Broadcast Jaringan Layer 2 (ARP/MetalLB)**| ❌ **Tertahan di Bridge** | ✅ Bisa di-Bridge | ✅ Nyata di Kabel LAN |
| **Penyimpanan Blok SAN / iSCSI / Multipathing** | ❌ **Hanya Volume Folder** | ✅ Disk QCOW2/RAW | ✅ LUN SAN Fisik |

---

## 4. Hands-On: Desain Modul Simulasi Host On-Premises via Docker

Mari kita bangun sebuah proyek simulasi lokal yang memetakan topologi kluster 3 node (1 Control-Plane + 2 Worker) menggunakan Docker Provider.

### 1. Struktur Direktori Proyek Lab
```text
simulasi-onprem/
├── versions.tf
├── variables.tf
├── main.tf
└── outputs.tf
```

---

### 2. File Deklarasi Provider: `versions.tf`
```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.2"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}
```

---

### 3. File Variabel Topologi Host: `variables.tf`
Perhatikan bagaimana variabel ini dirancang menggunakan tipe data `map(object(...))` dengan validasi port yang ketat:

```hcl
variable "simulated_cluster" {
  description = "Topologi server bare-metal yang disimulasikan menggunakan kontainer."
  type = map(object({
    role      = string
    host_port = number
    image     = string
  }))
  default = {
    "k3s-master-01" = {
      role      = "control-plane"
      host_port = 18443
      image     = "nginx:1.27.4-alpine"
    }
    "k3s-worker-01" = {
      role      = "worker"
      host_port = 18081
      image     = "nginx:1.27.4-alpine"
    }
    "k3s-worker-02" = {
      role      = "worker"
      host_port = 18082
      image     = "nginx:1.27.4-alpine"
    }
  }

  validation {
    condition = alltrue([
      for name, host in var.simulated_cluster :
      contains(["control-plane", "worker", "storage"], host.role)
    ])
    error_message = "Role server hanya boleh bernilai: control-plane, worker, atau storage."
  }

  validation {
    condition = alltrue([
      for name, host in var.simulated_cluster :
      host.host_port >= 1024 && host.host_port <= 65535
    ])
    error_message = "Seluruh port yang diekspos harus berada pada rentang aman 1024-65535."
  }
}
```

---

### 4. File Implementasi Resource: `main.tf`
```hcl
# 1. Membuat Jaringan Virtual Bridge Terisolasi (Simulasi VLAN Manajemen)
resource "docker_network" "management_vlan" {
  name = "sre-management-vlan"
  ipam_config {
    subnet  = "172.28.0.0/16"
    gateway = "172.28.0.1"
  }
}

# 2. Mengunduh Image Dasar untuk Simulasi Host
resource "docker_image" "base_host" {
  name         = "nginx:1.27.4-alpine"
  keep_locally = true
}

# 3. Membuat Kontainer yang Mewakili Server Fisik On-Premises
resource "docker_container" "simulated_node" {
  for_each = var.simulated_cluster

  name  = each.key
  image = docker_image.base_host.image_id

  # Menghubungkan ke Jaringan Manajemen
  networks_advanced {
    name = docker_network.management_vlan.name
  }

  # Port Forwarding untuk Simulasi Akses API & Layanan
  ports {
    internal = 80
    external = each.value.host_port
  }

  # Metadata Label SRE untuk Pelacakan
  labels {
    label = "sre.role"
    value = each.value.role
  }
  labels {
    label = "sre.cluster"
    value = "production-simulation"
  }
  labels {
    label = "managed_by"
    value = "opentofu"
  }
}
```

---

### 5. Ekstraksi Metadata Non-Secret untuk Ansible: `outputs.tf`
```hcl
output "ansible_dynamic_inventory" {
  description = "Metadata inventaris non-secret untuk diserahkan ke tahap Ansible."
  value = {
    for name, node in docker_container.simulated_node : name => {
      ansible_host = "127.0.0.1"
      ansible_port = var.simulated_cluster[name].host_port
      role         = var.simulated_cluster[name].role
      internal_ip  = node.network_data[0].ip_address
      status       = "ready-for-bootstrap"
    }
  }
}
```

---

## 5. Menjalankan Jalur Runtime Disposable (Fast Lane)

Mari kita uji cetak biru simulasi ini di terminal MacBook Anda:

```bash
# 1. Inisialisasi provider Docker
$ tofu init

# 2. Periksa kesesuaian format dan validasi sintaks HCL
$ tofu fmt -check
$ tofu validate
Success! The configuration is valid.

# 3. Buat rencana eksekusi dan kunci ke file binary
$ tofu plan -out=simulasi.tfplan
Plan: 5 to add, 0 to change, 0 to destroy.

# 4. Terapkan rencana ke runtime Docker lokal
$ tofu apply simulasi.tfplan
docker_network.management_vlan: Creating...
docker_image.base_host: Creating...
docker_container.simulated_node["k3s-master-01"]: Creating...
docker_container.simulated_node["k3s-worker-01"]: Creating...
docker_container.simulated_node["k3s-worker-02"]: Creating...

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

# 5. Periksa output metadata yang siap diserahkan ke Ansible
$ tofu output -json ansible_dynamic_inventory | jq .
{
  "k3s-master-01": {
    "ansible_host": "127.0.0.1",
    "ansible_port": 18443,
    "internal_ip": "172.28.0.2",
    "role": "control-plane",
    "status": "ready-for-bootstrap"
  },
  "k3s-worker-01": {
    "ansible_host": "127.0.0.1",
    "ansible_port": 18081,
    "internal_ip": "172.28.0.3",
    "role": "worker",
    "status": "ready-for-bootstrap"
  },
  "k3s-worker-02": {
    "ansible_host": "127.0.0.1",
    "ansible_port": 18082,
    "internal_ip": "172.28.0.4",
    "role": "worker",
    "status": "ready-for-bootstrap"
  }
}

# 6. Uji konektivitas fisik ke salah satu node simulasi dari browser/curl Mac
$ curl -sI http://127.0.0.1:18443
HTTP/1.1 200 OK
Server: nginx/1.27.4
```

---

## 6. Prosedur Jalur Statis (Static Verification Lane)

Jika di lingkungan Anda runtime Docker atau binary OpenTofu belum terpasang:
1. **Tetap Lakukan Code Review:** Analisis file `main.tf` dan `variables.tf` secara seksama.
2. **Susun Predicted Plan Table:** Buat tabel prediksi perubahan secara manual:
   * Alamat resource: `docker_container.simulated_node["k3s-master-01"]` ──► *Action: + create*
   * Alamat resource: `docker_network.management_vlan` ──► *Action: + create*
3. **Tandai Status Eksekusi:** Di laporan lab Anda, cantumkan catatan transparan:
   *"Kode HCL telah divalidasi secara logika dan lulus analisis statis; eksekusi runtime ditandai BELUM DIVERIFIKASI karena ketiadaan daemon lokal."*
   Ini adalah sikap integritas teknis tertinggi seorang SRE profesional.

---

## 7. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Gejala Masalah | Titik Investigasi | Penyebab Akar (*Root Cause*) | Solusi Pemulihan SRE |
|---|---|---|---|
| `Error: Bind for 0.0.0.0:18443 failed: port is already allocated` | `tofu apply` pada kontainer | Port host Mac yang diminta di HCL sudah dipakai oleh service Mac lain. | Ubah nomor port di variabel `simulated_cluster` ke port non-root lain (misal 19443). |
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` | `tofu init` / `plan` | Aplikasi OrbStack / Docker Desktop di macOS belum dinyalakan. | Buka aplikasi OrbStack di Mac Anda dan pastikan statusnya *Running*. |
| `Simulasi berjalan sangat lambat dan memakan 100% CPU Mac` | Activity Monitor macOS | Image Docker yang digunakan berarsitektur `linux/amd64` sehingga berjalan lewat emulasi Rosetta 2. | Ganti tag image ke image yang mendukung native Apple Silicon ARM64 (`linux/arm64`). |
| `Error: IPAM failed: address already in use` | `docker_network` | Subnet virtual `172.28.0.0/16` bentrok dengan network Docker lokal yang pernah dibuat sebelumnya. | Ganti nilai subnet di blok `ipam_config` (misal menjadi `172.29.0.0/16`). |

---

## 8. Ringkasan Praktis & Cheatsheet Simulasi Lokal

```bash
# 1. Memeriksa ketersediaan Docker daemon di macOS
docker info --format '{{.OSType}}/{{.Architecture}}'

# 2. Membersihkan seluruh kontainer simulasi lab secara bersih
tofu plan -destroy -out=destroy.tfplan
tofu apply destroy.tfplan

# 3. Menghapus jaringan virtual bridge yang tersisa jika diperlukan
docker network prune -f

# 4. Mengekspor metadata host ke file JSON untuk pengujian Ansible
tofu output -json ansible_dynamic_inventory > inventory.json
```

---

## 9. Cek Pemahaman Mandiri

1. Mengapa seorang SRE dilarang keras menganggap kontainer Docker di laptop Mac sebagai bukti bahwa instalasi sistem operasi dan kluster k3s pada server bare-metal pasti berhasil? Sebutkan 3 batasan teknis Docker dibandingkan mesin virtual sejati!
2. Apa keuntungan utama menggunakan simulator lokal (*Fast Lane*) dalam siklus perancangan arsitektur Infrastructure as Code (IaC)?
3. Jelaskan bagaimana atribut `labels` pada kontainer Docker dapat digunakan untuk menyimulasikan peran (*role*) server fisik di data center!
4. Mengapa pada blok `outputs.tf` kita memilih mengekspor alamat IP internal dan port forward, alih-alih mengekspor seluruh atribut mentah kontainer Docker?
5. Apa yang harus dilakukan oleh seorang SRE jika lingkungan runtime lokal di laptopnya tidak tersedia saat harus menyelesaikan peninjauan kode IaC? Jelaskan konsep *Static Verification Lane*!

---

*Langkah Selanjutnya:* Masuk ke [03 — OpenTofu, Ansible, dan k3s Handoff](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.3-konteks-onprem/03-opentofu-ansible-k3s-handoff.md) untuk mempelajari bagaimana menyambungkan tongkat estafet antara OpenTofu, Ansible, dan k3s secara mulus dan aman tanpa celah kebocoran kredensial.
