# 04 — State Management: Remote Backend MinIO, Locking, Adopsi Import, dan Rekonsiliasi Drift

> **Ringkasan:** File state (`terraform.tfstate`) adalah basis data paling berharga sekaligus paling sensitif di dalam ekosistem OpenTofu. Ia bertindak sebagai pemegang hak milik tunggal (*ownership registry*) yang memetakan kode HCL ke objek infrastruktur fisik. Dokumen ini membedah struktur internal state, membandingkan risiko local state vs keunggulan remote state S3-compatible (MinIO), memahami mekanisme *State Locking* untuk mencegah tabrakan eksekusi (*The Lost Update Problem*), melakukan adopsi infrastruktur lama via `tofu import`, operasi refactoring via `tofu state mv`, hingga dua strategi penanganan *Configuration Drift*.

---

## 1. Analogi Pemula: Buku BPKB Kendaraan & Lampu Merah Persimpangan

Untuk memahami peran state dan penguncian (*locking*), mari pelajari dua analogi penting berikut:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DUA ANALOGI KUNCI: STATE & LOCKING                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ ANALOGI 1: BUKU BPKB KEPEMILIKAN MOBIL (STATE FILE) ]                    │
│  - Mobil fisik berwarna merah terparkir di garasi Anda (Infrastruktur Riil). │
│  - Di lemari tersimpan Buku BPKB resmi kepolisian (**terraform.tfstate**).  │
│  - BPKB mencatat: "Mobil merek X nomor rangka 123 adalah milik Bapak Budi". │
│                                                                             │
│  Jika BPKB hilang:                                                          │
│  Mobil merah tetap ada di garasi secara fisik, tetapi di mata hukum Anda    │
│  tidak bisa membuktikan kepemilikannya! OpenTofu yang kehilangan state      │
│  akan menganggap garasi kosong melompong, lalu mencoba membeli mobil baru   │
│  dengan nama yang sama (menyebabkan error tabrakan resource)!               │
│                                                                             │
│  [ ANALOGI 2: LAMPU MERAH DI PERSIMPANGAN TOL (STATE LOCKING) ]             │
│  Dua engineer (Budi dan Ani) sama-sama ingin melakukan perubahan kluster:   │
│  - Budi ingin menambah 2 kontainer baru.                                    │
│  - Ani ingin mengubah nama network.                                         │
│                                                                             │
│  Tanpa Lampu Merah (No State Lock):                                         │
│  Budi dan Ani mengeksekusi 'apply' di detik yang sama. Keduanya menimpa     │
│  file state bersamaan. Mobil tabrakan beruntun di persimpangan! File state  │
│  rusak terkorupsi, dan perubahan salah satu orang hilang (*Lost Update*).   │
│                                                                             │
│  Dengan Lampu Merah (State Locking Aktif):                                  │
│  Ketika Budi mulai jalan, lampu merah menyala (State LOCKED). Perintah Ani  │
│  otomatis ditahan dengan sabar: "Menunggu giliran, jalur sedang dipakai!".  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Apa Sebenarnya Isi File `terraform.tfstate`?

Secara teknis, file state adalah sebuah dokumen berformat **JSON standar** yang dibuat dan diperbarui otomatis oleh OpenTofu. Mari kita intip isi anatominya:

```json
{
  "version": 4,
  "terraform_version": "1.8.0",
  "serial": 12,
  "lineage": "c8b4e1a2-5f3d-4c8e-9b1a-2d3e4f5a6b7c",
  "outputs": {
    "application_url": {
      "value": "http://127.0.0.1:18080",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "docker_container",
      "name": "web_app",
      "provider": "provider[\"registry.opentofu.org/kreuzwerker/docker\"]",
      "instances": [
        {
          "schema_version": 2,
          "attributes": {
            "id": "7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b",
            "name": "sre-lab-web",
            "image": "sha256:d6b69a6...",
            "network_data": [
              {
                "ip_address": "172.17.0.2"
              }
            ]
          }
        }
      ]
    }
  ]
}
```

### Komponen Kunci di Dalam State File:
* `serial`: Nomor urut revisi state. Setiap kali ada perubahan infrastruktur yang berhasil diterapkan, nilai serial bertambah $+1$. Ini digunakan untuk mendeteksi konflik versi.
* `lineage`: UUID unik identitas kluster. Mencegah satu proyek menimpa state milik proyek lain yang berbeda.
* `resources`: Basis data inventaris yang memetakan nama resource di kode HCL (`docker_container.web_app`) dengan ID objek fisik di runtime (`id: 7a8b9c...`).

