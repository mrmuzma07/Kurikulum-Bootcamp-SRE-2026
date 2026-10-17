# 02 — k3s Multi-Node & High Availability (HA) dengan Embedded etcd

> **Ringkasan:** Membangun cluster Kubernetes tangguh bertaraf produksi (*Production-Ready*): memahami konsep **High Availability (HA)**, membedah teori **Quorum etcd** dan mengapa jumlah server master wajib ganjil (3 atau 5), menginisialisasi cluster menggunakan parameter sakti **`--cluster-init`**, menghubungkan node Server dan Agent tambahan, memverifikasi konsensus etcd, serta mensimulasikan kegagalan server nyata (*Chaos Testing*) tanpa menghentikan operasional aplikasi.

---

## Tujuan Pembelajaran
Setelah mempelajari materi ini dan menyelesaikan praktiknya di terminal, kamu akan mampu:
- Menjelaskan bahaya *Single Point of Failure (SPOF)* pada cluster single-node dan mengapa production wajib menggunakan arsitektur **High Availability (HA)**.
- Menghitung batas toleransi kegagalan node master menggunakan rumus matematika **Quorum Konsensus Raft (`(N/2) + 1`)**.
- Menjelaskan keunggulan fitur **Embedded etcd** bawaan k3s yang meniadakan kebutuhan instalasi cluster etcd terpisah.
- Membangun cluster k3s 5-node (3 Node Server Control Plane + 2 Node Agent Worker) di atas mesin virtual OrbStack.
- Menjelaskan perbedaan peran parameter instalasi: **`server --cluster-init`**, **`server --server`**, dan **`agent --server`**.
- Mengekstrak dan menyuntikkan token otentikasi cluster (`node-token`) untuk mengamankan proses penggabungan (*join*) node baru.
- Melakukan simulasi matinya server master (*node failure*) dan membuktikan bahwa cluster tetap dapat melayani perintah secara normal.
- Menjalankan prosedur pencopotan instalasi (*uninstall*) multi-node dengan urutan yang benar.

---

## 1. Konsep & Arsitektur: Mengapa Harus High Availability (HA)?

### Tragedi Single Point of Failure (SPOF)
Pada materi sebelumnya, kita memasang k3s di satu mesin virtual `k3s-cp1`.
Di dalam satu mesin tersebut terdapat:
- `kube-apiserver` (Pintu API)
- Database cluster (SQLite / etcd)
- `kubelet` dan Pod aplikasi

Jika komputer server tersebut mati lampu, kehabisan memori, atau kabel jaringannya putus, maka **seluruh cluster Kubernetes mati total seketika (*Single Point of Failure*)**. Tidak ada yang bisa menerima perintah dan seluruh aplikasi pengguna tumbang!

### Definisi High Availability (HA) di Kubernetes
Sebuah cluster dikatakan berkemampuan **High Availability (Ketersediaan Tinggi)** jika:
1. **Memiliki lebih dari 1 server Control Plane (Master).**
2. **Database state cluster (`etcd`) didistribusikan ke beberapa server fisik yang berbeda.**
3. **Jika salah satu server master mati mendadak, server master lainnya yang masih hidup otomatis mengambil alih kepemimpinan tanpa ada gangguan layanan (*zero downtime*)!**

