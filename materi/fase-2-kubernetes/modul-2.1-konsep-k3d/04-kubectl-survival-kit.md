# 04 — kubectl Survival Kit: Perintah Harian, Format Output & Alur Debug SRE

> **Ringkasan:** Menguasai pisau lipat serbaguna seorang SRE di terminal: mahir menggunakan empat serangkai perintah inti (**`get`**, **`describe`**, **`logs`**, **`exec`**), memfilter data dengan **Label Selectors** dan **JSONPath**, menguasai alur sistematis investigasi insiden (**5-Step Debug Flow**), membedah arti **Exit Code** kematian container, serta trik tingkat lanjut **Ephemeral Debug Container** pada image tanpa shell (*Distroless*).

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan mempraktikkannya di terminal MacBook, kamu akan mampu:
- Menavigasi dan menginvestigasi objek Kubernetes dengan cepat menggunakan perintah **`kubectl`** tanpa bergantung pada antarmuka grafis (GUI).
- Memilih format tampilan data yang tepat (**`-o wide`**, **`-o yaml`**, **`-o jsonpath`**, dan **`-o name`**) untuk kebutuhan inspeksi manual maupun skrip otomasi.
- Menjalankan prosedur investigasi standar insiden SRE (**5-Step Debug Flow**) saat menemukan Pod yang bermasalah.
- Membaca dan menafsirkan arti dari riwayat **Events** dan pesan kematian aplikasi menggunakan **`kubectl logs --previous`**.
- Mendiagnosa penyebab status abnormal pada Pod (**`Pending`**, **`ImagePullBackOff`**, **`CrashLoopBackOff`**, dan **`OOMKilled`** / Exit Code 137).
- Melakukan investigasi interaktif pada container *Distroless* menggunakan fitur canggih **`kubectl debug`**.
- Mengonfigurasi alias dan *shell autocompletion* di terminal untuk melipatgandakan kecepatan kerja harian (*muscle memory*).

---

## 1. Mindset SRE: Mengapa Terminal Adalah Rumah Utama Kita?

Di lingkungan cloud publik atau tutorial pemula, sering kali orang tergoda menggunakan dashboard grafis web (seperti Kubernetes Dashboard atau k9s).
Namun, di dunia **Site Reliability Engineering (SRE) Production On-Premises**:
1. Server di data center terisolasi di balik jaringan privat ketat (*air-gapped*) yang **tidak mengizinkan pembukaan port antarmuka web** demi keamanan.
2. Ketika terjadi insiden kebakaran sistem (*outage*), dashboard grafis sering kali ikut tumbang atau lambat memuat data karena API server sedang kebanjiran beban.
3. Kecepatan seorang SRE dalam memulihkan sistem dihitung dalam menit dan detik. Menggunakan perintah terminal **`kubectl`** yang sudah melekat di jari (*muscle memory*) adalah cara tercepat untuk menyelamatkan operasional perusahaan!

---

## 2. Empat Serangkai Perintah Inti: Get, Describe, Logs, dan Exec

Hampir 90% waktu investigasimu di Kubernetes akan dihabiskan bersama empat perintah berikut:

```text
 ┌─────────────────────────────────────────────────────────────────────────┐
 │               EMPAT PILAR INVESTIGASI HARIAN KUBECTL                    │
 ├───────────────────┬─────────────────────────────────────────────────────┤
 │ 1. kubectl get    │ "Ada apa saja di sini dan bagaimana status umumnya?"│
 │ 2. kubectl describe│ "Ceritakan riwayat hidup & masalah apa yang terjadi"│
 │ 3. kubectl logs   │ "Apa kata aplikasi di dalam layar terminalnya?"     │
 │ 4. kubectl exec   │ "Izinkan saya masuk ke dalam untuk memeriksa!"      │
 └───────────────────┴─────────────────────────────────────────────────────┘
```

---

### Pilar 1: `kubectl get` — Memeriksa Kondisi Umum Objek
Perintah `get` digunakan untuk menampilkan daftar satu atau banyak objek dalam format tabel ringkas:

```bash
# Melihat Pod di namespace aktif
kubectl get pods

# Melihat Pod di SELURUH namespace yang ada di cluster (-A = --all-namespaces)
kubectl get pods -A

# Melihat objek gabungan sekaligus (Pod, Service, Deployment):
kubectl get pods,svc,deploy

# Menampilkan seluruh objek utama yang ada di namespace default:
kubectl get all
```

---

### Pilar 2: `kubectl describe` — Membaca Rekam Medis Lengkap Objek
Jika `get` hanya memberikan gambaran sekilas (misal: status `CrashLoopBackOff`), maka **`describe`** adalah dokter spesialis yang membuka **Buku Rekam Medis Lengkap** dari objek tersebut:

```bash
kubectl describe pod <nama-pod>
kubectl describe node <nama-node>
```

> 🔍 **Bagian Paling Berharga di Output Describe: `Events`!**
> Saat menjalankan `kubectl describe pod`, **jangan berhenti di baris atas! Gulir layar terminalmu ke bagian paling bawah (bagian `Events`)**.
> Di sanalah Kubernetes mencatat buku harian kronologis kejadian:
> - Kapan Pod dijadwalkan oleh scheduler?
> - Apakah proses penarikan image dari registry gagal (*Failed to pull image*)?
> - Apakah container ditolak karena memori server penuh?
> - Berapa kali container mati dan menyala ulang?
> **90% akar masalah kerusakan Pod akan langsung terungkap di bagian Events ini!**

---

### Pilar 3: `kubectl logs` — Membaca Cetakan Log Aplikasi
Jika bagian `Events` di `describe` tidak menunjukkan kesalahan infrastruktur, artinya masalahnya ada di dalam **kodingan aplikasi itu sendiri** (misalnya: gagal membaca password database atau sintaks error saat parsing file JSON).

```bash
# Membaca log output (stdout/stderr) dari sebuah Pod:
kubectl logs <nama-pod>

# Memantau aliran log secara langsung/mengalir (-f = follow / live streaming):
kubectl logs -f <nama-pod>

# Membaca 50 baris terakhir saja (--tail):
kubectl logs --tail=50 <nama-pod>

# Membaca log dari seluruh Pod di dalam sebuah Deployment sekaligus:
kubectl logs -f deployment/<nama-deployment>

# Jika Pod memiliki LEBIH DARI 1 container (Multi-Container/Sidecar), sebutkan nama containernya:
kubectl logs <nama-pod> -c <nama-container>
```

> 💀 **Senjata Rahasia SRE: Bendera `--previous`!**
> Bayangkan skenario ini: Aplikasi di dalam Pod mengalami crash, mati, lalu otomatis dinyalakan ulang oleh kubelet.
> Saat kamu mengetik `kubectl logs <nama-pod>`, yang tampil adalah log dari **container baru yang baru saja menyala**, sedangkan pesan error penyebab kematian container sebelumnya sudah hilang!
> Gunakan bendera **`--previous`** untuk membaca log dari **container yang mati sesaat sebelum restart**:
> ```bash
> kubectl logs <nama-pod> --previous
> ```

---

### Pilar 4: `kubectl exec` — Masuk ke Dalam Lingkungan Container
Terkadang membaca log saja belum cukup. Kita perlu masuk ke dalam sistem file container untuk mengecek apakah file konfigurasi benar-benar terpasang, menguji koneksi database menggunakan `nc` atau `ping`, atau mengintip variabel lingkungan:

```bash
# Masuk secara interaktif (-it) ke dalam shell container:
kubectl exec -it <nama-pod> -- sh
# (atau '/bin/bash' jika image berbasis Ubuntu/Debian)

# Menjalankan satu perintah langsung dari luar TANPA masuk shell interaktif:
kubectl exec <nama-pod> -- printenv
kubectl exec <nama-pod> -- cat /etc/config/app.conf
```

---