> [!CAUTION]
> **Peringatan Keamanan Kritis: Bahaya Kebocoran State!**
> Perhatikan bagian `attributes` di atas. OpenTofu mencatat **seluruh atribut fisik yang dikembalikan oleh API provider**, termasuk atribut yang tidak kita tulis di HCL! Jika Anda membuat database RDS atau Secret Kubernetes, password database dan private key akan **TERCATAT SECARA POLOS (PLAINTEXT TELANJANG)** di dalam JSON state ini.
> - **DILARANG KERAS MENG-COMMIT `terraform.tfstate` KE GIT REPOSITORY!**
> - Selalu tambahkan `*.tfstate` dan `*.tfstate.*` ke dalam file `.gitignore`.

---

## 3. Local State vs Remote State S3-Compatible (MinIO)

Secara default, OpenTofu menyimpan file state di harddisk laptop lokal Anda (`terraform.tfstate`). Namun di dunia kerja nyata, pendekatan ini sangat dilarang:

```text
┌─────────────────────────────────┬─────────────────────────────────┐
│ LOCAL STATE (Hanya untuk Uji)   │ REMOTE STATE (Standar Industri) │
├─────────────────────────────────┼─────────────────────────────────┤
│ Tersimpan di laptop masing-     │ Tersimpan terpusat di Object    │
│ masing engineer.                │ Storage (MinIO / S3 on-prem).   │
├─────────────────────────────────┼─────────────────────────────────┤
│ **TIDAK MEMILIKI STATE LOCKING!**│ **STATE LOCKING OTOMATIS!**     │
│ Rawan tabrakan dan korupsi data.│ Mencegah eksekusi bersamaan.    │
├─────────────────────────────────┼─────────────────────────────────┤
│ Rawan hilang jika laptop rusak, │ Dilindungi fitur versioning,    │
│ dicuri, atau partisi terformat. │ replication, dan daily backup.  │
├─────────────────────────────────┼─────────────────────────────────┤
│ Tidak bisa diintegrasikan dengan│ Runner CI/CD dapat mengakses    │
│ pipeline otomatisasi CI/CD.     │ state secara aman via IAM role. │
└─────────────────────────────────┴─────────────────────────────────┘
```

---

## 4. Arsitektur Remote State On-Premises Menggunakan MinIO di OrbStack

Di lingkungan *on-premises* atau *private data center*, kita tidak menggunakan layanan cloud publik AWS S3. Sebagai gantinya, standar industri SRE mengandalkan **MinIO**, yaitu server penyimpanan objek berkinerja tinggi yang kompatibel 100% dengan protokol AWS S3 API.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ARSITEKTUR REMOTE BACKEND MINIO ON-PREM                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Laptop SRE (Mac Host)               Pipeline GitLab CI/CD Runner          │
│   $ tofu plan / apply                      $ tofu apply                      │
│            │                                    │                           │
│            ▼ HTTPS S3 API                       ▼ HTTPS S3 API              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ MinIO S3-Compatible Storage (Berjalan di OrbStack / Bare-Metal)     │   │
│   │ Endpoint: http://minio.internal:9000                                │   │
│   │                                                                     │   │
│   │ Bucket: "sre-tofu-state"                                            │   │
│   │ └── Key: "modul-3.1/lab-01/terraform.tfstate" (Disimpan & Dilock)   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Konfigurasi Deklaratif Backend S3 di HCL (`versions.tf`):
Perhatikan bagaimana kita mengonfigurasi backend S3 agar mengarah ke MinIO lokal tanpa menaruh password di dalam file:

```hcl
terraform {
  required_version = ">= 1.6.0"

  # Konfigurasi Remote State menggunakan protokol S3
  backend "s3" {
    bucket                      = "sre-tofu-state"
    key                         = "modul-3.1/lab-01/terraform.tfstate"
    region                      = "us-east-1"                 # Nilai formalitas untuk driver S3
    endpoint                    = "http://127.0.0.1:9000"     # Alamat endpoint MinIO lokal
    skip_credentials_validation = true                       # Abaikan verifikasi AWS STS
    skip_region_validation      = true                       # Abaikan validasi region AWS
    skip_requesting_account_id  = true                       # Abaikan AWS Account ID check
    use_path_style              = true                       # Wajib untuk MinIO (bucket di path URL)
  }
}
```

