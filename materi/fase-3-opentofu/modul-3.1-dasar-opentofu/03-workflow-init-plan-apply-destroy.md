# 03 — Workflow OpenTofu: Siklus Terukur `init → plan → apply → destroy`

> **Ringkasan:** Di lingkungan produksi enterprise, mengeksekusi perubahan infrastruktur bukanlah tindakan coba-coba tanpa pengawasan. OpenTofu memaksakan disiplin pemisahan antara tahap perancangan (*plan*) dan tahap eksekusi (*apply*). Dokumen ini membedah siklus hidup operasional IaC standar industri, memahami kode simbol perubahan pada output `plan` (termasuk bahaya tersembunyi `-/+ replace`), mengamankan rencana eksekusi menggunakan file binary terkomit (*plan files*), melakukan verifikasi runtime pasca-apply, hingga prosedur pembongkaran aman (*graceful destroy*).

---

## 1. Analogi Pemula: Kontraktor Membangun Gedung Bertingkat

Untuk memahami alur kerja standar OpenTofu, bayangkan sebuah proyek konstruksi bangunan sipil bertaraf internasional:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI KONTRAKTOR BANGUNAN & WORKFLOW OPENTOFU             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ TAHAP 1: tofu init (PERSIAPAN LOGISTIK & ALAT BERAT) ]                  │
│  Sebelum batu bata dipasang, mandor mendatangkan mesin molen semen, derek   │
│  crane, dan helm proyek dari gudang pusat ke lokasi tanah (Mengunduh        │
│  binary provider Docker/Proxmox dan mengunci hash di .terraform.lock.hcl).  │
│                                                                             │
│  [ TAHAP 2: tofu fmt & validate (PEMERIKSAAN KELAYAKAN TEKNIK) ]            │
│  Insinyur teknik memeriksa lembaran kalkulasi struktur. Apakah ada rumus    │
│  beban yang salah hitung atau garis gambar yang miring? Jika ada cacat,     │
│  proyek ditahan sebelum semen dicor!                                        │
│                                                                             │
│  [ TAHAP 3: tofu plan (PRESENTASI RENCANA KERJA KE PEMILIK GEDUNG) ]        │
│  Mandor mempresentasikan cetak biru terperinci ke pemilik gedung:           │
│  - "Kami akan menambah 2 pilar beton baru (+)"                              │
│  - "Kami akan mengganti warna cat dinding (~)"                              │
│  - "Kami akan MEROBOHKAN TEMBOK LAMA lalu membangun ulang (-/+)"            │
│  Pemilik gedung dapat meneliti dampak risiko dan biayanya sebelum setuju.   │
│                                                                             │
│  [ TAHAP 4: tofu apply (PELAKSANAAN KONSTRUKSI NYATA) ]                     │
│  Setelah pemilik menandatangani persetujuan, mandor menginstruksikan kuli   │
│  untuk mengecor beton dan memasang bata sesuai cetak biru yang disepakati.  │
│                                                                             │
│  [ TAHAP 5: tofu destroy (PEMBONGKARAN TERTIB PASCA SEWA) ]                 │
│  Ketika masa sewa proyek selesai, seluruh struktur dibongkar secara rapi   │
│  dan lahan dikembalikan ke kondisi semula tanpa merusak kavling tetangga.   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Klasifikasi Perintah: Memahami Profil Risiko Operasi

Sebagai calon engineer SRE profesional, Anda wajib mengetahui tingkat risiko dari setiap perintah yang Anda ketikkan di terminal:

```text
┌─────────────────────┬────────────────────────────────────┬──────────────────────────────────┐
│ Kategori Perintah   │ Contoh Perintah                    │ Karakteristik & Tingkat Risiko   │
├─────────────────────┼────────────────────────────────────┼──────────────────────────────────┤
│ **Read-Only**       │ `tofu fmt -check`                  │ **Aman (Zero Risk).**            │
│ (Hanya Membaca)     │ `tofu validate`                    │ Hanya membaca sintaks lokal atau │
│                     │ `tofu plan`                        │ status infrastruktur tanpa       │
│                     │ `tofu state list`                  │ mengubah sistem fisik sedikit pun│
├─────────────────────┼────────────────────────────────────┼──────────────────────────────────┤
│ **State Mutation**  │ `tofu import`                      │ **Risiko Sedang-Tinggi.**        │
│ (Mengubah Rekaman)  │ `tofu state mv`                    │ Memodifikasi file state pemetaan.│
│                     │ `tofu state rm`                    │ Berpotensi memutus kepemilikan   │
│                     │ `tofu init -migrate-state`         │ objek tanpa menyentuh server riil│
├─────────────────────┼────────────────────────────────────┼──────────────────────────────────┤
│ **Infra Mutation**  │ `tofu apply`                       │ **Kritis (Tinggi).**             │
│ (Mengubah Fisik)    │ `tofu destroy`                     │ Memanggil API provider untuk     │
│                     │                                    │ MEMBUAT, MENGUBAH, atau MEMBUNUH │
│                     │                                    │ mesin/kontainer di dunia nyata!  │
└─────────────────────┴────────────────────────────────────┴──────────────────────────────────┘
```

