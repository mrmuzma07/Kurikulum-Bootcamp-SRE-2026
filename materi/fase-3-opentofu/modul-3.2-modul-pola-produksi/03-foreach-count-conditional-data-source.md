# 03 — Pola Instansiasi: `for_each`, `count`, Conditional, dan Data Source

> **Ringkasan:** Membuat banyak sumber daya infrastruktur yang serupa (*multi-instance resources*) adalah kebutuhan umum dalam perancangan arsitektur modern. Namun, salah memilih antara konstruksi `count` dan `for_each` dapat berakibat fatal: seluruh server produksi Anda dapat terhapus dan dibangun ulang tanpa sengaja akibat fenomena pergeseran indeks (*Index Churn*). Dokumen ini membedah mekanisme penentuan alamat resource di dalam state, menyusun logika kondisional bersyarat (*feature toggle*) yang aman, memahami batas kepemilikan data read-only (*Data Source*), hingga menulis validasi koleksi data tingkat lanjut.

---

## 1. Analogi Pemula: Nomor Antrean Kursi Bioskop vs Tiket Bernama KTP

Untuk memahami perbedaan paling mendasar antara `count` dan `for_each`, bayangkan dua sistem reservasi tiket:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ANALOGI SISTEM TIKET: COUNT VS FOR_EACH                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ METODE COUNT: NOMOR URUT KURSI BIOSKOP (ARRAY INDEX [0], [1], [2]) ]     │
│  Tiga orang sahabat memesan kursi di bioskop:                               │
│  - Kursi [0] diduduki oleh Ali.                                             │
│  - Kursi [1] diduduki oleh Budi.                                            │
│  - Kursi [2] diduduki oleh Citra.                                           │
│                                                                             │
│  Tiba-tiba Ali sakit perut dan membatalkan tiketnya (Ali dihapus).          │
│  Apa yang terjadi pada sistem antrean nomor kursi?                          │
│  - Petugas bioskop menyuruh Budi PINDAH DUDUK ke Kursi [0].                 │
│  - Petugas menyuruh Citra PINDAH DUDUK ke Kursi [1].                        │
│  - Kursi [2] dihancurkan/dihapus.                                           │
│                                                                             │
│  BENCANA DI OPENTOFU: Karena alamat Budi dan Citra berubah indeks,         │
│  OpenTofu akan MENGHANCURKAN server Budi dan Citra lalu membangunnya ulang  │
│  dari awal (**INDEX CHURN / DESTROY AND RECREATE**) hanya karena Ali pergi! │
│                                                                             │
│  [ METODE FOR_EACH: TIKET RESERVASI BERNAMA IDENTITAS (MAP KEYS) ]          │
│  Setiap tiket dicetak dengan Nama KTP pemesan yang permanen:                │
│  - Kursi["ali"] diduduki oleh Ali.                                          │
│  - Kursi["budi"] diduduki oleh Budi.                                        │
│  - Kursi["citra"] diduduki oleh Citra.                                      │
│                                                                             │
│  Ketika Ali membatalkan tiketnya:                                           │
│  - Hanya Kursi["ali"] yang dihapus secara terisolasi.                       │
│  - Budi dan Citra tetap duduk tenang di kursinya tanpa terganggu sedikit pun│
│    karena alamat identitas mereka TIDAK PERNAH BERUBAH (**ZERO DOWNTIME**)! │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Bedah Teknis `count`: Karakteristik & Bahaya "Index Churn"

`count` adalah metode perulangan paling sederhana di OpenTofu. `count` menerima nilai integer dan menghasilkan alamat resource berbasis indeks array angka: `[0]`, `[1]`, `[2]`, dan seterusnya.

### Contoh Implementasi `count`:
```hcl
variable "server_names" {
  type    = list(string)
  default = ["frontend", "backend", "database"]
}

resource "docker_container" "app" {
  count = length(var.server_names)

  name  = "sre-${var.server_names[count.index]}"
  image = "nginx:1.27.4-alpine"
}
```

Alamat resource yang dicatat di dalam file `terraform.tfstate`:
- `docker_container.app[0]` ──► memetakan ke kontainer `"sre-frontend"`
- `docker_container.app[1]` ──► memetakan ke kontainer `"sre-backend"`
- `docker_container.app[2]` ──► memetakan ke kontainer `"sre-database"`

---

### Bencana Kasus Nyata: Jebakan Index Churn (The Shift-and-Destroy Trap)
Suatu hari, Anda memutuskan untuk menghapus komponen `"frontend"` dari list variabel, sehingga variabel Anda menjadi:
```hcl
variable "server_names" {
  type    = list(string)
  default = ["backend", "database"] # "frontend" dihapus dari indeks 0!
}
```