### Memberikan Kredensial Secara Aman Tanpa Membocorkan Secret:
OpenTofu driver S3 secara otomatis akan membaca kredensial dari environment variable sistem operasi:

```bash
# Set kredensial MinIO di sesi terminal (Jangan ditulis di HCL!):
$ export AWS_ACCESS_KEY_ID="minioadmin"
$ export AWS_SECRET_ACCESS_KEY="minioadmin"

# Lakukan inisialisasi remote backend:
$ tofu init
Initializing the backend...
Successfully configured the backend "s3"! OpenTofu will now use this backend.
```

---

## 5. State Locking & Penanganan Insiden Concurrency

Ketika sebuah proses `tofu apply` dimulai, OpenTofu membuat berkas kunci (*lock object*) di backend remote yang berisi metadata:
* **Lock ID:** ID kriptografi acak (misal: `7c8a9b-1122-3344...`)
* **Operation:** Jenis operasi (`OperationTypeApply`)
* **Who:** Username dan hostname mesin yang sedang mengeksekusi (`muzma@macbook-air`)
* **Created:** Waktu kapan kunci mulai aktif.

### Apa yang Terjadi Jika Ada Dua Orang Meng-Apply Bersamaan?
Engineer kedua yang mencoba mengeksekusi apply akan langsung ditolak dengan pesan:

```text
Error: Error acquiring the state lock

Lock Info:
  ID:        7c8a9b-1122-3344-5566-778899aabbcc
  Path:      sre-tofu-state/modul-3.1/lab-01/terraform.tfstate
  Operation: OperationTypeApply
  Who:       muzma@macbook-air
  Version:   1.8.0
  Created:   2026-10-17 14:35:10 UTC

OpenTofu acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again.
```

### Prosedur Standar SRE Menangani State Lock Macet:
Sering kali proses apply di laptop seorang engineer terputus karena laptop kehabisan baterai atau koneksi Wi-Fi putus di tengah jalan, meninggalkan status lock menggantung (*stale lock*).

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 SOP PENANGANAN STATE LOCK MACET                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. BACA METADATA: Periksa kolom 'Who' dan 'Created'. Siapa yang memegang?   │
│ 2. KONFIRMASI TIM: Hubungi orang yang bersangkutan (misal via Slack/Teams): │
│    "Apakah kamu sedang menjalankan apply di modul-3.1?".                    │
│ 3. PASTIKAN PROSES MATI: Pastikan proses apply di laptopnya benar-benar     │
│    sudah mati dan tidak sedang menulis data.                                │
│ 4. FORCE UNLOCK TERUKUR: Jalankan perintah force-unlock HANYA dengan        │
│    menyertakan Lock ID spesifik yang tertera di pesan error!                │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Perintah pelepasan kunci darurat (Emergency Force Unlock)
$ tofu force-unlock 7c8a9b-1122-3344-5566-778899aabbcc
Do you really want to force-unlock this lock?
  OpenTofu will remove the lock on the remote state.
  This can cause state corruption if another process is currently running!

  Enter a value: yes

Lock 7c8a9b-1122-3344-5566-778899aabbcc successfully unlocked.
```

---

## 6. Adopsi Infrastruktur Lama (Brownfield) via `tofu import`

Di dunia nyata, seorang SRE jarang sekali memulai proyek di padang rumput hijau yang bersih (*greenfield*). Sering kali perusahaan sudah memiliki ratusan server dan kontainer yang dibuat manual oleh tim lama bertahun-tahun lalu (*legacy/brownfield infrastructure*).

Bagaimana cara memasukkan server manual tersebut ke dalam kendali OpenTofu tanpa mematikan aplikasinya? Menggunakan perintah **`tofu import`**!

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ALUR 3-LANGKAH RESMI TOFU IMPORT                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Langkah 1: Tulis blok deklarasi resource kosong di file main.tf            │
│  Langkah 2: Jalankan perintah 'tofu import' dengan menyertakan ID fisiknya  │
│  Langkah 3: Lengkapi atribut di HCL hingga 'tofu plan' menghasilkan 0 diff  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Praktek Langkah-Demi-Langkah:

#### Langkah 1: Buat Kerangka Resource Kosong di `main.tf`
```hcl
# Definisikan resource tanpa atribut lengkap dahulu
resource "docker_container" "legacy_web" {
  # Atribut akan diisi setelah import berhasil
}
```

#### Langkah 2: Eksekusi Perintah Import
Cari ID atau nama kontainer yang dibuat manual di Docker host (misal kontainer bernama `my-old-nginx`), lalu jalankan import:

```bash
$ tofu import docker_container.legacy_web my-old-nginx
docker_container.legacy_web: Importing from ID "my-old-nginx"...
docker_container.legacy_web: Import prepared!
  Prepared docker_container for import