### Bonus Jalan Tol Sementara: `kubectl port-forward`
Bagaimana cara mengetes layanan database atau backend yang bertipe `ClusterIP` (yang tidak punya IP publik) langsung dari laptop Mac milikmu tanpa perlu memasang Ingress?
Gunakan **`kubectl port-forward`**:

```bash
# Buka jalan tol dari port 9090 di Mac langsung menuju port 80 milik Service:
kubectl port-forward svc/<nama-service> 9090:80

# Atau langsung tembak ke satu Pod spesifik:
kubectl port-forward pod/<nama-pod> 8080:8080
```
Selama jendela terminal tersebut terbuka, kamu bisa membuka `http://localhost:9090` di browser Mac-mu! Tekan `Ctrl + C` untuk menutup jalan tol tersebut.

---

## 3. Seni Memilih Format Tampilan Data (`-o`)

Sebagai seorang SRE, kamu sering kali butuh mengekstrak data spesifik dari ratusan Pod untuk dimasukkan ke dalam skrip bash atau laporan insiden.
Gunakan bendera **`-o` (output format)**:

| Format Bendera | Tampilan yang Dihasilkan | Kapan Digunakan? |
|---|---|---|
| *(Default tanpa `-o`)* | Tabel ringkas standar | Pemeriksaan cepat sehari-hari |
| **`-o wide`** | Tabel lebar dengan kolom ekstra (**IP Pod, Node tempat Pod tinggal**, dan arsitektur) | Memeriksa apakah Pod tersebar merata di antar-node |
| **`-o yaml`** | Seluruh spesifikasi teknis dan status dalam format YAML | Menyelidiki konfigurasi detail atau menyalin manifes objek |
| **`-o name`** | Hanya mencetak jenis dan nama objek (misal: `pod/web-123`) | Sangat berguna untuk looping di dalam bash script |
| **`-o jsonpath=...`** | Mengekstrak isi data spesifik menggunakan kueri path JSON | Mengambil alamat IP atau status Pod secara presisi |

### Contoh Penggunaan JSONPath Praktis:
```bash
# 1. Ambil HANYA nama Pod pertama di namespace default:
kubectl get pods -o jsonpath='{.items[0].metadata.name}'

# 2. Ambil seluruh Alamat IP internal milik semua node worker:
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# 3. Cetak daftar nama Pod beserta status phase-nya dipisahkan tab:
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

---

## 4. Seni Memfilter: Labels, Selectors, dan Namespaces

Di cluster yang memiliki 1.000 Pod, menjalankan `kubectl get pods` biasa akan menghasilkan dinding teks yang membuat pusing. Kita harus menyaringnya menggunakan **Label Selector**:

```bash
# 1. Filter persamaan tunggal (Ambil Pod yang berlabel app=portal):
kubectl get pods -l app=portal

# 2. Filter kombinasi DAN (app=portal DAN env=production):
kubectl get pods -l app=portal,env=production

# 3. Filter himpunan (Ambil Pod yang perannya frontend ATAU backend):
kubectl get pods -l 'tier in (frontend, backend)'

# 4. Tampilkan semua Pod beserta kolom label yang mereka miliki:
kubectl get pods --show-labels
```

### Mengganti Namespace Default pada Terminal
Bosan mengetik `-n production` di setiap akhir perintah?
Ubah namespace aktif pada context Kubeconfig komputermu dengan perintah ini:
```bash
kubectl config set-context --current --namespace=production
# Sekarang setiap perintah 'kubectl get pods' otomatis melihat namespace production!
```

---

## 5. Prosedur Baku SRE: 5-Step Debug Flow Saat Pod Bermasalah

Ketika alarm sistem berbunyi atau sebuah Pod gagal menyala, **JANGAN MENEBAK-NEBAK!**
Gunakan protokol investigasi 5 langkah terstruktur berikut:

```text
 ┌─────────────────────────────────────────────────────────────────────────┐
 │               PROTOKOL INVESTIGASI 5 LANGKAH SRE (DEBUG FLOW)           │
 └────────────────────────────────────┬────────────────────────────────────┘
                                      │
  LANGKAH 1: IDENTIFIKASI             ▼
  kubectl get pods -o wide     ───► Apa statusnya? (Pending / CrashLoop / OOM?)
                                      │
  LANGKAH 2: REKAM MEDIS & EVENT      ▼
  kubectl describe pod <nama>  ───► Baca bagian 'Events' di paling bawah!
                                    (Apakah node penuh? Image salah?)
                                      │
  LANGKAH 3: LOG RUNTIME INTERNAL     ▼
  kubectl logs <nama>          ───► Baca output error aplikasi.
  kubectl logs <nama> --previous    (Jika container mati-hidup, baca log sebelumnya!)
                                      │
  LANGKAH 4: INSPEKSI LANGSUNG        ▼
  kubectl exec -it <nama> -- sh───► Periksa file config, disk, atau DNS database.
  (atau kubectl debug)
                                      │
  LANGKAH 5: PERBAIKAN & VALIDASI     ▼
  kubectl apply -f manifest.yaml ─► Terapkan perbaikan, pantau 'kubectl rollout status'
