# 01 — Runbook Node, Disk, Certificate, dan CrashLoopBackOff

> **Ringkasan:** Mempelajari filosofi dan anatomi penyusunan **Runbook Operasional** standar industri (*The 7-Section Runbook Contract*), serta menyajikan panduan penanganan langkah-demi-langkah (*step-by-step troubleshooting guide*) untuk 4 skenario insiden paling umum di klaster produksi on-premises: **Node Down**, **Disk Full**, **Certificate Expired**, dan **Pod CrashLoopBackOff**.

---

## 1. Filosofi & Kontrak Standar Runbook SRE

### 1.1 Analogi Dunia Nyata: "Buku Prosedur Darurat Pilot Pesawat (Quick Reference Handbook / QRH)"
Ketika indikator mesin pesawat Boeing atau Airbus menyala merah di ketinggian 35.000 kaki, kedua pilot **dilarang keras** berdebat, menebak-nebak tombol, atau mencari tutorial di internet.
Pilot langsung membuka **Quick Reference Handbook (QRH)**—buku panduan darurat berlaminasi yang berisi daftar periksa (*checklist*) teruji:
- **Langkah Verifikasi Awal:** Pastikan sensor mana yang berbunyi (*Read-Only Check*).
- **Pohon Keputusan yang Jelas:** *"Jika indikator A menyala, lakukan tindakan X. Jika indikator B menyala, lakukan tindakan Y"*.
- **Kondisi Berhenti (*Stop Condition*):** Kapan harus membatalkan tindakan dan segera melakukan pendaratan darurat.

Di dunia SRE, **Runbook** adalah QRH versi digital. Ketika engineer on-call dibangunkan pukul 03:00 pagi oleh pager darurat, otak manusia sedang berada dalam kondisi stres dan penurunan fungsi kognitif. Runbook yang baik menuntun engineer mengambil tindakan yang terukur, aman, dan dapat diaudit tanpa bergantung pada ingatan pribadi.

---

### 1.2 Kontrak Standar Runbook 7-Bagian (The 7-Section Runbook Contract)

Setiap Runbook resmi di repositori GitOps wajib mematuhi kerangka 7 bagian berikut:

```yaml
# Kontrak Standar Metadata Runbook SRE
runbook_metadata:
  runbook_id: "RB-CORE-001"
  title: "Panduan Pemulihan Node NotReady"
  revision: "v2.1.0"
  git_commit: "a8f1b2c3d4e5"
  last_reviewed_utc: "2026-10-01T00:00:00Z"
  expires_utc: "2027-04-01T00:00:00Z" # Kadaluwarsa dalam 6 bulan jika tidak ditinjau
  sre_owner: "infra-platform-team"
  escalation_channel: "#oncall-infra-p1"
```

1. **Metadata & Kepemilikan:** ID unik, versi Git, tanggal tinjauan, dan saluran eskalasi.
2. **Gejala & Dampak Bisnis (*Symptoms & Impact*):** Apa alert yang memicu runbook ini dan apa dampaknya bagi pelanggan.
3. **Pemeriksaan Awal Tanpa Mutasi (*Read-Only First Checks*):** Perintah diagnostik yang **TIDAK mengubah state sistem** untuk memvalidasi fakta di lapangan.
4. **Pohon Keputusan Diagnostik (*Diagnostic Decision Tree*):** Logika percabangan untuk menemukan akar masalah.
5. **Kondisi Berhenti (*Stop Condition*):** Kapan engineer **WAJIB BERHENTI** mengeksekusi perintah dan segera melakukan eskalasi.
6. **Langkah Mitigasi Aman & Rollback (*Safe Mitigation & Rollback*):** Perintah teruji untuk menghentikan dampak buruk ke pengguna.
7. **Verifikasi Pasca-Pemulihan (*Post-Check Evidence*):** Bukti metrik yang membuktikan bahwa sistem sudah kembali normal.

---

## 2. Empat Runbook Produksi Lengkap

---

### RUNBOOK 1: Penanganan Node Down / NotReady