---

## 3. Tahap 1: `tofu init` (Inisialisasi Lingkungan Proyek)

Setiap kali Anda membuat proyek baru, mengklon repositori Git dari tim lain, atau menambahkan blok `provider` baru di HCL, perintah pertama yang **wajib** dijalankan adalah:

```bash
$ tofu init
```

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       APA YANG DILAKUKAN TOFU INIT?                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Memeriksa blok backend state (apakah local disk atau remote S3 MinIO).    │
│ 2. Membaca versions.tf, lalu mengunduh plugin binary provider yang cocok    │
│    dengan OS Anda (misal darwin_arm64) ke direktori rahasia .terraform/     │
│ 3. Memverifikasi atau membuat file gembok .terraform.lock.hcl.              │
│ 4. Menyiapkan modul-modul eksternal (jika ada).                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Opsi Parameter Krusial pada `tofu init`:
* `tofu init -upgrade`: Memperbarui versi plugin provider yang terpasang ke versi rilis terbaru yang masih diizinkan oleh batasan `version` di HCL, sekaligus memperbarui checksum hash di `.terraform.lock.hcl`.
* `tofu init -reconfigure`: Mengabaikan konfigurasi backend lama yang tersimpan di cache dan menginisialisasi backend baru dari nol.
* `tofu init -migrate-state`: Menyalin seluruh riwayat data state dari backend lama ke backend baru (misal migrasi dari laptop lokal ke remote MinIO S3).

---

## 4. Tahap 2: `tofu fmt` & `tofu validate` (Pemeriksaan Kualitas Statis)

Sebelum mengajukan perubahan ke tim Anda (*Code Review*), lakukan dua tahap audit statis lokal:

### 1. Merapikan Format Kode (`tofu fmt`)
OpenTofu memiliki standar gaya penulisan kode resmi. Jangan biarkan spasi atau indentasi yang berantakan masuk ke Git:

```bash
# Merapikan seluruh file .tf di direktori saat ini secara otomatis
$ tofu fmt

# Di pipeline CI/CD: Memeriksa apakah ada file yang tidak rapi tanpa mengubahnya
$ tofu fmt -check -diff
```

### 2. Memvalidasi Logika Sintaksis (`tofu validate`)
Perintah ini memverifikasi kebenaran sintaksis HCL, kecocokan tipe data variabel, dan ketersediaan atribut yang dideklarasikan:

```bash
$ tofu validate
Success! The configuration is valid.
```

> [!NOTE]
> **Batasan `tofu validate`:**
> Perintah `tofu validate` bersifat **offline dan lokal**. Ia hanya memastikan bahwa kode HCL Anda tidak memiliki kesalahan ketik (*syntax error*). Ia **tidak memeriksa** apakah Docker socket Anda menyala, apakah port 8080 di laptop Anda sedang bentrok, atau apakah password Anda benar.

---

## 5. Tahap 3: `tofu plan` (Pemeriksaan Rencana Perubahan)

`tofu plan` adalah fitur keselamatan paling penting di OpenTofu. Perintah ini membaca kondisi nyata saat ini, membandingkannya dengan kode HCL Anda, lalu menghitung selisih tindakan (*delta/diff*) yang harus dilakukan:

```bash
$ tofu plan
```