```text
 ┌─ ARSITEKTUR K3S HA MULTI-SERVER DENGAN EMBEDDED ETCD ────────────────────────┐
 │                                                                              │
 │  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐       │
 │  │   k3s-cp1 (VM)  │      │   k3s-cp2 (VM)  │      │   k3s-cp3 (VM)  │       │
 │  │  Control Plane  │      │  Control Plane  │      │  Control Plane  │       │
 │  │  + Embedded etcd│◄────►│  + Embedded etcd│◄────►│  + Embedded etcd│       │
 │  └────────┬────────┘      └────────┬────────┘      └────────┬────────┘       │
 │           │                        │                        │                │
 │           └────────────────────────┼────────────────────────┘                │
 │                                    │ Jaringan Sinkronisasi Raft              │
 │                                    ▼                                         │
 │                    ┌───────────────────────────────┐                         │
 │                    │   KONSENSUS QUORUM (2 dari 3) │                         │
 │                    └───────────────┬───────────────┘                         │
 │                                    │ Melayani Perintah Pod                   │
 │                    ┌───────────────┴───────────────┐                         │
 │                    ▼                               ▼                         │
 │          ┌──────────────────┐            ┌──────────────────┐                │
 │          │   k3s-w1 (VM)    │            │   k3s-w2 (VM)    │                │
 │          │   Agent Worker   │            │   Agent Worker   │                │
 │          └──────────────────┘            └──────────────────┘                │
 └──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Teori Quorum etcd: Mengapa Jumlah Server Wajib Ganjil?

Jantung dari ketahanan cluster Kubernetes terletak pada database **`etcd`**.
etcd tidak menggunakan sistem database tradisional (seperti replikasi master-slave di MySQL), melainkan menggunakan algoritma konsensus terdistribusi bernama **Raft**.

### Rumus Matematika Quorum Raft
Agar sebuah data baru (misal: penambahan Deployment atau Pod baru) sah dicatat ke dalam database, data tersebut harus disetujui oleh **Mayoritas Anggota (Quorum)**:

$$\text{Quorum} = \left\lfloor \frac{N}{2} \right\rfloor + 1$$

*(Di mana $N$ adalah total jumlah server master/etcd di dalam cluster).*

Mari kita lihat tabel konsekuensi logisnya:

| Total Server Master ($N$) | Jumlah Suara Mayoritas (Quorum) | Jumlah Server yang Boleh Mati (Fault Tolerance) | Status & Rekomendasi SRE |
|---|---|---|---|
| **1 Server** | $(1/2) + 1 =$ **1** | **0 Server** | Sangat rentan (SPOF). Hanya untuk lab/dev. |
| **2 Server** | $(2/2) + 1 =$ **2** | **0 Server** | ❌ **DILARANG KERAS!** Menambah dari 1 ke 2 server justru memperburuk keandalan sistem! |
| **3 Server** | $(3/2) + 1 =$ **2** | **1 Server Mati** | ✅ **Standar Minimum Produksi.** Tahan jika 1 server mati. |
| **5 Server** | $(5/2) + 1 =$ **3** | **2 Server Mati** | ✅ **Standar Enterprise Besar.** Tahan jika 2 server mati sekaligus. |

> 🧠 **Membongkar Paradoks: Mengapa 2 Server Lebih Buruk daripada 1 Server?**
> - Pada sistem **1 Server**, kuorumnya adalah 1. Sistem baru mati jika 1 server itu rusak (peluang rusak: 1 titik).
> - Pada sistem **2 Server**, kuorum yang dibutuhkan adalah **2**. Artinya, **KEDUA server tersebut wajib sama-sama hidup!** Jika server nomor 2 mati, server nomor 1 yang tersisa hanya berjumlah 1 suara (`1 < 2`), sehingga server nomor 1 menolak menulis data demi mencegah fenomena perpecahan data (*Split-Brain*).
> Akibatnya: peluang mati pada sistem 2 server menjadi dua kali lipat lebih besar dibanding 1 server! Maka dari itu, **jumlah server master Kubernetes wajib selalu GANJIL (3, 5, atau 7)**.

---

## 3. Topologi Laboratorium Kita: 3 Server + 2 Agent

Untuk mensimulasikan lingkungan data center fisik secara akurat, kita akan menyiapkan **5 buah Mesin Virtual (VM)** di OrbStack:

```text
 DAFTAR RENCANA ALOKASI VM:
 1. k3s-cp1 (198.19.249.10) ➔ Server Master 1 (Penginisialisasi Cluster etcd)
 2. k3s-cp2 (198.19.249.11) ➔ Server Master 2 (Anggota etcd Quorum)
 3. k3s-cp3 (198.19.249.12) ➔ Server Master 3 (Anggota etcd Quorum)
 4. k3s-w1  (198.19.249.20) ➔ Worker Agent 1  (Tempat Pod Aplikasi Bekerja)
 5. k3s-w2  (198.19.249.21) ➔ Worker Agent 2  (Tempat Pod Aplikasi Bekerja)