#### 1. Gejala & Pemicu
- **Pemicu:** Alert `HostNodeDown` atau status node Kubernetes berubah menjadi `NotReady`.
- **Dampak:** Pod yang berada di node tersebut tidak dapat menerima trafik baru; jika kapasitas klaster menipis, pod tidak dapat dijadwalkan ulang.

#### 2. Pemeriksaan Awal (Read-Only)
```bash
# 1. Periksa status kondisi node dan alasan NotReady
kubectl describe node <NAMA_NODE> | grep -A 8 "Conditions:"

# 2. Uji apakah sistem operasi dan kartu jaringan server fisik masih merespons
ping -c 3 <IP_NODE>

# 3. Uji apakah port SSH bastion atau port Kubelet (10250) masih terbuka
nc -zvw3 <IP_NODE> 22
nc -zvw3 <IP_NODE> 10250
```

#### 3. Pohon Keputusan & Mitigasi
```text
Apakah IP Server Fisik Merespons Ping?
  ├── TIDAK ──> [Masalah Hardware / Trafo Daya / Switch Jaringan]
  │               ├── Buka antarmuka IPMI / iDRAC server: `ipmitool -H <IPMI_IP> power status`
  │               ├── Jika power OFF, nyalakan via IPMI: `ipmitool power on`
  │               └── Jika mati total, hubungi tim teknisi data center fisik!
  │
  └── YA    ──> [Masalah OS Linux / Kubelet / CNI Network]
                  ├── Masuk SSH via Bastion: `ssh <IP_NODE>`
                  ├── Cek apakah daemon k3s/kubelet mati: `sudo systemctl status k3s-agent`
                  ├── Periksa apakah disk root penuh 100%: `df -h /`
                  ├── Cek log error kubelet: `sudo journalctl -u k3s-agent -n 50 --no-pager`
                  └── Restart daemon kubelet jika macet: `sudo systemctl restart k3s-agent`
```

#### 4. Stop Condition & Peringatan Data
- **STOP:** Jika node menyimpan **Local Persistent Volume (Local PV)** database utama, **JANGAN PERNAH** menjalankan `kubectl delete node` secara gegabah! Menghapus node secara paksa dapat memicu pelepasan volume data secara permanen.
- **Rollback:** Jika daemon kubelet gagal menyala setelah direstart, segera lakukan `kubectl cordon <NAMA_NODE>` agar pod tidak dijadwalkan ke server tersebut, lalu eskalasi ke Lead Infra.

---

### RUNBOOK 2: Penanganan Disk Full / Inode Exhaustion

#### 1. Gejala & Pemicu
- **Pemicu:** Alert `HostDiskSpaceRunningOut` (> 85%), atau Pod mengalami pengusiran massal dengan status `Evicted` (`DiskPressure`).
- **Dampak:** Aplikasi gagal menulis transaksi baru, database mengunci tabel (*read-only*), proses penulisan log macet.

#### 2. Pemeriksaan Awal (Read-Only)
```bash
# 1. Cek penggunaan kapasitas partisi disk dan penggunaan Inode (df -i)
df -h
df -i

# 2. Temukan 5 direktori paling memakan tempat di sistem (tanpa mencetak isi rahasia)
sudo du -xh /var 2>/dev/null | sort -rh | head -n 5
```

#### 3. Pohon Keputusan & Mitigasi
```text
Partisi Mana yang Mengalami Disk Full?
  ├── `/var/log/pods` atau `/var/log/journal` Penuh (Log Menumpuk):
  │     ├── Bersihkan log sistem systemd lama: `sudo journalctl --vacuum-time=2d`
  │     └── Paksa rotasi log container: `sudo logrotate -f /etc/logrotate.d/k3s`
  │
  ├── `/var/lib/rancher/k3s/agent/containerd` Penuh (Image Sampah):
  │     └── Jalankan pembersihan image yang tidak terpakai:
  │         `sudo k3s crictl rmi --prune`
  │
  └── Direktori PersistentVolume Database (`/mnt/data/...`) Penuh:
        └── DILARANG MENGHAPUS FILE SECARA MANUAL!
            Lakukan prosedur ekspansi ukuran PVC via Kubernetes / TrueNAS storage!
```