```text
OpenTofu will perform the following actions:

  # docker_image.nginx_image will be created
  + resource "docker_image" "nginx_image" {
      + id           = (known after apply)
      + image_id     = (known after apply)
      + keep_locally = true
      + name         = "nginx:1.27.4-alpine"
    }

  # docker_container.web_app will be created
  + resource "docker_container" "web_app" {
      + id           = (known after apply)
      + image        = (known after apply)
      + name         = "sre-lab-web"
      + ports {
          + external = 18080
          + internal = 80
        }
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

### Membedah 4 Simbol Sakral Aksi Eksekusi

| Simbol Tindakan | Arti & Karakteristik | Tingkat Risiko SRE |
|---|---|---|
| **`+ create`** | Objek baru akan dibuat dari nol. Objek ini belum pernah ada sebelumnya. | **Rendah (Aman).** Tidak ada objek yang dimatikan. |
| **`~ update in-place`** | Objek yang ada akan dimodifikasi atributnya tanpa harus mematikan atau menghapus objek fisik (misalnya mengubah tag label atau metadata). | **Rendah-Sedang.** Layanan tetap berjalan tanpa restart. |
| **`-/+ replace`** | **BAHAYA MAKSIMUM!** Objek lama akan **DIHANCURKAN DAN DIMUSNAHKAN DAHULU (`-`)**, baru kemudian objek baru dengan nama yang sama dibuat ulang (`+`). | **KRITIS! Pemicu Utama Downtime & Data Loss!** |
| **`- destroy`** | Objek fisik yang ada saat ini akan dihapus permanen dari dunia nyata karena kodenya dihapus dari file HCL. | **Tinggi.** Pastikan resource memang sudah tidak dipakai. |

> [!CAUTION]
> **Bahaya Laten Simbol `-/+ replace`:**
> Mengapa sebuah resource di-*replace* dan tidak di-*update in-place*? Karena API provider tidak mengizinkan perubahan atribut tersebut saat objek sedang hidup (disebut atribut `ForceNew`).
> *Contoh:* Mengubah subnet pada kartu jaringan VM, atau mengubah parameter `name` pada kontainer Docker. Jika kontainer tersebut menyimpan data di disk lokal dan di-*replace*, **seluruh data di dalamnya akan musnah seketika!** Selalu teliti simbol `-/+` sebelum menyetujui perubahan.

---

### Praktik Terbaik SRE: Mengunci Perubahan dengan Binary Plan File (`-out`)

Di lingkungan produksi modern, mengeksekusi `tofu apply` secara polos tanpa file plan adalah risiko keamanan (*Time-of-Check to Time-of-Use / TOCTOU Race Condition*).

Mengapa? Karena antara waktu Anda membaca output `tofu plan` di layar dengan waktu Anda mengetikkan `yes`, kondisi infrastruktur di cloud/server bisa saja berubah diubah orang lain!

```bash
# 1. Simpan rencana eksekusi matematis ke dalam file binary terenkripsi
$ tofu plan -out=tfplan

# 2. Terapkan TEPAT rencana yang ada di dalam file tersebut tanpa rekalkulasi ulang!
$ tofu apply tfplan
```

Dengan alur ini, OpenTofu dijamin **hanya akan mengeksekusi apa yang tercatat di dalam file `tfplan`**. Jika ada perubahan infrastruktur tak terduga saat plan dieksekusi, OpenTofu akan otomatis membatalkan eksekusi demi keamanan (*fail-safe*).

---

## 6. Tahap 4: `tofu apply` (Penerapan Perubahan Nyata)

Setelah rencana ditinjau dan disetujui, terapkan perubahan ke infrastruktur:

```bash
$ tofu apply tfplan
docker_image.nginx_image: Creating...
docker_image.nginx_image: Creation complete after 3s [id=sha256:...]
docker_container.web_app: Creating...
docker_container.web_app: Creation complete after 1s [id=7a8b9c0d...]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

application_url = "http://127.0.0.1:18080"
container_id = "7a8b9c0d1e2f3a4b..."
```

> [!WARNING]
> **Larangan Penggunaan Flag `-auto-approve` di Produksi:**
> Perintah `tofu apply -auto-approve` melewati tahap konfirmasi manusia dan langsung mengeksekusi perubahan. Perintah ini **DILARANG KERAS** digunakan di terminal manual laptop Anda! Satu kesalahan ketik dapat memusnahkan database produksi tanpa peringatan. Flag ini hanya boleh digunakan di dalam pipeline CI/CD yang telah memiliki proteksi *approval gate*.

---

## 7. Tahap 5: Pasca-Apply (Inspeksi & Smoke Testing)

Sebagai SRE, tugas Anda belum selesai saat `tofu apply` menampilkan kata *complete*. Anda harus memvalidasi kesehatan runtime aplikasi secara independen:

```bash
# 1. Memeriksa daftar resource yang resmi tercatat di dalam state
$ tofu state list
docker_container.web_app
docker_image.nginx_image

# 2. Memeriksa detail atribut kontainer yang baru lahir
$ tofu state show docker_container.web_app
# docker_container.web_app:
resource "docker_container" "web_app" {
    id                   = "7a8b9c0d1e2f..."
    image                = "sha256:d6b69..."
    name                 = "sre-lab-web"
    ports {
        external = 18080
        internal = 80
        ip       = "0.0.0.0"
        protocol = "tcp"
    }
}

# 3. Validasi fisik di runtime Docker host (OrbStack Mac)
$ docker ps --filter "name=sre-lab-web"
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                   NAMES
7a8b9c0d1e2f   nginx:alpine   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:18080->80/tcp   sre-lab-web

# 4. Validasi respon HTTP aplikasi (Smoke Test Layer 7)
$ curl -sI http://127.0.0.1:18080
HTTP/1.1 200 OK
Server: nginx/1.27.4
Date: Sat, 17 Oct 2026 07:15:20 GMT
Content-Type: text/html
Content-Length: 615
```

---

## 8. Tahap 6: `tofu destroy` yang Terukur & Aman

Ketika sebuah lingkungan lab atau staging sementara sudah selesai digunakan, kita wajib membersihkan sumber daya tersebut agar tidak membuang kuota komputasi dan memori:

```bash
# LANGKAH AMAN: Selalu buat plan penghancuran terlebih dahulu!
$ tofu plan -destroy -out=destroy.tfplan
Plan: 0 to add, 0 to change, 2 to destroy.