```

### Mengapa Memisahkan Server (Control Plane) dan Agent (Worker)?
1. **Perlindungan Kinerja etcd:** etcd sangat sensitif terhadap kecepatan baca-tulis disk (*Disk I/O Latency*). Jika Pod aplikasi pengguna yang rakus memori dan CPU ditaruh di server master yang sama, kinerja etcd bisa melambat dan menyebabkan cluster mengalami *timeout*.
2. **Keamanan:** Pekerja luar dan pengguna publik hanya mengakses node Worker (`agent`), sedangkan node Control Plane (`server`) diisolasi di jaringan internal yang lebih terlindungi.

---

## 4. Langkah Instalasi Multi-Node HA Step-by-Step

### Langkah 1: Siapkan Seluruh Mesin Virtual di OrbStack
Buka terminal laptop Mac milikmu dan jalankan perulangan perintah berikut:

```bash
# 1. Buat seluruh VM yang dibutuhkan (Ubuntu 24.04 ARM64):
for vm in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do
  orb create ubuntu:24.04 $vm 2>/dev/null || orb start $vm
done

# 2. Periksa daftar VM dan pastikan statusnya 'running':
orb list
```

Pasang kunci SSH dari Mac ke seluruh VM agar proses otomasi berjalan lancar:
```bash
for vm in k3s-cp1 k3s-cp2 k3s-cp3 k3s-w1 k3s-w2; do
  orb ssh $vm "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
  cat ~/.ssh/id_ed25519.pub | orb ssh $vm "cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
done
```

---

### Langkah 2: Inisialisasi Server Pertama (`k3s-cp1`) dengan `--cluster-init`
Server pertama bertugas sebagai pondasi awal yang mendirikan cluster etcd:

```bash
# Ambil IP k3s-cp1:
CP1_IP=$(orb ip k3s-cp1)

# Jalankan instalasi server pertama dengan parameter --cluster-init:
ssh k3s-cp1 "curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --tls-san ${CP1_IP} \
  --node-external-ip ${CP1_IP}"
```

#### Membedah Tiga Parameter Krusial di Atas:
- **`--cluster-init`**: Memberitahu k3s untuk mengaktifkan database **Embedded etcd** terdistribusi (bukan SQLite lokal) dan bertindak sebagai pemegang suara pertama.
- **`--tls-san ${CP1_IP}`**: Menambahkan Alamat IP server ke dalam sertifikat keamanan TLS API Server (*Subject Alternative Name*). Tanpa parameter ini, ketika laptop Mac atau server lain mencoba menghubungi IP tersebut, koneksi akan ditolak dengan error keamanan sertifikat SSL!
- **`--node-external-ip ${CP1_IP}`**: Mengunci alamat IP yang akan didaftarkan ke Kubernetes agar tidak salah memilih antarmuka jaringan virtual internal.

---

### Langkah 3: Ekstraksi Token Rahasia Cluster
Setiap server atau agent baru yang ingin bergabung ke dalam cluster wajib menyertakan **Token Rahasia** yang disimpan di server pertama:

```bash
# Ambil token dari k3s-cp1 dan simpan ke variabel di terminal Mac:
CLUSTER_TOKEN=$(ssh k3s-cp1 "sudo cat /var/lib/rancher/k3s/server/node-token")
echo "Token Cluster k3s: ${CLUSTER_TOKEN}"
```

---

### Langkah 4: Hubungkan Server Kedua (`k3s-cp2`) dan Ketiga (`k3s-cp3`)
Sekarang kita gabungkan server master 2 dan 3 ke server master 1 agar terbentuk konsensus **3 Node Quorum**:

```bash
for node in k3s-cp2 k3s-cp3; do
  NODE_IP=$(orb ip $node)
  echo "Menggabungkan server master: $node ($NODE_IP)..."
  
  ssh $node "curl -sfL https://get.k3s.io | sh -s - server \
    --server https://${CP1_IP}:6443 \
    --token ${CLUSTER_TOKEN} \
    --tls-san ${NODE_IP} \
    --node-external-ip ${NODE_IP}"
done
```

> 🔍 **Perhatikan Perbedaannya:**
> Pada server 2 dan 3, kita **TIDAK** menulis `--cluster-init`. Kita menulis **`--server https://${CP1_IP}:6443`** dan menyertakan **`--token`**. Parameter ini memerintahkan k3s: *"Jangan buat cluster baru! Hubungi server cp1 dan daftarkan mesin ini sebagai anggota baru etcd cluster yang sudah ada!"*

---