Ketika Anda menjalankan `tofu plan`, inilah bencana yang akan terjadi:

```text
OpenTofu will perform the following actions:

  # docker_container.app[0] must be replaced (-/+)
  ~ name: "sre-frontend" => "sre-backend" (forces new resource!)

  # docker_container.app[1] must be replaced (-/+)
  ~ name: "sre-backend" => "sre-database" (forces new resource!)

  # docker_container.app[2] will be destroyed (-)
  - resource "docker_container" "app" {}

Plan: 2 to add, 0 to change, 3 to destroy.
```

> [!CAUTION]
> **Mengapa Ini Terjadi?**
> Karena indeks array bergeser! OpenTofu melihat bahwa `app[0]` yang tadinya bernama `frontend` kini diubah namanya menjadi `backend`. Karena nama kontainer bersifat `ForceNew`, OpenTofu **MENGHANCURKAN kontainer backend yang sedang melayani transaksi live**, menghancurkan database, dan mematikan seluruh sistem produksi Anda!

### Kapan `count` Boleh Digunakan Secara Aman?
Gunakan `count` **HANYA** untuk dua kondisi berikut:
1. **Kumpulan Server yang 100% Homogen:** Kumpulan instans worker tanpa nama khusus dan tanpa IP tetap (misalnya: `count = 5` untuk membuat 5 worker pool identik di mana penambahan atau pengurangan hanya terjadi di elemen paling akhir).
2. **Kondisional Boolean On/Off (`0` atau `1`):** Untuk mengaktifkan atau menonaktifkan satu resource tunggal (`count = var.enable_feature ? 1 : 0`).

---

## 3. `for_each`: Standar Emas Instansiasi Multi-Resource

Untuk seluruh resource yang memiliki nama, port, subnet, atau fungsi bisnis yang unik, **`for_each` adalah standar wajib industri SRE**.

`for_each` menerima input berupa struktur data **`map`** atau **`set of strings`**. Alamat resource di dalam state tidak lagi menggunakan angka indeks, melainkan menggunakan kunci (*keys*) bisnis yang stabil.

### Contoh Implementasi `for_each` Standar Enterprise:

```hcl
variable "web_services" {
  description = "Daftar layanan web dengan spesifikasi unik per komponen."
  type = map(object({
    container_name = string
    host_port      = number
    image_tag      = string
  }))
  default = {
    frontend = {
      container_name = "sre-prod-ui"
      host_port      = 18080
      image_tag      = "1.27.4-alpine"
    }
    api = {
      container_name = "sre-prod-api"
      host_port      = 18081
      image_tag      = "1.27.4-alpine"
    }
    metrics = {
      container_name = "sre-prod-metrics"
      host_port      = 18082
      image_tag      = "1.27.4-alpine"
    }
  }
}

# Memanggil modul menggunakan for_each
module "web_cluster" {
  source   = "../../modules/web-server"
  for_each = var.web_services

  # each.key   = "frontend", "api", "metrics"
  # each.value = objek data di dalam map
  container_name = each.value.container_name
  host_port      = each.value.host_port
  image_tag      = each.value.image_tag
  environment    = "prod"
}
```

Alamat resource yang dicatat di dalam state:
- `module.web_cluster["frontend"].docker_container.this`
- `module.web_cluster["api"].docker_container.this`
- `module.web_cluster["metrics"].docker_container.this`

### Apa yang Terjadi Jika Layanan `frontend` Dihapus?
Jika Anda menghapus key `"frontend"` dari map variabel, OpenTofu akan membaca diff secara terisolasi:

```text
Plan: 0 to add, 0 to change, 1 to destroy.

- module.web_cluster["frontend"].docker_container.this will be destroyed
```
Layanan `api` dan `metrics` **sama sekali tidak tersentuh dan tidak mengalami restart sedikit pun**! Inilah keunggulan stabilitas mutlak dari `for_each`.

---

## 4. Pola Logika Kondisional (Conditional Resources)

Di dalam HCL, kita sering kali perlu menyalakan atau mematikan resource tertentu berdasarkan lingkungan (misalnya: membuat kontainer monitoring hanya di lingkungan `staging` dan `prod`, tetapi dinonaktifkan di `dev` untuk menghemat RAM laptop).

OpenTofu menggunakan **Ternary Operator**:
`kondisi ? nilai_jika_true : nilai_jika_false`

