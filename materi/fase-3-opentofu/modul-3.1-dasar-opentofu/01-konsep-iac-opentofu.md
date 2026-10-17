# 01 — Konsep IaC dan OpenTofu: Fondasi Otomasi Infrastruktur Deklaratif

> **Ringkasan:** Mengelola infrastruktur secara manual melalui antarmuka web (*ClickOps*) atau eksekusi perintah SSH ad-hoc adalah penyebab utama inkonsistensi sistem, kerentanan keamanan, dan pemadaman layanan di lingkungan produksi. Dokumen ini membedah paradigma *Infrastructure as Code* (IaC), membandingkan pendekatan imperatif vs deklaratif, memahami relasi segitiga antara *Desired State*, *Current State*, dan *State File*, mengupas sejarah lahirnya proyek open-source OpenTofu di bawah Linux Foundation, serta bagaimana IaC menjadi pilar utama dalam disiplin Site Reliability Engineering (SRE).

---

## 1. Masalah Utama: Jebakan "ClickOps" & Server Butiran Salju (Snowflake Servers)

Di era komputasi tradisional, seorang administrator sistem (*sysadmin*) menyiapkan server baru dengan cara:
1. Membuka dashboard virtualisasi atau cloud, lalu mengklik tombol *Create VM*.
2. Memilih ukuran CPU dan RAM berdasarkan perkiraan sesaat.
3. Melakukan SSH ke server, menjalankan puluhan perintah `apt-get install`, mengedit file konfigurasi di `/etc/` dengan `nano` atau `vim`.
4. Jika ada error, mencoba berbagai perintah acak di terminal hingga aplikasi akhirnya berhasil berjalan, tanpa mencatat langkah mana yang sebenarnya menyelesaikan masalah.

### Mengapa Pendekatan Manual Ini Menjadi Bencana Produksi?

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KONSEKUENSI BURUK PENDEKATAN MANUAl                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. FENOMENA SERVER BUTIRAN SALJU (SNOWFLAKE SERVERS)                       │
│     Setiap server menjadi unik dan rapuh, persis seperti butiran salju      │
│     (snowflake)—tidak ada dua server yang benar-benar identik di dunia!     │
│     Tidak ada yang berani menyentuh, me-restart, atau meng-upgrade server   │
│     tersebut karena takut sistem akan rusak permanen dan tidak ada yang     │
│     tahu cara membangunnya kembali dari awal.                               │
│                                                                             │
│  2. CONFIGURATION DRIFT (PERGESERAN KONFIGURASI)                            │
│     Lingkungan Staging dan Production lambat laun menjadi berbeda karena    │
│     ada perbaikan darurat yang dilakukan langsung via SSH di production     │
│     tetapi lupa diterapkan di staging. Akibatnya: "Aplikasi lancar di       │
│     staging, tetapi meledak saat rilis ke production!".                     │
│                                                                             │
│  3. ZERO AUDIT TRAIL & SINGLE POINT OF FAILURE MANUSIA                      │
│     Tidak ada catatan siapa yang mengubah port firewall, kapan perubahan    │
│     dibuat, dan apa alasannya. Ketika engineer yang bersangkutan cuti atau  │
│     pindah kantor, pengetahuan tentang infrastruktur ikut lenyap bersamanya.│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Paradigma Modern SRE: Pets vs Cattle (Hewan Peliharaan vs Hewan Ternak)
* **Pets (Pendekatan Lama):** Server diperlakukan seperti anjing atau kucing kesayangan. Diberi nama unik (`zeus`, `hermes`, `db-prod-legacy`), disayang-sayang, dan jika sakit (rusak), dokter hewan dipanggil untuk merawatnya berhari-hari.
* **Cattle (Pendekatan IaC SRE):** Server diperlakukan seperti ternak bernomor (`worker-01`, `worker-02`, `worker-03`). Jika salah satu server sakit, kita tidak meratapinya—server tersebut langsung dimusnahkan (*destroyed*) dan server baru yang identik langsung dilahirkan dalam hitungan detik menggunakan kode cetak biru yang sama!

---

## 2. Analogi Pemula: Resep Masakan Koki vs Memesan di Restoran