```

---

## 6. Kamus Status Abnormal Pod & Exit Code Kematian Container

Berikut adalah tabel contekan cepat untuk mengenali arti dari status-status error yang paling sering muncul di terminal:

| Status Pod di Terminal | Apa yang Sebenarnya Terjadi? | Penyebab Paling Sering & Solusi |
|---|---|---|
| **`Pending`** | Pod sudah dicatat di etcd, namun belum mulai berjalan di node mana pun. | **Scheduler gagal menemukan node yang cukup:** Seluruh node kehabisan jatah CPU/RAM (`Insufficient memory`), atau PVC penyimpanan belum berhasil terpasang. |
| **`ImagePullBackOff`** / `ErrImagePull` | Container gagal menyala karena imagenya tidak bisa diunduh. | **Salah ketik nama image/tag**, repositori privat belum diberi `imagePullSecret`, atau image belum di-import ke k3d (`k3d image import`). |
| **`CrashLoopBackOff`** | Container berhasil diunduh dan sempat jalan, tetapi langsung mati lalu di-restart berulang kali oleh kubelet. | **Aplikasi gagal start:** File konfigurasi salah ketik, variabel lingkungan database belum disuntikkan, atau *Liveness Probe* gagal merespons. |
| **`OOMKilled`** (Exit 137) | Container dibunuh secara paksa oleh Kernel Linux menggunakan sinyal `SIGKILL`. | Aplikasi memakan RAM melebihi angka `resources.limits.memory` pada manifes Pod, atau ada kebocoran memori (*memory leak*). Solusi: Naikkan limit memori. |
| **`CreateContainerConfigError`** | Kubernetes menolak membuat container sebelum ia menyala. | Pod mereferensikan nama **ConfigMap** atau **Secret** yang belum dibuat atau salah ketik namanya. |

---

### Membedah Angka "Exit Code"
Ketika container berhenti, sistem operasi Linux selalu meninggalkan sebuah nomor kode keluar (*Exit Code*):

```text
  Exit Code = 0   ➔ Berjalan lancar sampai selesai (Sukses / Normal).
  Exit Code = 1   ➔ Aplikasi berhenti karena error internal kodingan / exception.
  Exit Code = 137 ➔ OOMKilled (128 + Sinyal 9 SIGKILL: Dibunuh paksa oleh kernel karena rakus RAM).
  Exit Code = 143 ➔ Graceful Shutdown (128 + Sinyal 15 SIGTERM: Dimatikan sopan oleh k8s saat rollout).
  Exit Code = 126 ➔ Perintah utama di container tidak memiliki izin eksekusi (chmod +x hilang).
  Exit Code = 127 ➔ Perintah utama / biner program tidak ditemukan di dalam PATH container.