# Terapkan penghancuran setelah diverifikasi
$ tofu apply destroy.tfplan
docker_container.web_app: Destroying... [id=7a8b9c0d...]
docker_container.web_app: Destruction complete after 1s
docker_image.nginx_image: Destruction complete after 0s

Destroy complete! Resources: 2 destroyed.
```

### Checklist Keamanan Sebelum Menjalankan Destroy:
- [ ] Konteks direktori dan branch Git sudah benar (Pastikan Anda TIDAK berada di direktori `production`!).
- [ ] Backend state yang aktif diverifikasi menunjuk ke environment lab non-kritis.
- [ ] Seluruh data penting yang ada di volume telah dibackup ke lokasi terpisah.
- [ ] Tidak ada aplikasi lain yang bergantung pada network atau resource yang akan dimusnahkan.

---

## 9. Matriks Pemecahan Masalah (Troubleshooting) Workflow

| Gejala Masalah | Titik Investigasi | Penyebab Akar (*Root Cause*) | Langkah Solusi SRE |
|---|---|---|---|
| `Error: Backend initialization required` | `tofu plan` | Anda baru saja menambahkan blok backend atau mengganti provider, tetapi belum menjalankan inisialisasi. | Jalankan `tofu init` di direktori proyek tersebut. |
| `Error: Failed to lock state (Lock Info: ...)` | `tofu apply` | Ada proses OpenTofu lain yang sedang berjalan bersamaan, atau proses apply sebelumnya terputus paksa. | Periksa apakah ada engineer lain yang sedang bekerja; jika proses lama sudah mati, gunakan `tofu force-unlock <LOCK-ID>`. |
| `Apply gagal di tengah jalan (Partial Apply)` | Terminal error saat apply | Sebagian resource berhasil dibuat (misal image berhasil), namun kontainer gagal karena port bentrok. | Jangan hapus state! Perbaiki masalah port di HCL, lalu jalankan `tofu apply` ulang untuk melanjutkan rekonsiliasi. |
| `tofu destroy` mencoba menghapus objek yang tidak diinginkan | `tofu plan -destroy` | Ada referensi resource di HCL yang tidak sengaja mencakup modul shared atau resource bersama. | Hentikan proses! Periksa kembali batasan deklarasi resource dan file state. |

---

## 10. Ringkasan Praktis & Cheatsheet Workflow

```bash
# 1. Alur standar harian pengembangan IaC
tofu fmt                          # 1. Rapikan format
tofu validate                     # 2. Cek sintaks
tofu plan -out=tfplan             # 3. Hitung selisih perubahan & simpan plan
tofu apply tfplan                 # 4. Terapkan rencana yang sudah disetujui

# 2. Inspeksi cepat pasca apply
tofu output                       # Tampilkan data output
tofu state list                   # Tampilkan daftar resource yang dikelola

# 3. Pembongkaran aman
tofu plan -destroy -out=del.tfplan
tofu apply del.tfplan
```

---

## 11. Cek Pemahaman Mandiri

1. Mengapa seorang SRE sangat disarankan menyimpan hasil rencana ke dalam file binary (`tofu plan -out=tfplan`) sebelum mengeksekusi `tofu apply`, dibandingkan langsung menjalankan `tofu apply` tanpa argumen?
2. Jelaskan perbedaan dampak operasional antara simbol perubahan `~ update in-place` dengan simbol `-/+ replace` pada output `tofu plan`! Mengapa simbol `-/+` harus dicermati secara kritis?
3. Apa saja yang dilakukan oleh OpenTofu saat perintah `tofu init` dijalankan di sebuah direktori baru? Mengapa direktori `.terraform/` tidak boleh di-commit ke repositori Git?
4. Apa yang dimaksud dengan *Partial Apply Failure*? Jika dari 5 resource hanya 3 yang berhasil dibuat dan 2 gagal, apa yang terjadi pada file `terraform.tfstate`?
5. Mengapa parameter `-auto-approve` dianggap sebagai anti-pattern jika dieksekusi secara manual oleh operator di terminal lingkungan produksi?

---

*Langkah Selanjutnya:* Masuk ke [04 — State, Remote, Import, Drift](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.1-dasar-opentofu/04-state-remote-import-drift.md) untuk mempelajari manajemen state tingkat lanjut, konfigurasi remote backend MinIO S3, mekanisme locking, teknik adopsi infrastruktur via `tofu import`, dan penanganan *configuration drift*.