Untuk memahami perbedaan paling fundamental dalam otomatisasi sistem, mari pelajari dua paradigma berpikir: **Imperatif** vs **Deklaratif**.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                  ANALOGI IMPERATIF VS DEKLARATIF                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ PARADIGMA IMPERATIF: RESEP LANGKAH-DEMI-LANGKAH ]                       │
│  Anda masuk ke dapur dan mendiktekan instruksi satu demi satu ke koki:      │
│  1. "Ambil wajan nomor 2."                                                  │
│  2. "Nyalakan kompor dengan api sedang."                                    │
│  3. "Tuangkan minyak goreng 2 sendok makan."                                │
│  4. "Pecahkan sebutir telur ayam."                                          │
│  5. "Aduk selama 3 menit."                                                  │
│                                                                             │
│  Masalah: Jika Anda mengulang perintah ini untuk kedua kalinya, koki akan   │
│  mengambil wajan baru, menyalakan kompor lagi, dan membuat telur kedua!     │
│  Perintah imperatif berfokus pada "BAGAIMANA CARA MELAKUKANNYA" (HOW).      │
│                                                                             │
│  [ PARADIGMA DEKLARATIF: MEMESAN DI RESTORAN MEWAH ]                        │
│  Anda duduk santai di meja makan dan berkata kepada pelayan:                │
│  "Saya ingin sepiring Nasi Goreng Telur Dadar dengan tingkat kepedasan      │
│   sedang tersaji di atas meja saya."                                        │
│                                                                             │
│  Anda tidak peduli merek wajan apa yang dipakai atau tangan mana yang       │
│  mengaduk beras. Yang Anda pedulikan adalah HASIL AKHIR YANG DIINGINKAN.    │
│  Jika pesanan diulang, pelayan akan melihat meja Anda:                      │
│  "Nasi goreng Anda sudah ada di atas meja. Tidak ada yang perlu dibuat."   │
│  Pendekatan deklaratif berfokus pada "HASIL APA YANG DIINGINKAN" (WHAT).    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Komparasi Kode Nyata: Skrip Bash (Imperatif) vs HCL OpenTofu (Deklaratif)

Mari kita buktikan secara teknis mengapa skrip shell imperatif sangat berbahaya untuk mengelola infrastruktur dibandingkan OpenTofu.

### Skenario: Menjalankan Kontainer Web NGINX

#### Pendekatan Imperatif (Bash Script `setup-web.sh`):
```bash
#!/usr/bin/env bash
set -e

# Langkah 1: Buat network
docker network create web-net

# Langkah 2: Jalankan kontainer
docker run -d --name web-server --network web-net -p 8080:80 nginx:1.27-alpine
```

*Apa yang terjadi jika skrip Bash di atas dijalankan untuk kedua kalinya?*
```text
$ ./setup-web.sh
Error response from daemon: network with name web-net already exists
```
Skrip langsung **CRASH dan GAGAL** di baris pertama! Untuk membuatnya aman, Anda harus menulis puluhan baris logika kondisional (*defensive programming*): mengecek apakah network sudah ada (`docker network inspect`), mengecek apakah kontainer sudah jalan (`docker ps`), mengecek apakah port 8080 sudah dipakai, dan seterusnya. Skrip menjadi sangat rumit, kotor, dan sulit dirawat.

---

#### Pendekatan Deklaratif (OpenTofu HCL `main.tf`):
```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

resource "docker_network" "web_net" {
  name = "web-net"
}

resource "docker_container" "web_server" {
  name  = "web-server"
  image = "nginx:1.27-alpine"

  networks_advanced {
    name = docker_network.web_net.name
  }

  ports {
    internal = 80
    external = 8080
  }
}
```

*Apa yang terjadi jika OpenTofu dijalankan untuk pertama kali, kedua kali, atau ke-100 kali?*
- **Eksekusi 1:** OpenTofu mendeteksi network dan kontainer belum ada ──► Membuat keduanya (*1 to add, 0 to change, 0 to destroy*).
- **Eksekusi 2:** OpenTofu memeriksa kondisi riil, melihat network dan kontainer sudah berjalan persis sesuai spesifikasi ──► **No changes! Infrastructure matches the configuration** (Tidak ada yang diubah, zero side-effect).
- **Jika seseorang mengubah port manual via Docker CLI menjadi 9090:** OpenTofu mendeteksi perbedaan (*drift*) ──► Mengembalikan port kontainer ke 8080 agar sesuai dengan cetak biru!

---

## 4. Tiga Realitas Infrastruktur yang Wajib Dipahami SRE

Di dalam ekosistem OpenTofu, ada tiga komponen kondisi (*state*) yang saling berinteraksi:

```text
               ┌─────────────────────────────────────────┐
               │         1. DESIRED CONFIGURATION        │
               │         (Kode HCL di Git: *.tf)         │
               └────────────────────┬────────────────────┘
                                    │
                         tofu plan  │  Rekonsiliasi
                                    ▼
               ┌─────────────────────────────────────────┐
               │             3. STATE FILE               │
               │        (terraform.tfstate)              │
               │   "Memori Rekaman Terakhir OpenTofu"    │
               └────────────────────┬────────────────────┘
                                    │
                         tofu apply │  API Provider
                                    ▼
               ┌─────────────────────────────────────────┐
               │        2. CURRENT INFRASTRUCTURE        │
               │       (Mesin Riil di Cloud/Docker)      │
               └─────────────────────────────────────────┘
```

1. **Desired Configuration (Apa yang Kita Inginkan):** Seluruh file konfigurasi `.tf` yang kita tulis dan simpan di dalam Git repository. Ini adalah cetak biru resmi yang telah melalui proses *code review*.
2. **Current Infrastructure (Kondisi Nyata di Lapangan):** Sumber daya fisik atau virtual yang benar-benar sedang hidup di dalam data center, Proxmox, AWS, atau Docker daemon saat ini.
3. **State File (`terraform.tfstate`):** Basis data internal milik OpenTofu yang mencatat pemetaan (*mapping*) antara nama resource di dalam kode HCL dengan ID unik resource yang diterbitkan oleh API provider (misalnya ID kontainer Docker `7a8b9c...` atau ID instance cloud `i-0123456789`).

> [!IMPORTANT]
> **Fungsi Utama State File:**
> Tanpa file state, OpenTofu tidak akan tahu apakah sebuah kontainer bernama `web-server` di server Anda adalah kontainer yang dibuat oleh OpenTofu minggu lalu, atau kontainer milik engineer lain yang dibuat secara manual. File state adalah "buku sertifikat kepemilikan sah" OpenTofu.

---

## 5. Prinsip Idempotency & Tantangan di Dunia Nyata

Salah satu pilar terpenting dalam Site Reliability Engineering adalah **Idempotency**.

Secara matematis, sebuah fungsi $f$ dikatakan idempoten jika:
$$f(f(x)) = f(x)$$

Artinya: **Mengeksekusi operasi sebanyak satu kali, sepuluh kali, atau seribu kali akan selalu menghasilkan kondisi akhir sistem yang persis sama tanpa efek samping negatif.**

### Apa Saja yang Bisa Merusak Idempotency di Lingkungan Nyata?
Meskipun OpenTofu dirancang idempoten, kesalahan manusia dapat merusak sifat ini:
1. **Menggunakan Tag Image yang Berubah-ubah (`:latest`):** Hari ini `:latest` merujuk ke NGINX 1.25, besok di-pull menjadi NGINX 1.27. Selalu gunakan tag versi spesifik yang *immutable* (misal: `nginx:1.27.4-alpine`) atau sertakan *image digest* (SHA256).
2. **Nilai Acak atau Waktu Dinamis Tanpa Lifecycle:** Menggunakan fungsi `timestamp()` di dalam nama resource akan menyebabkan OpenTofu selalu mendeteksi perubahan setiap kali `tofu plan` dijalankan.
3. **Perubahan Manual di Luar OpenTofu (*Out-of-Band Changes*):** Seseorang melakukan SSH ke server dan mengubah file konfigurasi atau mematikan kontainer secara diam-diam.
4. **API Provider yang Mengalami *Eventual Consistency*:** API cloud membutuhkan waktu beberapa detik untuk mereplikasi data antar-zona, menyebabkan pembacaan state sesaat menghasilkan data lama.

---

## 6. Sejarah & Fakta: Mengapa OpenTofu Lahir? (OpenTofu vs Terraform)

Bagi seorang engineer SRE modern, memahami lanskap lisensi perangkat lunak adalah bagian integral dari mitigasi risiko kelangsungan bisnis (*business continuity*).

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KRONOLOGI LAHIRNYA PROYEK OPENTOFU                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  10 AGUSTUS 2023:                                                           │
│  HashiCorp secara mengejutkan mengubah lisensi Terraform (dan produk        │
│  lainnya) dari lisensi Open-Source murni (Mozilla Public License / MPL 2.0) │
│  menjadi lisensi Business Source License (BUSL 1.1) yang restriktif         │
│  terhadap kompetisi komersial.                                              │
│                                                                             │
│  15 AGUSTUS 2023:                                                           │
│  Komunitas global, ratusan perusahaan teknologi, dan ribuan kontributor     │
│  mempublikasikan "OpenTF Manifesto" yang meminta HashiCorp mengembalikan    │
│  Terraform ke lisensi open-source sejati.                                   │
│                                                                             │
│  5 SEPTEMBER 2023:                                                          │
│  Karena tidak ada kesepakatan, komunitas resmi melakukan fork dari basis    │
│  kode Terraform MPL terakhir (v1.5.7) dan meluncurkan proyek OPENTOFU.      │
│                                                                             │
│  20 SEPTEMBER 2023:                                                         │
│  OpenTofu resmi bergabung di bawah naungan LINUX FOUNDATION sebagai proyek  │
│  netral, bebas vendor, dan dijamin akan selalu bersifat open-source 100%   │
│  selamanya di bawah lisensi MPL 2.0.                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tabel Komparasi Faktual: OpenTofu vs Terraform