#### 4. Stop Condition & Peringatan Data
- **DILARANG KERAS:** Menjalankan perintah mentah `rm -rf /var/lib/...` atau menghapus file database `.db / .wal`! Menghapus file database yang sedang dibuka oleh proses aktif akan merusak indeks basis data secara permanen.

---

### RUNBOOK 3: Penanganan TLS/SSL Certificate Expired

#### 1. Gejala & Pemicu
- **Pemicu:** Alert `CertificateExpiringSoon` (< 7 hari), atau curl ke endpoint mengembalikan error: `x509: certificate has expired or is not yet valid`.
- **Dampak:** Seluruh panggilan API antar pod atau transaksi web dari browser pengguna ditolak oleh proteksi keamanan TLS.

#### 2. Pemeriksaan Awal (Read-Only)
```bash
# 1. Periksa tanggal kedaluwarsa sertifikat internal k3s
sudo k3s check-config | grep -i cert
sudo openssl x509 -in /var/lib/rancher/k3s/server/tls/server-ca.crt -noout -dates

# 2. Periksa sertifikat Ingress domain publik via OpenSSL
echo | openssl s_client -servername payment.example.com -connect payment.example.com:443 2>/dev/null | openssl x509 -noout -dates
```

#### 3. Pohon Keputusan & Mitigasi
- **Kasus A — Sertifikat Internal Klaster k3s Kedaluwarsa:**
  Sertifikat internal k3s di `/var/lib/rancher/k3s/server/tls` dapat dirotasi otomatis dengan memutar ulang service:
  ```bash
  # Rotasi sertifikat internal k3s dan restart server master
  sudo k3s certificate rotate
  sudo systemctl restart k3s
  ```
- **Kasus B — Sertifikat Ingress Domain Publik (cert-manager):**
  Periksa status resource `Certificate` dan `Order` di namespace terkait:
  ```bash
  kubectl get certificate,certificaterequest,order,challenge -n production
  # Paksa pembaruan ulang cert-manager jika tersangkut
  kubectl renew certificate <NAMA_CERTIFICATE> -n production
  ```

#### 4. Stop Condition & Peringatan Data
- **STOP:** Jika sertifikat yang kedaluwarsa adalah **Root CA Klaster**, pembaruan akan membutuhkan pendistribusian ulang kunci publik CA ke seluruh worker node. Jangan lakukan rotasi Root CA tanpa koordinasi jendela pemeliharaan resmi!

---

### RUNBOOK 4: Penanganan Pod CrashLoopBackOff

#### 1. Gejala & Pemicu
- **Pemicu:** Alert `PodCrashLooping` atau status Pod berulang kali restart (`CrashLoopBackOff`).
- **Dampak:** Kapasitas replika layanan berkurang; jika seluruh pod crash, terjadi pemadaman total.

#### 2. Pemeriksaan Awal (Read-Only)
```bash
# 1. Ambil alasan kematian dan Exit Code container sebelumnya
kubectl describe pod <NAMA_POD> -n production | grep -E "(Exit Code|Reason|Last State):"

# 2. Baca log container tepat sebelum mati pada siklus sebelumnya (--previous)
kubectl logs <NAMA_POD> -n production --previous --tail=50
```

#### 3. Pohon Keputusan Berdasarkan Exit Code
```text
Berapakah Nilai Exit Code Container?
  ├── Exit Code 137 ──> [OOMKilled - Kehabisan Memori RAM!]
  │     ├── Container melampaui `resources.limits.memory`.
  │     └── Mitigasi: Naikkan nilai limit memory di GitOps values.yaml atau perbaiki memory leak.
  │
  ├── Exit Code 1   ──> [Application Exception / Startup Crash]
  │     ├── Baca log: Periksa apakah koneksi database ditolak atau variabel config salah.
  │     └── Mitigasi: Revert commit GitOps terakhir jika masalah terjadi setelah deploy baru.
  │
  ├── Exit Code 139 ──> [Segmentation Fault]
  │     └── Bug library biner internal C/Go atau ketidakcocokan arsitektur ARM64 vs AMD64!
  │
  └── Liveness Probe Failure (Exit Code 0 atau 143)
        └── Aplikasi sehat tapi lambat menjawab healthcheck karena server overload.
            Mitigasi: Naikkan `initialDelaySeconds` atau `timeoutSeconds` pada livenessProbe.
```