### Langkah 5: Hubungkan Node Pekerja (`k3s-w1` dan `k3s-w2`)
Node pekerja tidak menjalankan etcd dan tidak menjalankan Control Plane. Oleh karena itu, kita menginstalnya menggunakan mode **`agent`** (bukan `server`):

```bash
for node in k3s-w1 k3s-w2; do
  NODE_IP=$(orb ip $node)
  echo "Menggabungkan worker agent: $node ($NODE_IP)..."
  
  ssh $node "curl -sfL https://get.k3s.io | sh -s - agent \
    --server https://${CP1_IP}:6443 \
    --token ${CLUSTER_TOKEN} \
    --node-external-ip ${NODE_IP}"
done
```

---

## 5. Verifikasi Status Kesehatan Cluster dari Mac

Tarik file `kubeconfig` dari `k3s-cp1` ke laptop Mac milikmu dan periksa apakah kelima node sudah bergabung dengan status `Ready`:

```bash
# 1. Ekstraksi kubeconfig ke Mac:
ssh k3s-cp1 "sudo cat /etc/rancher/k3s/k3s.yaml" | sed "s/127.0.0.1/${CP1_IP}/g" > /tmp/k3s-ha.yaml
KUBECONFIG=~/.kube/config:/tmp/k3s-ha.yaml kubectl config view --flatten > ~/.kube/config.tmp
mv ~/.kube/config.tmp ~/.kube/config
rm -f /tmp/k3s-ha.yaml

# 2. Set nama context menjadi 'k3s-ha' dan aktifkan:
kubectl config rename-context default k3s-ha 2>/dev/null || true
kubectl config use-context k3s-ha

# 3. Periksa status 5 node cluster:
kubectl get nodes -o wide
```

*Contoh tampilan output yang membanggakan:*
```text
NAME      STATUS   ROLES                       AGE     VERSION        INTERNAL-IP      OS-IMAGE
k3s-cp1   Ready    control-plane,etcd,master   5m      v1.30.4+k3s1   198.19.249.10    Ubuntu 24.04 LTS
k3s-cp2   Ready    control-plane,etcd,master   3m      v1.30.4+k3s1   198.19.249.11    Ubuntu 24.04 LTS
k3s-cp3   Ready    control-plane,etcd,master   90s     v1.30.4+k3s1   198.19.249.12    Ubuntu 24.04 LTS
k3s-w1    Ready    <none>                      45s     v1.30.4+k3s1   198.19.249.20    Ubuntu 24.04 LTS
k3s-w2    Ready    <none>                      30s     v1.30.4+k3s1   198.19.249.21    Ubuntu 24.04 LTS
```
Perhatikan kolom `ROLES`:
- `k3s-cp1`, `cp2`, dan `cp3` bertindak sebagai **`control-plane,etcd,master`**.
- `k3s-w1` dan `w2` bertindak murni sebagai **Worker Agent (`<none>`)**.

---

### Menguji Persebaran Beban Pod di Node Worker
Mari kita deploy aplikasi web dengan 4 replika dan amati bagaimana Kubernetes menyebarkannya:

```bash
kubectl create deployment web-ha --image=nginx:1.27-alpine --replicas=4
kubectl rollout status deployment/web-ha
kubectl get pods -o wide
```
Kamu akan melihat ke-4 Pod tersebar secara merata di antara node-node yang tersedia!

---

## 6. Simulasi Insiden Nyata: Uji Ketahanan Quorum (*Chaos Testing*)

Sebagai calon SRE, kita tidak boleh hanya percaya pada teori di atas kertas. Mari kita buktikan ketahanan arsitektur High Availability ini dengan **mematikan salah satu server master secara paksa**!

```bash
# --- SIMULASI BENCANA: Matikan paksa server master k3s-cp3! ---
echo "Mematikan server k3s-cp3..."
orb stop k3s-cp3

# Berikan waktu 15 detik bagi Kubernetes untuk mendeteksi hilangnya detak jantung node:
sleep 15

# Periksa status node dari terminal Mac:
kubectl get nodes
```