| Dimensi Parameter | OpenTofu | Terraform |
|---|---|---|
| **Badan Pengelola** | **Linux Foundation** (Konsorsium Komunitas Netral) | **HashiCorp / IBM** (Korporasi Komersial) |
| **Lisensi Perangkat Lunak**| **Mozilla Public License 2.0 (MPL 2.0)** (Open Source Murni) | **Business Source License (BUSL 1.1)** (Source Available, Non-OSI) |
| **Kompatibilitas Kode** | Mendukung 99.9% sintaks HCL Terraform v1.5.x s.d. v1.8.x | Format standar HCL Terraform bawaan |
| **Dukungan Provider** | Mendukung seluruh provider di OpenTofu Registry & mirror resmi | Terraform Registry resmi milik HashiCorp |
| **Fitur Inovatif Baru** | **Native State Encryption** (Enkripsi state bawaan sejak v1.7 tanpa addon eksternal!) | Fitur enkripsi terikat pada ekosistem berbayar HCP Terraform |
| **Risiko Lisensi Bisnis** | **Nol:** Aman digunakan oleh startup, enterprise, maupun penyedia layanan cloud. | Wajib ditinjau oleh tim legal perusahaan jika membangun platform internal. |

---

## 7. Arsitektur Provider sebagai Jembatan Antarmuka API

OpenTofu adalah mesin orkestrasi yang cerdas, namun OpenTofu sendiri **tidak memiliki kode untuk berbicara langsung ke Docker, Proxmox, atau Kubernetes**.

OpenTofu menggunakan arsitektur modular berbasis **Provider Plugin**:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ARSITEKTUR PLUGIN OPENTOFU                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   File Konfigurasi HCL (*.tf) ──► [ OpenTofu Core Engine ]                  │
│                                             │                               │
│                                             ▼ Protokol gRPC over RPC        │
│                         ┌───────────────────┴───────────────────┐           │
│                         │                                       │           │
│                         ▼                                       ▼           │
│               ┌──────────────────┐                    ┌──────────────────┐  │
│               │ Docker Provider  │                    │ Proxmox Provider │  │
│               │ (Plugin Binary)  │                    │ (Plugin Binary)  │  │
│               └────────┬─────────┘                    └────────┬─────────┘  │
│                        │ Unix Socket                           │ HTTPS API  │
│                        ▼                                       ▼            │
│                 Docker Daemon                          Proxmox VE Cluster   │
│                 (OrbStack Mac)                         (Server Bare-Metal)  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### File Kunci Pengaman: `.terraform.lock.hcl`
Ketika Anda menjalankan perintah `tofu init`, OpenTofu mengunduh binary provider yang sesuai dengan arsitektur laptop Anda (misal `darwin_arm64` untuk Mac Apple Silicon) dan mencatat nilai hash kriptografisnya ke dalam file gembok:

```hcl
# .terraform.lock.hcl
provider "registry.opentofu.org/kreuzwerker/docker" {
  version     = "3.0.2"
  constraints = "~> 3.0"
  hashes = [
    "h1:xxxx...",
    "zh:yyyy...",
  ]
}
```

> [!TIP]
> **Praktik Terbaik Git SRE:**
> File `.terraform.lock.hcl` **WAJIB DI-COMMIT KE GIT**! File ini menjamin bahwa seluruh anggota tim dan pipeline CI/CD akan menggunakan versi binary provider yang persis sama bit-demi-bit, mencegah serangan *supply chain attack* atau perubahan perilaku provider yang tidak terduga.

---

## 8. Integrasi IaC dalam Siklus Hidup SRE

Di lingkungan kerja Site Reliability Engineering kelas dunia, perubahan infrastruktur mengikuti alur kerja yang ketat dan terotomasi (*GitOps & Continuous Delivery*):