---

## 3. Tata Kelola Evidence & Metadata Audit

Seluruh tindakan mitigasi yang dieksekusi berdasarkan Runbook **wajib mencatat bukti rekaman (*Audit Evidence Trail*)** dengan aturan:
- Simpan ID tiket, nama pelaksana, timestamp UTC awal dan akhir perbaikan.
- **DILARANG** menempelkan kredensial, token, isi variabel environment mentah, atau data pribadi nasabah (PII) ke dalam catatan bukti runbook!
- Jika sebuah Runbook telah melewati batas masa tinjau (**`expires_utc`**), engineer dilarang melakukan improvisasi dan wajib memvalidasi langkah tersebut bersama Lead SRE terlebih dahulu.

---

## 4. Cheatsheet & Ringkasan Perintah Penting

```bash
# ==============================================================================
# 1. PERINTAH DIAGNOSA CEPAT SEBELUM EKSEKUSI MITIGASI RUNBOOK
# ==============================================================================

# Menampilkan seluruh pod yang sedang bermasalah di semua namespace
kubectl get pods -A --field-selector status.phase!=Running,status.phase!=Succeeded

# Menampilkan 10 baris event peringatan terbaru di klaster
kubectl get events -A --field-selector type=Warning --sort-by='.metadata.creationTimestamp' | tail -n 10

# Memeriksa utilisasi resource pod saat ini (Cek apakah mendekati limit)
kubectl top pods -n production --sort-by=memory
```

---

## 5. Evaluasi Pemahaman Mandiri

### 5.1 Pilihan Ganda (3 Soal)

#### Soal 1
Mengapa dalam kontrak standar **Runbook SRE**, tahapan **Pemeriksaan Awal (*Read-Only First Checks*)** harus selalu dieksekusi sebelum menjalankan perintah mitigasi apa pun?
- A. Agar jam kerja engineer on-call terlihat lebih lama.
- B. Untuk memvalidasi fakta dan gejala nyata di lapangan secara objektif tanpa risiko merusak data sistem (*non-mutative*), sehingga tindakan perbaikan yang diambil tidak salah sasaran.
- C. Karena Kubernetes tidak mengizinkan pengetikan perintah mutasi di malam hari.
- D. Agar kuota internet server tidak cepat habis.

#### Soal 2
Sebuah Pod aplikasi pembayaran mengalami status `CrashLoopBackOff` dengan informasi container **`Exit Code: 137`**. Apa arti dari kode keluar (*Exit Code*) tersebut, dan tindakan mitigasi apa yang paling tepat?
- A. Aplikasi mengalami pembagian nol; solusinya hapus pod.
- B. Container dihentikan paksa oleh Linux Kernel karena kehabisan alokasi memori RAM (**OOMKilled** / `SIGKILL 128 + 9 = 137`); solusinya adalah menaikkan batas `resources.limits.memory` pada manifes GitOps atau menambal kebocoran memori pada kode aplikasi.
- C. Sertifikat SSL kedaluwarsa; solusinya perbarui cert-manager.
- D. Hard disk server penuh 100%.

#### Soal 3
Kapan seorang engineer on-call **WAJIB MENGHENTIKAN TINDAKAN (*Trigger Stop Condition*)** saat mengeksekusi runbook pemulihan partisi hard disk yang penuh?
- A. Ketika penggunaan disk sudah turun menjadi 50%.
- B. Ketika direktori yang penuh ternyata adalah folder data penyimpanan basis data transaksi (`/var/lib/postgresql/data`) dan belum ada rencana pemulihan backup yang teruji; engineer dilarang keras menghapus file database secara manual dengan `rm -rf`!
- C. Setiap kali jam kerja kantor selesai.
- D. Ketika temperatur prosesor server di bawah 40°C.

---

### 5.2 Soal Kasus Nyata (Studi Kasus SRE)

**Skenario:**
Pada pukul 03:15 dini hari, pager on-call berdering: `HostNodeDown` pada server fisik `worker-02` di data center on-prem.
Seorang engineer junior yang sedang bertugas langsung panik. Ia membuka terminal dan mengetik perintah:
`kubectl delete node worker-02`
dengan harapan Kubernetes akan otomatis memindahkan seluruh beban kerja ke server lain.

