# Kuis & Kunci Jawaban — Modul 2.2 k3s untuk Simulasi Production

> **Cara kerja:** jawab sendiri tanpa buka catatan. Cek di bagian bawah. Target ≥ 80% (15 dari 18).

---

## Bagian A — Pilihan Ganda (10 soal)

**1. k3s cocok untuk on-prem karena...
- A. punya UI grafis lengkap
- B. satu biner ringan, embedded etcd, install `curl | sh`
- C. hanya jalan di cloud
- D. butuh Kubernetes penuh terpisah

**2. Setelah `curl -sfL https://get.k3s.io | sh`, k3s jalan sebagai...
- A. container Docker
- B. systemd service (`k3s.service`), auto-start saat reboot
- C. proses tanpa supervisor
- D. Pod di cluster lain

**3. Kenapa kubeconfig dari VM harus diganti `127.0.0.1` → IP VM sebelum dipakai dari Mac?
- A. karena etcd butuh IP publik
- B. agar API server (6443) reachable dari Mac; `127.0.0.1` hanya berlaku di dalam VM
- C. karena TLS cert menolak localhost
- D. tidak perlu diganti

**4. Jumlah server HA yang BENAR (etcd quorum) adalah...
- A. 2 (hemat)
- B. 3 atau 5 (ganjil; 3 tahan 1 gagal)
- C. genap saja
- D. 1 sudah HA

**5. Pada cluster 3 server, jika 1 server mati...
- A. cluster mati (quorum hilang)
- B. cluster tetap jalan (quorum 2/3 masih mayoritas)
- C. data hilang
- D. semua Pod dihapus

**6. Pada cluster 2 server, jika 1 server mati...
- A. cluster tetap jalan
- B. cluster mati/baca-saja (quorum 1/2 bukan mayoritas)
- C. tidak berpengaruh
- D. etcd auto-expand

**7. Flag untuk menginisialisasi cluster etcd HA pada server pertama adalah...
- A. `--ha`
- B. `--cluster-init`
- C. `--etcd-start`
- D. `--multi-node`

**8. ServiceLB (klipper) harus di-disable sebelum MetalLB karena...
- A. MetalLB lebih cepat
- B. keduanya berebut kontrol alamat LoadBalancer → bentrok
- C. ServiceLB tidak bekerja di VM
- D. MetalLB menggantikan CoreDNS

**9. Setelah `--disable servicelb`, Service `type=LoadBalancer` akan...
- A. dapat IP otomatis dari MetalLB
- B. status `EXTERNAL-IP: <pending>` (tidak ada yang beri IP)
- C. error saat dibuat
- D. jadi ClusterIP otomatis

**10. Perintah uninstall k3s (single-node) adalah...
- A. `kubectl delete cluster`
- B. `sudo /usr/local/bin/k3s-uninstall.sh`
- C. `orb rm k3s-cp1`
- D. `systemctl remove k3s`

## Bagian B — Esai (8 soal)

**11.** Jelaskan 2 perbedaan teknis install k3d vs k3s (dari sudut "rumahnya": container vs VM, systemd, kubeconfig).

**12.** Kenapa jumlah server HA harus ganjil (3/5)? Jelaskan apa itu quorum & apa yang terjadi pada 2 server saat 1 mati.

**13.** Apa fungsi token (`/var/lib/rancher/k3s/server/node-token`)? Apa risiko jika token bocor ke Git?

**14.** Sebut 2 alasan seseorang menonaktifkan Traefik bawaan k3s (bukan servicelb).

**15.** Saat ada Traefik + nginx-ingress berjalan bersama, bagaimana K8s tahu Ingress mana ditangani siapa? (konsep apa)

**16.** Di cloud, Service `type=LoadBalancer` otomatis dapat IP dari provider. Di on-prem tanpa MetalLB, apa yang terjadi? Apa solusi "production-like"-nya (modul mana)?

**17.** Anda menginstall k3s HA manual di 5 VM. Sebut 1 alasan ini painful & apa solusinya di bootcamp (fase mana, tool apa).

**18.** Sebut 2 kebiasaan agar tidak salah `kubectl delete` di production saat punya context `k3d-lab` (latihan) & `k3s-ha` (production).

---

## Kunci Jawaban

### Bagian A

**1. B** — k3s = satu biner ringan, embedded etcd (HA tanpa setup terpisah), install via `curl | sh`. Dirancang Rancher untuk edge/on-prem. Bukan cloud-only, bukan butuh K8s terpisah.

**2. B** — k3s terpasang sebagai systemd service (`k3s.service`); `systemctl is-enabled k3s` = enabled → auto-start saat VM reboot. (k3d = container, beda.)

**3. B** — `127.0.0.1` di kubeconfig hanya reachable di dalam VM. Dari Mac, API server (6443) harus diakses via IP VM (`orb ip`). Tanpa ganti, `kubectl` dari Mac connection refused.

**4. B** — ganjil (3 atau 5). etcd quorum = mayoritas. 3 → tahan 1; 5 → tahan 2. 2 server tidak punya mayoritas yang berguna (lihat soal 6).

**5. B** — 3 server, 1 mati → 2 hidup = mayoritas (quorum 2/3) → cluster tetap jalan. Pod di node mati reschedule.

**6. B** — 2 server, 1 mati → 1 hidup = bukan mayoritas (1/2) → etcd read-only, cluster berhenti (tidak bisa write). Ini kenapa jangan 2 server.