*Output yang muncul di layar:*
```text
NAME      STATUS     ROLES                       AGE   VERSION
k3s-cp1   Ready      control-plane,etcd,master   10m   v1.30.4+k3s1
k3s-cp2   Ready      control-plane,etcd,master   8m    v1.30.4+k3s1
k3s-cp3   NotReady   control-plane,etcd,master   6m    v1.30.4+k3s1  ◄── MATI!
k3s-w1    Ready      <none>                      5m    v1.30.4+k3s1
k3s-w2    Ready      <none>                      5m    v1.30.4+k3s1
```

### Uji Kelangsungan Hidup Cluster:
Apakah cluster masih bisa menerima perintah baru saat `k3s-cp3` mati?
Mari kita coba melakukan scaling Pod:

```bash
kubectl scale deployment web-ha --replicas=6
kubectl get pods
```
**Luar biasa!** Kubernetes tetap merespons seketika dan membuat 2 Pod baru. Mengapa?
Karena dari total 3 server etcd, masih ada **2 server yang hidup (`k3s-cp1` dan `k3s-cp2`)**.
Sesuai rumus: **`2 suara >= Kuorum 2`**, sehingga cluster tetap mencapai kata sepakat mayoritas dan operasional berjalan normal 100%!

```bash
# --- PEMULIHAN: Hidupkan kembali server k3s-cp3 ---
orb start k3s-cp3
sleep 15
kubectl get nodes
# Node k3s-cp3 otomatis kembali 'Ready' dan menyinkronkan data etcd yang tertinggal!
```

---

## 7. Prosedur Uninstall Multi-Node yang Benar

Jika suatu hari kamu perlu mencopot instalasi cluster multi-node, patuhi urutan baku berikut agar tidak ada proses yang tertinggal (*orphan*):

```bash
# 1. Copot instalasi di seluruh Worker Agent terlebih dahulu:
for node in k3s-w1 k3s-w2; do
  ssh $node "sudo /usr/local/bin/k3s-agent-uninstall.sh"
done

# 2. Copot instalasi di seluruh Server Master:
for node in k3s-cp3 k3s-cp2 k3s-cp1; do
  ssh $node "sudo /usr/local/bin/k3s-uninstall.sh"
done

# 3. Bersihkan context di Mac:
kubectl config delete-context k3s-ha 2>/dev/null || true
```

---

## 8. Hands-On Mandiri Step-by-Step (30 Menit)

Rangkuman skenario praktis pembuatan cluster 3 Server HA:

```bash
# --- TAHAP 1: Inisialisasi CP1 ---
CP1_IP=$(orb ip k3s-cp1)
ssh k3s-cp1 "curl -sfL https://get.k3s.io | sh -s - server --cluster-init --tls-san ${CP1_IP} --node-external-ip ${CP1_IP}"
TOKEN=$(ssh k3s-cp1 "sudo cat /var/lib/rancher/k3s/server/node-token")

# --- TAHAP 2: Join CP2 & CP3 ---
for node in k3s-cp2 k3s-cp3; do
  N_IP=$(orb ip $node)
  ssh $node "curl -sfL https://get.k3s.io | sh -s - server --server https://${CP1_IP}:6443 --token ${TOKEN} --tls-san ${N_IP} --node-external-ip ${N_IP}"
done

# --- TAHAP 3: Konfigurasi Kubeconfig di Mac ---
ssh k3s-cp1 "sudo cat /etc/rancher/k3s/k3s.yaml" | sed "s/127.0.0.1/${CP1_IP}/g" > /tmp/k3s-ha.yaml
KUBECONFIG=~/.kube/config:/tmp/k3s-ha.yaml kubectl config view --flatten > ~/.kube/config.tmp
mv ~/.kube/config.tmp ~/.kube/config
rm -f /tmp/k3s-ha.yaml
kubectl config rename-context default k3s-ha 2>/dev/null || true
kubectl config use-context k3s-ha

# --- TAHAP 4: Verifikasi Node ---
kubectl get nodes -o wide
```

---

## 9. Pola Kegagalan Umum (Failure Modes) & Diagnosa SRE

Berikut adalah 3 kendala yang sering ditemui pada cluster k3s High Availability beserta langkah diagnosanya:

### Kasus 1: Node Kedua Gagal Join dengan Pesan `Failed to join etcd cluster`
- **Gejala:** Saat menginstal k3s di `k3s-cp2`, terminal menggantung (*hang*) lalu memunculkan pesan error kegagalan koneksi ke etcd server 1.
- **Penyebab:**
  1. Port komunikasi antarnode etcd (**port 2380 atau 6443**) terhalang firewall `ufw` di server 1.
  2. Nilai `--token` salah atau terpotong saat disalin.
