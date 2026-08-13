# Kuis & Kunci Jawaban — Modul 1.2 OrbStack sebagai Lab Harian

> **Cara kerja:** jawab sendiri tanpa buka catatan. Cek di bagian bawah. Target ≥ 80% (12 dari 15).

---

## Bagian A — Pilihan Ganda (10 soal)

**1. Dua kemampuan utama OrbStack adalah...
- A. Container runtime + Kubernetes engine
- B. Container runtime (kompatibel Docker API) + Linux Machine (VM)
- C. Hypervisor + package manager
- D. Image registry + VM manager

**2. Kenapa OrbStack butuh "Machine" (kernel Linux) padahal Mac sudah bisa `docker run`?
- A. Karena Docker API butuh daemon Linux
- B. Karena Mac bukan Linux; container Linux butuh kernel Linux sebagai fondasi
- C. Karena Machine lebih cepat dari container
- D. Karena container tidak bisa jalan tanpa VM

**3. Yang membedakan OrbStack Machine dengan container biasa...
- A. Machine punya systemd (init system penuh), container biasa tidak
- B. Machine lebih kecil dari container
- C. Machine tidak bisa di-SSH
- D. Container bisa persisten, Machine tidak

**4. IP OrbStack Machine setelah `orb stop` lalu `orb start` akan...
- A. Berubah-ubah acak
- B. Tetap (stabil), berbeda dari Docker Desktop VM
- C. Hilang
- D. Reset ke 127.0.0.1

**5. Folder Mac dapat diakses dari dalam Machine lewat...
- A. `/mnt/docker`
- B. `/mnt/mac` (filesystem sharing OrbStack)
- C. `/Volumes/Mac`
- D. Tidak bisa diakses

**6. Memory limit global OrbStack 8GB artinya...
- A. Tiap container dapat maksimal 8GB
- B. Total semua container+machine OrbStack tidak boleh melebihi 8GB
- C. Mac hanya menyisakan 8GB untuk OrbStack
- D. 8GB dialokasikan per Machine

**7. Kenapa memory limit global OrbStack harus ≥ jumlah limit per-container yang berjalan bersamaan?
- A. Agar container tidak crash
- B. Karena limit global adalah total; jika jumlah per-container melebihi global, ada yang tidak bisa jalan/OOM
- C. Tidak ada hubungannya
- D. Agar Mac tidak hang

**8. `docker system prune -a` berbahaya dibanding `prune` biasa karena...
- A. Menghapus semua image tidak terpakai (termasuk yang mau dipakai nanti), bukan hanya dangling
- B. Menghapus volume
- C. Menghapus Mac filesystem
- D. Tidak ada bedanya

**9. k3d menjalankan k3s sebagai...
- A. VM di OrbStack
- B. Container Docker
- C. Bare-metal process
- D. Kubernetes addon

**10. Fungsi `--port 8080:80@loadbalancer` saat `k3d cluster create` adalah...
- A. Forward Mac:8080 → loadbalancer internal k3d → Traefik Ingress :80
- B. Ekspos Pod langsung ke Mac
- C. Set port API server Kubernetes
- D. Aktifkan TLS otomatis

## Bagian B — Esai (5 soal)

**11.** Jelaskan 2 perbedaan teknis OrbStack vs Docker Desktop yang membuat OrbStack dipilih untuk bootcamp di Mac M5.

**12.** Kenapa k3d cocok untuk latihan harian tetapi **tidak ideal** untuk simulasi MetalLB L2 mode? (sebut aspek teknisnya)

**13.** Saat `k3d cluster stop lab` lalu `kubectl get nodes`, apa yang terjadi? Jelaskan kenapa, dan apa yang terjadi pada Pod saat `start` lagi.

**14.** Anda diminta mensimulasikan Ansible menginstal k3s di 3 server on-prem dengan static IP. Pilih k3d atau VM OrbStack + k3s, dan beri 3 alasan.

**15.** Jelaskan kapan Anda pakai: (a) **container** (`docker run`), (b) **Machine** (`orb create`), (c) **k3d cluster** — masing-masing 1 contoh konkret dari bootcamp ini.

---

## Kunci Jawaban

### Bagian A

**1. B** — OrbStack = container runtime (kompatibel Docker API) + Linux Machine (VM dengan kernel sendiri). Bukan Kubernetes engine (itu k3d/k3s, tools di atasnya).

**2. B** — Mac bukan Linux; container Linux butuh kernel Linux. OrbStack menjalankan kernel Linux ringan sebagai fondasi container, dan Machine memberi VM penuh untuk simulasi server.