docker_container.legacy_web: Refreshing state... [id=8f2a1b3c4d5e...]

Import successful!
The resources that were imported are shown above. These resources are now in
your OpenTofu state; will be managed by OpenTofu from now on.
```

#### Langkah 3: Inspeksi Atribut & Cocokkan Kode HCL
Import hanya mengisi data ke dalam file state, tetapi **belum menulis kode HCL secara lengkap**. Kita membaca data dari state:

```bash
$ tofu state show docker_container.legacy_web
# Catat atribut penting seperti: image, name, ports
```

Lengkapi kode di `main.tf`:
```hcl
resource "docker_container" "legacy_web" {
  name  = "my-old-nginx"
  image = "nginx:1.25-alpine"

  ports {
    internal = 80
    external = 8081
  }
}
```

Jalankan `tofu plan` untuk memverifikasi kesempurnaan adopsi:
```bash
$ tofu plan
No changes. Your infrastructure matches the configuration.
```
Status **No changes** adalah pembuktian bahwa kontainer lama kini 100% resmi diadopsi ke dalam OpenTofu tanpa pernah mengalami *downtime* sedetik pun!

---

## 7. Operasi Bedah State: `state mv` dan `state rm`

Terkadang kita perlu melakukan pemeliharaan internal pada struktur kode (*refactoring*).

### 1. Mengubah Nama Resource Tanpa Merusak Fisik (`tofu state mv`)
Jika Anda mengganti nama resource di HCL dari `docker_container.web` menjadi `docker_container.frontend_app`, OpenTofu secara default akan **MENGHANCURKAN kontainer lama dan MEMBUAT kontainer baru**!

Untuk mencegah kehancuran ini, lakukan pemindahan alamat kepemilikan di state:

```bash
# Pindahkan alamat kepemilikan di dalam state
$ tofu state mv docker_container.web docker_container.frontend_app
Move "docker_container.web" to "docker_container.frontend_app"
Successfully moved 1 object(s).