```text
 1. Engineer membuat tiket perubahan (Jira/GitLab Issue)
      │
      ▼
 2. Membuat branch Git baru & menulis kode deklaratif HCL (*.tf)
      │
      ▼
 3. Validasi lokal: $ tofu fmt -check && tofu validate
      │
      ▼
 4. Membuka Pull Request (PR) ke branch main
      │
      ▼
 5. Pipeline CI/CD otomatis menjalankan: $ tofu plan
    (Output plan ditempelkan secara otomatis ke komentar PR untuk ditinjau tim)
      │
      ▼
 6. Peer Review & Security Approval oleh Senior SRE (Memeriksa blast radius!)
      │
      ▼
 7. Merge ke branch main ──► Pipeline CD mengeksekusi: $ tofu apply
      │
      ▼
 8. Sistem Observabilitas (Prometheus/Grafana) memantau metrik & mendeteksi drift
```

---

## 9. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Gejala Masalah | Penyebab Akar (*Root Cause*) | Perintah Investigasi | Solusi Pemulihan SRE |
|---|---|---|---|
| **`Incompatible provider architecture` saat `tofu init`** | Provider yang diminta tidak menyediakan binary untuk arsitektur macOS Apple Silicon (`darwin_arm64`). | `tofu version`<br>`uname -m` | Cari provider alternatif yang mendukung ARM64 atau gunakan versi provider yang lebih baru. |
| **`tofu plan` selalu ingin me-replace (`-/+`) objek** | Ada atribut HCL yang bersifat `ForceNew` (misalnya mengganti nama kontainer atau subnet network yang tidak bisa diubah *in-place*). | `tofu plan \| grep -B2 -A5 "must be replaced"` | Pahami bahwa objek lama akan dihancurkan; pastikan tidak ada data persisten yang hilang atau gunakan `lifecycle { prevent_destroy = true }`. |
| **`Configuration drift` tidak terdeteksi di plan** | Objek yang diubah manual berada di luar cakupan resource yang didefinisikan di dalam HCL. | `tofu state list`<br>`docker ps` | Import resource manual tersebut ke dalam OpenTofu atau hapus resource liar yang tidak tercatat di Git. |
| **Eksekusi apply gagal dengan error `Resource already exists`** | Objek fisik sudah ada di dunia nyata (dibuat manual via GUI/CLI sebelumnya), tetapi belum terdaftar di dalam state file OpenTofu. | `docker ps -a`<br>`tofu state list` | Gunakan perintah `tofu import` untuk mengadopsi objek tersebut ke dalam state OpenTofu. |

---

## 10. Ringkasan Konsep Kunci (Cheat Sheet)

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                      RINGKASAN KONSEP IAC & OPENTOFU                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Filosofi Utama   : Perlakukan server seperti Cattle (Ternak), bukan Pet! │
│ 2. Paradigma HCL    : Deklaratif (Fokus pada hasil akhir, bukan urutan cara)│
│ 3. Tiga Realitas    : Desired State (HCL) vs State File vs Current Infra.   │
│ 4. Idempotency      : Eksekusi berulang menghasilkan kondisi akhir identik. │
│ 5. OpenTofu         : Fork open-source resmi Terraform di bawah Linux Fdn.  │
│ 6. Plugin Provider  : Jembatan penerjemah HCL ke API (Docker, Proxmox, K8s).│
│ 7. File Gembok      : .terraform.lock.hcl WAJIB di-commit ke repositori Git.│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Cek Pemahaman Mandiri

1. Jelaskan perbedaan mendasar antara pendekatan imperatif pada skrip Bash dengan pendekatan deklaratif pada HCL OpenTofu saat menangani kondisi di mana sumber daya yang ingin dibuat ternyata sudah ada di server!
2. Mengapa seorang SRE tidak boleh memperlakukan server produksi sebagai *Snowflake Server*? Apa konsekuensi jangka panjangnya terhadap proses *disaster recovery*?
3. Jelaskan hubungan segitiga antara *Desired Configuration*, *Current Infrastructure*, dan *State File* (`terraform.tfstate`)! Apa peran perintah `tofu plan` dalam menghubungkan ketiganya?
4. Apa yang melatarbelakangi lahirnya proyek OpenTofu di bawah Linux Foundation pada tahun 2023? Mengapa aspek lisensi perangkat lunak menjadi perhatian penting bagi tim SRE enterprise?
5. Sebutkan dua faktor di dunia nyata yang dapat merusak sifat *idempotency* pada eksekusi OpenTofu, dan bagaimana cara memitigasinya!

---

*Langkah Selanjutnya:* Masuk ke [02 — HCL, Resource, Variable, Output, Provider](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.1-dasar-opentofu/02-hcl-resource-variable-output-provider.md) untuk membedah sintaksis HCL, merancang variabel input yang aman, menyusun keluaran modular, dan memetakan *dependency graph*.