Namun, di dalam `worker-02` tersebut terdapat pod database `order-db` yang menggunakan penyimpanan **Local Persistent Volume (Local PV)** di hard disk SSD lokal server tersebut.

**Pertanyaan & Tugas Anda sebagai Senior SRE:**
1. Mengapa tindakan engineer junior tersebut (`kubectl delete node`) melanggar prosedur Runbook dan justru memperparah insiden? Apa yang terjadi pada data database transaksi di dalam Local PV tersebut?
2. Bagaimana prosedur penanganan insiden **Node Down** yang seharusnya dijalankan langkah demi langkah sesuai panduan Runbook 1 di atas?
3. Langkah pemulihan darurat apa yang harus diambil untuk menyelamatkan node `worker-02` tersebut agar kembali bergabung ke dalam klaster?

---

### Kunci Jawaban & Pembahasan

#### Jawaban Pilihan Ganda:
1. **B.** Mengetik perintah perbaikan sebelum memahami fakta sistem sering kali menjadi penyebab utama memburuknya insiden (*self-inflicted outage*).
2. **B.** Exit Code 137 adalah penanda standar sinyal `SIGKILL (9)` yang dikirim oleh OOM Killer kernel Linux (`128 + 9 = 137`) ketika container melampaui batas batas RAM yang ditentukan.
3. **B.** Stop Condition dirancang untuk mencegah keputusan destruktif yang tidak dapat dibatalkan (*irreversible actions*) seperti penghapusan file data secara sembrono.

#### Solusi Soal Kasus Nyata:
1. **Bahaya `kubectl delete node`:**
   - Menghapus objek node dari API Kubernetes tidak memperbaiki server fisiknya yang mati. Sebaliknya, tindakan tersebut menghapus status registrasi node di klaster dan melepaskan ikatan (*unbinding*) pod dari metadata node.
   - Karena `order-db` menggunakan **Local PV** yang secara fisik tersimpan di hard disk lokal `worker-02`, pod database tidak akan pernah bisa berjalan di server lain karena file datanya terkunci di SSD fisik server `worker-02` yang sedang mati!
2. **Prosedur yang Seharusnya Dijalankan Sesuai Runbook:**
   - **Langkah 1 (Read-Only Check):** Uji `ping <IP_WORKER_02>`. Cek konsol daya via IPMI: `ipmitool -H <IPMI_IP> power status`.
   - **Langkah 2 (Hardware Triage):** Jika server mati karena listrik, nyalakan kembali via IPMI (`ipmitool power on`).
   - **Langkah 3 (Safe Isolation):** Jika OS menyala tapi kubelet bermasalah, tandai node dengan `kubectl cordon worker-02` (bukan menghapus node!), lalu login SSH via Bastion untuk memeriksa log kubelet (`journalctl -u k3s-agent`).
3. **Langkah Pemulihan Darurat:**
   - Nyalakan kembali server fisik `worker-02` melalui IPMI/iDRAC.
   - Restart daemon agen k3s: `sudo systemctl restart k3s-agent`.
   - Begitu agent menghubungi control plane kembali, registrasi node akan otomatis pulih di Kubernetes, status Local PV kembali valid, dan pod `order-db` dapat membaca data transaksinya kembali secara utuh tanpa kehilangan data!

---

## 6. Langkah Selanjutnya & Kaitan Modul

Setelah menguasai penyusunan Runbook untuk 4 skenario kritis:
- Lanjutkan ke [02 — Runbook MetalLB, Topology, Evidence, dan Postmortem](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.3-runbook-dokumentasi/02-runbook-metallb-topology-evidence-postmortem.md) untuk mempelajari penanganan insiden jaringan bare-metal MetalLB, dokumentasi topologi arsitektur, dan pelaporan investigasi insiden resmi.
- Praktikkan simulasi di [LAB-01 — Runbook Validation Drill](file:///Users/muzma/Developer/repos/it-oss/training/BootcampSRE/materi/fase-8-sre-practices/modul-8.3-runbook-dokumentasi/lab/LAB-01-runbook-validation-drill.md).