# Jalankan plan untuk membuktikan tidak ada objek yang dihancurkan
$ tofu plan
No changes. Your infrastructure matches the configuration.
```

### 2. Melepas Kendali Objek Tanpa Menghapusnya (`tofu state rm`)
Jika sebuah resource ingin dikeluarkan dari manajemen OpenTofu agar dikelola manual atau dipindahkan ke repositori lain tanpa mematikan mesinnya di dunia nyata:

```bash
$ tofu state rm docker_container.legacy_web
Removed docker_container.legacy_web
Successfully removed 1 resource instance(s).
```
Kontainer `my-old-nginx` akan tetap hidup dan melayani pengguna di server fisik, namun OpenTofu tidak lagi mengawasi atau mengelolanya.

---

## 8. Deteksi & Rekonsiliasi Configuration Drift

**Configuration Drift** adalah fenomena di mana konfigurasi fisik di server nyata berubah melenceng dari apa yang tercatat di dalam repositori Git HCL (misalnya ada teknisi junior yang mengubah port kontainer via Docker CLI tengah malam).

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 DUA STRATEGI SRE MERESPONS PERGESERAN (DRIFT)               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ STRATEGI A: REKONSILIASI KEMBALI KE HCL (TOLAK PERUBAHAN MANUAL) ]       │
│  Jika perubahan manual tersebut ilegal atau melanggar standar:              │
│  Cukup jalankan: $ tofu apply                                               │
│  OpenTofu akan otomatis menimpa balik perubahan manual tersebut dan         │
│  mengembalikan kondisi server persis sesuai cetak biru HCL di Git!          │
│                                                                             │
│  [ STRATEGI B: ADOPSI KE DALAM HCL (SETUJUI PERUBAHAN MANUAL) ]             │
│  Jika perubahan manual tersebut adalah perbaikan darurat insiden yang sah:  │
│  1. Edit file main.tf di Git agar nilainya cocok dengan kondisi fisik baru. │
│  2. Jalankan $ tofu plan untuk memverifikasi 'No changes'.                  │
│  3. Commit dan merge perubahan HCL tersebut ke branch main.                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Matriks Pemecahan Masalah (Troubleshooting) State

| Gejala Masalah | Titik Investigasi | Penyebab Akar (*Root Cause*) | Langkah Solusi SRE |
|---|---|---|---|
| `Error acquiring the state lock` | `tofu apply` | Ada proses apply lain yang sedang berjalan atau sesi sebelumnya terputus tidak bersih. | Verifikasi tidak ada proses aktif, lalu jalankan `tofu force-unlock <LOCK-ID>`. |
| `Import successful but plan shows many changes` | `tofu plan` pasca import | Blok resource HCL yang ditulis belum mencantumkan seluruh atribut aktual yang ada di server. | Jalankan `tofu state show <address>` dan samakan atribut HCL dengan output state. |
| `Error: Failed to get existing workspaces: S3 bucket does not exist` | `tofu init` | Bucket MinIO yang dideklarasikan di blok backend HCL belum dibuat di server MinIO. | Buat bucket terlebih dahulu melalui MinIO Console web atau MinIO Client CLI (`mc mb`). |
| File `terraform.tfstate` terlanjur ter-commit ke Git | `git log` | File `.gitignore` belum dibuat saat pertama kali inisialisasi git. | Hapus dari cache git via `git rm --cached *.tfstate`, tambahkan ke `.gitignore`, dan segera putar (*rotate*) seluruh password/kredensial yang tercatat di state! |

---

## 10. Ringkasan Praktis & Cheatsheet State Management

```bash
# 1. Menampilkan seluruh resource yang terdaftar di state saat ini
tofu state list

# 2. Membedah seluruh atribut mendalam sebuah resource dari state
tofu state show <resource_type>.<resource_name>

# 3. Mengubah nama resource di state tanpa menyentuh server fisik
tofu state mv <nama_lama> <nama_baru>

# 4. Menghapus resource dari state tanpa membunuh objek di server
tofu state rm <resource_address>

# 5. Memaksa pelepasan kunci state yang macet akibat koneksi putus
tofu force-unlock <LOCK-ID>

# 6. Memeriksa perbedaan konfigurasi vs realitas fisik tanpa mengubah apa pun
tofu plan -refresh-only
```

---

## 11. Cek Pemahaman Mandiri

1. Mengapa file `terraform.tfstate` dianggap sebagai artefak paling berbahaya jika sampai bocor ke publik? Informasi apa saja yang tersimpan di dalamnya meskipun kita tidak menuliskan password di file HCL?
2. Jelaskan bahaya fenomena *The Lost Update Problem* jika dua orang engineer menjalankan perintah `tofu apply` secara bersamaan pada proyek yang menggunakan *Local State* tanpa mekanisme *Locking*!
3. Sebutkan urutan 3 langkah standar SRE untuk mengadopsi kontainer lama yang dibuat manual (*brownfield*) ke dalam pengelolaan OpenTofu menggunakan perintah `tofu import`!
4. Kapan seorang SRE harus menggunakan perintah `tofu state mv` dibandingkan langsung mengganti nama resource di file `main.tf`? Apa bahayanya jika kita langsung mengganti nama di `main.tf` tanpa `state mv`?
5. Apa yang dimaksud dengan *Configuration Drift*? Jelaskan dua strategi berbeda yang dapat diambil oleh tim SRE saat mendeteksi adanya pergeseran konfigurasi di server produksi!

---

*Selamat! Anda telah menguasai seluruh fondasi teori OpenTofu.* Lanjutkan ke [LAB-01 — Docker Web Server](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.1-dasar-opentofu/lab/LAB-01-tofu-docker-web-server.md) dan [LAB-02 — Remote State MinIO, Import & Drift](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.1-dasar-opentofu/lab/LAB-02-state-minio-import-drift.md) untuk mempraktikkan seluruh alur kerja ini secara langsung di terminal Anda!