**3. A** — Machine punya systemd (init system penuh, boot lengkap, `systemctl is-system-running` = `running`). Container biasa tidak punya init system (PID 1 = app, bukan systemd).

**4. B** — IP OrbStack Machine stabil (tidak berubah saat restart), berbeda dari Docker Desktop VM yang IP-nya berubah. Ini penting untuk simulasi static IP on-prem & MetalLB.

**5. B** — `/mnt/mac` adalah mount point filesystem sharing OrbStack; folder Mac terlihat di Machine. Juga via `/Users/<mac-user>`.

**6. B** — Memory limit global = total RAM yang boleh dipakai **semua** container+machine OrbStack bersamaan, bukan per-container.

**7. B** — Limit global adalah batas total. Kalau 5 container masing `--memory=2g` (total 10GB) tapi global 8GB, OrbStack tidak bisa memberi semua → ada yang OOM/tidak jalan. Per-container limit tidak boleh jumlahnya melebihi global.

**8. A** — `prune -a` menghapus **semua image tidak terpakai** (termasuk yang Anda simpan untuk nanti), bukan hanya dangling image. `prune` biasa hanya hapus dangling. Berbahaya kalau ada image yang belum dipakai tapi mau dipakai besok.

**9. B** — k3d menjalankan k3s (server+agent) sebagai **container Docker**, bukan VM. Itu sebabnya cepat & ringan.

**10. A** — `--port 8080:80@loadbalancer` forward Mac:8080 ke loadbalancer internal k3d, yang route ke Traefik Ingress (bawaan k3s) di port 80. Akses app via host header dari Mac.

### Bagian B

**11.** Dua perbedaan teknis:
- **Native ARM64**: OrbStack jalan native di Apple Silicon (cepat, tanpa emulasi); Docker Desktop ARM64 lewat VM/qemu lebih berat.
- **Hemat RAM**: OrbStack idle ~1GB vs Docker Desktop ~3–4GB; Machine dengan IP stabil (ideal simulasi on-prem) vs Docker Desktop VM IP berubah.
(Boleh juga sebut: startup cepat, filesystem sharing lebih cepat.)

**12.** MetalLB L2 mode bekerja via **ARP** (broadcast di jaringan nyata) untuk mengiklankan service IP ke node. k3d jalan sebagai container dengan IP container — ARP di dalam container/network Docker tidak terpropagasi ke jaringan fisik seperti server sungguhan. VM OrbStack di network nyata dengan IP stabil bisa melakukan ARP/MetalLB L2 dengan benar. Jadi k3d = cepat untuk latihan objek K8s, tapi tidak untuk simulasi bare-metal load balancer.

**13.** `k3d cluster stop lab` menghentikan container k3s (server+agent) tanpa menghapus. `kubectl get nodes` error/gagal karena API server (di server container yang di-stop) tidak merespons. Saat `k3d cluster start lab`, container hidup lagi, etcd/state persist (k3d simpan di volume), sehingga Pod tetap ada (definisi Pod disimpan di etcd) — Pod kembali Running setelah kubelet reconnect. Berbeda dengan `delete` yang menghapus total.

**14.** Pilih **VM OrbStack + k3s** (bukan k3d). Alasan:
- Ansible butuh target **server** dengan SSH & systemd (k3d = container, tidak punya systemd penuh sebagai target yang natural).
- Static IP stabil per VM (k3d IP container berubah/ephemeral) — MetalLB & Ansible inventory butuh IP tetap.
- Simulasi production on-prem nyata: k3s di VM = installer `curl | sh` + systemd service, persis seperti server sungguhan; k3d terlalu "mudah" dan menyembunyikan langkah produksi.
(Ansible install k3s di VM = persis skenario Fase 2.2 & 4.)

**15.**
- (a) **Container** (`docker run`/compose): app web, compose stack app+PostgreSQL (Modul 1.1 LAB-02) — cepat, stateless/semi-stateful, tidak butuh systemd.
- (b) **Machine** (`orb create`): VM Ubuntu `devbox` sebagai simulasi server on-prem untuk Ansible target, atau install k3s production-like dengan systemd (Fase 2.2/4).
- (c) **k3d cluster**: latihan Kubernetes harian — deploy Pod/Service/Ingress cepat, iterasi manifest, CI (Fase 2.1) — cepat & ringan, hapus total saat selesai.

---

## Penilaian

| Benar | Skor |
|---|---|
| 14–15 | Expert — siap Fase 2 Kubernetes |
| 12–13 | Lulus — boleh lanjut, perbaiki yang salah |
| 9–11 | Belum lulus — ulang materi, kerjakan ulang lab |
| < 9 | Ulangi semua materi, lanjut mentor |