### 1. Pola Kondisional Menggunakan `count` (Toggle On/Off Sederhana)
```hcl
variable "enable_monitoring_agent" {
  description = "Saklar untuk menyalakan agen pemantau metrik."
  type        = bool
  default     = true
}

resource "docker_container" "prometheus_node_exporter" {
  # Jika true buat 1 instans; jika false buat 0 instans (tidak dibuat sama sekali!)
  count = var.enable_monitoring_agent ? 1 : 0

  name  = "sre-node-exporter"
  image = "prom/node-exporter:v1.8.2"

  ports {
    internal = 9100
    external = 19100
  }
}
```

### 2. Cara Aman Menangkap Output dari Resource Kondisional
Sering kali pemula menulis output seperti ini:
```hcl
# CONTOH SALAH (AKAN CRASH JIKA MONITORING MATI!):
output "exporter_ip" {
  value = docker_container.prometheus_node_exporter[0].network_data[0].ip_address
}
```
Jika `var.enable_monitoring_agent = false`, resource tersebut memiliki jumlah `count = 0`. Mengakses indeks `[0]` akan langsung melempar pesan fatal: **`Error: Invalid index: index 0 out of range for empty tuple`**!

**Solusi SRE yang Benar:** Gunakan operator ternary atau fungsi `try()`:

```hcl
# SOLUSI BENAR & AMAN:
output "exporter_ip" {
  description = "IP address monitoring agent jika diaktifkan."
  value       = var.enable_monitoring_agent ? docker_container.prometheus_node_exporter[0].network_data[0].ip_address : "DISABLED"
}

# Atau menggunakan sintaks 'splat' [*] dan one():
output "exporter_id" {
  description = "ID kontainer monitoring (null jika nonaktif)."
  value       = one(docker_container.prometheus_node_exporter[*].id)
}
```

---

## 5. Data Source vs Resource: Memahami Batasan Kepemilikan (Ownership)

Salah satu konsep yang wajib dipahami di modul produksi adalah membedakan kapan kita harus menggunakan blok **`resource`** dan kapan harus menggunakan blok **`data`**:

```text
┌─────────────────────────────────┬─────────────────────────────────┐
│ BLOK RESOURCE                   │ BLOK DATA (DATA SOURCE)         │
├─────────────────────────────────┼─────────────────────────────────┤
│ **Kepemilikan Penuh (CRUD):**   │ **Hanya Membaca (Read-Only):**  │
│ OpenTofu bertanggung jawab      │ OpenTofu HANYA membaca objek    │
│ membuat fisik, mengubah, dan    │ yang SUDAH DIBUAT DULUAN oleh   │
│ menghancurkan objek tersebut.   │ tim/sistem lain di luar proyek. │
├─────────────────────────────────┼─────────────────────────────────┤
│ Mengubah siklus hidup di server │ **Nol Efek Samping Fisik:**     │
│ nyata saat di-apply/destroy.    │ Tidak akan pernah membuat atau  │
│                                 │ menghapus objek fisik di server.│
├─────────────────────────────────┼─────────────────────────────────┤
│ Contoh: Membuat kontainer baru, │ Contoh: Membaca ID VPC jaringan │
│ membuat subnet bridge baru.     │ bersama yang dikelola tim NetOps│
└─────────────────────────────────┴─────────────────────────────────┘
```

### Contoh Praktik Penggunaan Data Source:
Bayangkan tim infrastruktur pusat (*Platform Team*) sudah membuatkan jaringan Docker bersama bernama `shared-core-network`. Proyek aplikasi Anda tidak boleh menghapus network tersebut, melainkan hanya perlu menyambungkan kontainer Anda ke sana:

```hcl
# 1. BACA JARINGAN YANG SUDAH ADA DI SERVER VIA DATA SOURCE:
data "docker_network" "existing_platform_net" {
  name = "shared-core-network"
}

# 2. SAMBUNGKAN KONTAINER KITA MENGGUNAKAN ID DARI DATA SOURCE TERSEBUT:
resource "docker_container" "app_service" {
  name  = "sre-custom-api"
  image = "nginx:1.27.4-alpine"

  networks_advanced {
    name = data.docker_network.existing_platform_net.name
  }
}
```

> [!WARNING]
> **Anti-Pattern Data Source:**
> Jangan gunakan `data` source untuk membaca resource yang sebenarnya dideklarasikan di dalam file proyek HCL Anda sendiri. Itu adalah tanda bahwa arsitektur modular Anda mengalami kekacauan referensi. Gunakan referensi ekspresi langsung (`docker_network.my_net.id`).

---

## 6. Validasi Koleksi Kompleks Menggunakan HCL Expressions