```

---

## 7. Trik Tingkat Lanjut: Ephemeral Debug Container (`kubectl debug`)

Di Modul 1.1, kita belajar bahwa standar keamanan production mengharuskan kita menggunakan base image **Google Distroless** yang **tidak memiliki shell (`sh` atau `bash`)** dan tidak memiliki alat `curl`, `ls`, maupun `ping`.

Lalu, bagaimana jika sebuah Pod Distroless di production mengalami gangguan dan kita perlu mendebugnya? Jika kamu mengetik `kubectl exec -it <pod> -- sh`, terminal akan menolak dengan error: `OCI runtime exec failed: exec: "sh": executable file not found in $PATH`.

### Solusinya: `kubectl debug` (Suntikan Kontainer Dokter Sementara)
Kubernetes menyediakan fitur canggih bernama **Ephemeral Containers**:
Kita bisa menyuntikkan satu container baru yang lengkap dengan peralatan bedah (seperti Alpine Linux yang punya `sh`, `curl`, dan `netstat`) langsung ke dalam kamar Pod yang sedang sakit, lalu berbagi *Process Namespace* dengan aplikasi yang bermasalah!

```bash
# Suntikkan container dokter berbasis Alpine ke dalam Pod yang bermasalah:
kubectl debug -it <nama-pod> --image=nicolaka/netshoot --target=<nama-container-target>
```
Sekarang kamu memiliki terminal Linux lengkap dengan segala alat diagnosa jaringan untuk memeriksa aplikasi Distroless tersebut tanpa perlu mengubah atau merestart aplikasi aslinya!

---

## 8. Meningkatkan Kecepatan Tangan: Aliases & Autocompletion

Seorang engineer SRE senior tidak pernah mengetik kata `kubectl get pods` huruf demi huruf sebanyak 100 kali sehari. Mereka mengonfigurasi jalan pintas (*alias*) di dalam file profil shell (`~/.zshrc` di macOS):

### Tambahkan Baris Ini ke Dalam `~/.zshrc`:
```bash
# 1. Alias perintah dasar
alias k="kubectl"
alias kgp="kubectl get pods"
alias kgpa="kubectl get pods -A"
alias kgpw="kubectl get pods -o wide"
alias kdp="kubectl describe pod"
alias kl="kubectl logs"
alias klf="kubectl logs -f"
alias klp="kubectl logs --previous"
alias kx="kubectl exec -it"
alias kpf="kubectl port-forward"

# 2. Aktifkan autocompletion tombol TAB untuk alias 'k'
source <(kubectl completion zsh)
complete -o default -F __start_kubectl k
```

Setelah menyimpannya, jalankan `source ~/.zshrc`. Sekarang kamu cukup mengetik `k get pods` atau menekan tombol `TAB` dua kali untuk melengkapi nama Pod secara otomatis!

---

## 9. Hands-On Mandiri Step-by-Step (25 Menit)

Mari kita simulasikan 3 skenario kegagalan Pod di cluster `k3d` dan latih kemampuan diagnosamu menggunakan alur *Debug Flow*:

```bash
# --- LANGKAH 1: Pastikan Cluster k3d 'lab' Menyala ---
k3d cluster start lab 2>/dev/null || k3d cluster create lab --servers 1 --agents 2 --port 8080:80@loadbalancer

# --- LANGKAH 2: Simulasi Kasus 1 — ImagePullBackOff ---
# Buat deployment dengan nama tag yang sengaja dibuat salah:
kubectl create deployment kasus-salah-tag --image=nginx:versi-palsu-12345

# Jalankan Debug Flow Langkah 1 & 2:
kubectl get pods
# Amati status: ErrImagePull / ImagePullBackOff

kubectl describe pod -l app=kasus-salah-tag | tail -n 15
# Perhatikan bagian Events: "Failed to pull image ... manifest unknown"

# Bersihkan:
kubectl delete deployment kasus-salah-tag

# --- LANGKAH 3: Simulasi Kasus 2 — CrashLoopBackOff & Logs --previous ---
# Buat Pod yang perintahnya sengaja dibuat gagal setelah jalan 1 detik:
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kasus-crash
spec:
  containers:
  - name: gagal
    image: alpine:3.20
    command: ["sh", "-c", "echo 'Memulai inisialisasi...'; sleep 2; echo 'FATAL: File konfigurasi tidak ditemukan!' >&2; exit 1"]