**7. B** — `--cluster-init` menginisialisasi embedded etcd cluster pada server pertama. Server lain join via `--server https://CP1:6443 --token`.

**8. B** — ServiceLB (klipper) dan MetalLB sama-sama mengelola Service `type=LoadBalancer`. Keduanya aktif bersama = bentrok (dua LB berebut IP). Disable servicelb dulu, MetalLB ambil alih.

**9. B** — tanpa servicelb (dan MetalLB belum dipasang), tidak ada komponen yang assign IP eksternal → Service LoadBalancer `EXTERNAL-IP: <pending>`. Inilah "siap MetalLB".

**10. B** — `sudo /usr/local/bin/k3s-uninstall.sh` (single-node). Agent: `k3s-agent-uninstall.sh`. `orb rm` hapus VM (bukan uninstall k3s).

### Bagian B

**11.** Dua perbedaan teknis:
- **"Rumah":** k3d menjalankan k3s **sebagai container Docker** (ephemeral, `k3d cluster delete`); k3s diinstall **di VM Linux dengan systemd** (persistent, auto-start saat reboot, kernel sendiri).
- **Kubeconfig:** k3d auto-merge (`k3d kubeconfig merge`); k3s harus **salin manual** dari `/etc/rancher/k3s/k3s.yaml` & ganti `127.0.0.1` → IP VM.
(Boleh juga sebut: k3d tidak bisa MetalLB L2 / Ansible target; k3s di VM bisa.)

**12.** Quorum = mayoritas node etcd harus hidup agar cluster bisa write state. Jumlah ganjil (3/5) agar ada mayoritas jelas saat 1 (atau 2) mati. **2 server**: saat 1 mati, tersisa 1 dari 2 = bukan mayoritas (butuh 2) → etcd read-only, **cluster berhenti** (tidak bisa scheduling/scale). 3 server: 1 mati → 2/3 mayoritas → cluster tetap jalan. 5 server: 2 mati → 3/5 mayoritas → tetap jalan. Inilah kenapa ganjil & minimal 3.

**13.** Token (`node-token`) = kredensial yang dipakai node (server/agent) untuk **join** ke cluster (autentikasi ke API server pertama). Kalau bocor ke Git, siapa saja dengan token + reachabilitas API server bisa **join node jahat** ke cluster (eksekusi Pod, akses secret) — compromise cluster. Risiko: jangan commit token; di Fase 6 pakai SOPS/Vault untuk secret bootstrap (atau Ansible vault — Fase 4).

**14.** Dua alasan disable Traefik:
- **Konsistensi/standar tim:** tim sudah pakai nginx-ingress (lebih umum, tutorial banyak); Traefik bawaan berbeda konfigurasi/CRD → ganti agar seragam.
- **Kontrol penuh / versi:** ingin install Traefik/nginx versi spesifik sendiri (bukan yang dibundle k3s), atau ingin fitur/addon tertentu (cert-manager + nginx, dsb).
(Boleh juga: hemat resource di edge kalau tidak pakai Ingress layer 7.)

**15.** **IngressClass**. Tiap ingress controller punya class sendiri (`traefik`, `nginx`). Ingress menunjuk class via `spec.ingressClassName: nginx` (atau annotation `kubernetes.io/ingress.class: nginx` — lama/deprecated). Controller hanya menangani Ingress yang class-nya cocok. Tanpa IngressClass, saat ada >1 controller, routing ambigu.

**16.** Tanpa MetalLB (dan servicelb disabled), Service `type=LoadBalancer` → `EXTERNAL-IP: <pending>` selamanya (tidak ada komponen yang assign IP eksternal di on-prem). Cloud-provider LB controller tidak ada. Solusi "production-like": **MetalLB** (Modul 2.3) — mengisi kekosongan dengan assign IP dari pool (L2 via ARP, atau BGP). Sebelum MetalLB, pakai NodePort sebagai placeholder (port tinggi, kurang elegan).

**17.** Painful karena: install di **5 VM manual** — perintah berulang (curl|sh, token, IP), rentan salah ketik/tidak konsisten (flag disable lupa di 1 node), sulit direproduksi. Solusi: **Ansible (Fase 4)** — IaC/configuration-as-code: inventory (tabel node IP = topologi.md), role/playbook meng-automasi install k3s + flag konsisten di semua node, idempotent. "Rasakan dulu manual untuk menghargai otomasi" — prinsip bootcamp.

**18.** Dua kebiasaan:
- **Cek context sebelum operasi destruktif:** `kubectl config current-context` sebelum `delete`/`edit`/`scale`; atau pakai prompt/alias yang tampilkan context+namespace aktif.
- **Namespace & RBAC terbatas:** jalankan latihan di namespace `lab`/`demo` (bukan `default`/`kube-system`); di context production (`k3s-ha`) pakai akun/serviceAccount terbatas (tidak admin cluster) sehingga `delete -A`/`delete ns` diblok RBAC. Hindari `-A` (`--all-namespaces`) pada perintah destruktif.
(Bonus: `kubectl auth can-i delete pods -n prod` untuk verifikasi izin sebelum jalan.)

---

## Penilaian

| Benar | Skor |
|---|---|
| 16–18 | Expert — siap Modul 2.3 (MetalLB) |
| 14–15 | Lulus — boleh lanjut, perbaiki yang salah |
| 10–13 | Belum lulus — ulang materi, kerjakan ulang lab |
| < 10 | Ulangi semua materi, lanjut mentor |