Untuk menjamin bahwa data input dari tim developer tidak menyebabkan konflik di server, kita dapat menggabungkan fungsi-fungsi HCL tingkat lanjut (`alltrue`, `can`, `regex`, `keys`, `values`) di dalam blok `validation {}`:

### Contoh Validasi: Memastikan Seluruh Port Berada di Rentang Non-Root
```hcl
variable "assigned_ports" {
  type        = list(number)
  description = "Daftar port yang ingin dibuka untuk layanan kontainer."
  default     = [18080, 18081, 18082]

  validation {
    # Memeriksa apakah SEMUA angka di dalam list berada di rentang 1024 s.d. 65535
    condition     = alltrue([for p in var.assigned_ports : p >= 1024 && p <= 65535])
    error_message = "Seluruh port yang didaftarkan wajib berada di rentang port aman (1024-65535)."
  }

  validation {
    # Memeriksa apakah tidak ada duplikasi nomor port di dalam list yang sama!
    condition     = length(var.assigned_ports) == length(distinct(var.assigned_ports))
    error_message = "DILARANG mendaftarkan nomor port yang sama berulang kali (terdeteksi duplikasi port)!"
  }
}
```

---

## 7. Pola Kegagalan (Failure Modes) & Panduan Diagnosa

| Pesan Kesalahan OpenTofu | Penyebab Akar (*Root Cause*) | Solusi Pemulihan SRE |
|---|---|---|
| `Error: Invalid for_each argument` | Kunci map pada `for_each` bergantung pada nilai dinamis yang baru diketahui saat apply (*dynamic unknown values*). | Kunci (*keys*) pada `for_each` WAJIB berupa string statis yang sudah pasti diketahui sebelum eksekusi dimulai. |
| `Error: Invalid index: index 0 out of range` | Anda mencoba membaca output dari resource kondisional yang disetel `count = 0` (nonaktif). | Gunakan operator ternary `cond ? res[0].val : null` atau fungsi `one(res[*].val)`. |
| `Error: data.docker_network.existing: not found` | Objek eksternal yang ingin dibaca oleh Data Source belum pernah dibuat di server fisik. | Pastikan nama network/objek sudah benar atau koordinasikan dengan tim penyedia infrastruktur. |
| `Seluruh server ter-replace tak terduga` | Anda menggunakan `count` dan baru saja menghapus elemen di awal/tengah list variabel. | Segera batalkan apply (`Ctrl+C`), dan migrasikan struktur kode dari `count` ke `for_each`. |

---

## 8. Ringkasan Praktis & Cheatsheet Instansiasi

```bash
# 1. Menampilkan daftar resource yang dibuat menggunakan for_each
tofu state list | grep 'module.web_cluster'

# 2. Menginspeksi satu instans spesifik di dalam map for_each
tofu state show 'module.web_cluster["frontend"].docker_container.this'

# 3. Menghapus satu instans terisolasi tanpa mengganggu instans lain di map
tofu state rm 'module.web_cluster["frontend"].docker_container.this'

# 4. Memverifikasi rencana tanpa mengeksekusi mutasi
tofu plan
```

---

## 9. Cek Pemahaman Mandiri

1. Jelaskan fenomena *Index Churn (The Shift-and-Destroy Trap)* pada penggunaan konstruksi `count`! Mengapa menghapus satu item di awal list dapat memicu kehancuran seluruh server berikutnya?
2. Kapan sebuah resource sebaiknya diinstansiasi menggunakan `for_each` dibandingkan menggunakan `count`? Berikan contoh kasus nyata di lingkungan produksi!
3. Bagaimana cara yang aman untuk mengambil atribut output dari sebuah resource yang menggunakan kondisional `count = var.enabled ? 1 : 0` tanpa memicu error *Invalid index 0 out of range* saat kondisinya bernilai `false`?
4. Jelaskan perbedaan mendasar antara blok `resource` dengan blok `data` (Data Source) dari segi batasan kepemilikan (*ownership boundary*) dan efek samping terhadap siklus hidup objek fisik di server!
5. Mengapa kunci (*keys*) pada deklarasi `for_each` wajib berupa nilai statis yang sudah diketahui sebelum tahap `tofu apply` dimulai?

---

*Langkah Selanjutnya:* Masuk ke [04 — Secret Handling dan CI Plan](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-3-opentofu/modul-3.2-modul-pola-produksi/04-secret-handling-dan-ci-plan.md) untuk mempelajari bagaimana melindungi data rahasia (*secrets*) agar tidak bocor ke dalam repositori Git atau state publik, merancang pipeline CI/CD yang terisolasi, dan menyerahkan metadata aman ke Ansible.