EOF

# Tunggu 10 detik hingga Kubernetes mencoba me-restart container beberapa kali:
sleep 10
kubectl get pod kasus-crash
# Amati status: CrashLoopBackOff dengan jumlah Restarts bertambah!

# Jalankan Debug Flow Langkah 3 (Baca log container yang mati sebelumnya):
kubectl logs kasus-crash --previous

# Bersihkan:
kubectl delete pod kasus-crash

# --- LANGKAH 4: Simulasi Kasus 3 — OOMKilled (Exit Code 137) ---
# Buat Pod dengan batas limit RAM hanya 20MB, namun alokasikan 50MB di memori:
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kasus-oom
spec:
  containers:
  - name: rakus-ram
    image: alpine:3.20
    resources:
      limits:
        memory: "20Mi"
    command: ["sh", "-c", "dd if=/dev/zero of=/dev/shm/beban bs=1M count=50"]
EOF

sleep 5
# Periksa status kematian dan exit code:
kubectl describe pod kasus-oom | grep -E "Terminated|Exit Code|Reason"
# Output yang muncul: Reason: OOMKilled | Exit Code: 137

# Bersihkan:
kubectl delete pod kasus-oom
```

---

## 10. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas kubectl Survival Kit (Cheat Sheet)

| Kebutuhan Diagnosa | Perintah Terminal |
|---|---|
| **Cek status semua Pod di cluster** | `kubectl get pods -A -o wide` |
| **Buka rekam medis & riwayat Events**| `kubectl describe pod <nama-pod>` |
| **Streaming log aplikasi** | `kubectl logs -f <nama-pod>` |
| **Baca log container yang baru mati** | `kubectl logs <nama-pod> --previous` |
| **Masuk shell container** | `kubectl exec -it <nama-pod> -- sh` |
| **Bypass port ke laptop Mac** | `kubectl port-forward svc/<name> 9090:80` |
| **Cek pemakaian CPU & RAM Pod** | `kubectl top pods` |
| **Cek Pod dengan filter label** | `kubectl get pods -l app=<nama>,env=prod` |
| **Investigasi Pod tanpa shell** | `kubectl debug -it <pod> --image=alpine` |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan reflektif berikut untuk membuktikan ketajaman kemampuan troubleshooting milikmu:

1. **Kamu menemukan sebuah Pod berstatus `CrashLoopBackOff`. Sebutkan 5 langkah urutan investigasi sistematis (*Debug Flow*) yang harus kamu jalankan untuk menemukan akar masalahnya!**
2. **Saat menjalankan perintah `kubectl logs <pod-bermasalah>`, terminal hanya menampilkan baris log yang sangat sedikit dan terputus. Mengapa menambahkan parameter `--previous` menjadi sangat penting pada kasus ini?**
3. **Sebuah container mati dengan `Exit Code 137`. Jelaskan rumus matematika sistem operasi Linux di balik angka 137 tersebut, sinyal apa yang dikirimkan, dan apa penyebab utamanya?**
4. **Apa perbedaan fungsi praktis antara perintah `kubectl describe pod <nama>` dengan perintah `kubectl logs <nama>`? Informasi apa yang hanya bisa ditemukan di `describe` dan tidak ada di `logs`?**
5. **Jika sebuah aplikasi di production menggunakan image *Distroless* (tanpa shell `sh`/`bash`), bagaimana cara seorang SRE masuk dan menginvestigasi jaringan di dalam Pod tersebut tanpa mematikan aplikasi yang sedang berjalan?**

---
*Selamat! Kamu telah menguasai seluruh materi teori Modul 2.1. Sekarang saatnya menguji keahlianmu di laboratorium praktik sesungguhnya melalui [lab/LAB-01-pod-deployment-service.md](lab/LAB-01-pod-deployment-service.md) dan [lab/LAB-02-ingress-configmap-secret.md](lab/LAB-02-ingress-configmap-secret.md)!*