- **Diagnosa SRE:**
  Uji konektivitas port TCP dari node 2 ke node 1:
  ```bash
  nc -zv <IP_CP1> 6443
  ```
  Jika ditolak (*Connection refused/timed out*), matikan firewall sementara di server 1: `sudo ufw disable`.

---

### Kasus 2: Cluster Berhenti Menerima Perintah Tulis (*Cluster Freeze*)
- **Gejala:** Kamu mematikan 2 server master sekaligus di cluster 3-node. Saat mengetik `kubectl create deployment ...`, perintah membeku dan akhirnya keluar pesan `etcdserver: leader lost` atau `context deadline exceeded`.
- **Penyebab:** **Quorum hilang!** Dari 3 server, hanya tersisa 1 yang hidup (`1 < 2`). Database etcd otomatis mengunci dirinya menjadi mode *Read-Only* untuk mencegah korupsi data.
- **Solusi SRE:** Nyalakan kembali minimal 1 server master yang mati agar jumlah node hidup mencapai angka 2, maka etcd akan otomatis memilih *Leader* baru dan cluster kembali normal.

---

### Kasus 3: Error `x509: certificate is valid for ..., not ...` Saat Menghubungi Master 2
- **Gejala:** Jika `k3s-cp1` mati dan kamu mengarahkan kubeconfig ke IP `k3s-cp2`, muncul error penolakan sertifikat SSL.
- **Penyebab:** Saat menginstal `k3s-cp2`, kamu lupa menyertakan parameter `--tls-san <IP_CP2>`.
- **Solusi:** Selalu sertakan `--tls-san` berisi alamat IP server tersebut pada setiap node Control Plane yang kamu instal.

---

## 10. Ringkasan & Langkah Selanjutnya

### Tabel Jalan Pintas Multi-Node HA (Cheat Sheet)

| Kebutuhan Operasional | Perintah Terminal |
|---|---|
| **Inisialisasi Master Pertama (etcd)** | `curl ... \| sh -s - server --cluster-init --tls-san <IP>` |
| **Join Master Berikutnya (HA Quorum)** | `curl ... \| sh -s - server --server https://<IP1>:6443 --token <T> --tls-san <IP>` |
| **Join Worker Agent Biasa** | `curl ... \| sh -s - agent --server https://<IP1>:6443 --token <T>` |
| **Lokasi Token Cluster di Master 1** | `/var/lib/rancher/k3s/server/node-token` |
| **Quorum 3 Master** | Membutuhkan minimal **2 Server Hidup** (Tahan 1 mati) |
| **Quorum 5 Master** | Membutuhkan minimal **3 Server Hidup** (Tahan 2 mati) |

---

### Uji Pemahaman Mandiri
Jawablah pertanyaan berikut untuk menguji pemahaman arsitekturalmu tentang cluster High Availability:

1. **Jelaskan mengapa di lingkungan produksi kita sangat dilarang membangun cluster dengan 2 server master Control Plane! Mengapa 3 server master jauh lebih andal dibanding 2 server master?**
2. **Apa perbedaan fungsi mendasar antara parameter `--cluster-init` dengan parameter `--server https://...:6443` saat menginstal k3s? Di server mana saja parameter `--cluster-init` boleh dijalankan?**
3. **Mengapa node Worker Agent tidak membutuhkan parameter `--cluster-init` dan tidak membutuhkan parameter `--tls-san`? Apa perbedaan tugas Agent dibanding Server?**
4. **Pada cluster 3 server master, jika server nomor 1 dan server nomor 2 mati bersamaan, apa yang akan terjadi pada Pod aplikasi yang saat itu sedang berjalan di node Worker? Apakah Pod tersebut langsung mati? Dan apakah kamu bisa menambah Pod baru lewat `kubectl`?**

---
*Lanjutkan ke materi berikutnya: [03-disable-komponen-ingress.md](03-disable-komponen-ingress.md) untuk mempelajari cara menonaktifkan komponen bawaan k3s (seperti ServiceLB dan Traefik) guna mempersiapkan cluster menyambut integrasi MetalLB di Modul 2.3!*