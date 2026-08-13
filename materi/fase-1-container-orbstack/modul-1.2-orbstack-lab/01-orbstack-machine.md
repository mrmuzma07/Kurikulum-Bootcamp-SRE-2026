# 01 — OrbStack Machine

> Linux VM asli di Mac, filesystem sharing, dan networking yang stabil — apa sebenarnya OrbStack.

## Tujuan
- Paham arsitektur OrbStack (container runtime + Machine Linux)
- Bisa membuat, mengakses SSH, & mengelola OrbStack Machine
- Paham filesystem sharing antara Mac ↔ Machine/container
- Mengerti networking mode OrbStack (IP stabil, DNS otomatis)

## 1. Apa Itu OrbStack?

OrbStack = runtime Linux/container untuk macOS, ringan, **native ARM64**. Pengganti Docker Desktop yang lebih hemat RAM di Apple Silicon.

Dua kemampuan utuh:
1. **Container runtime** — kompatibel Docker API (`docker` CLI jalan tanpa konfigurasi).
2. **Linux Machine** — VM Linux asli (Ubuntu/Debian) dengan kernel sendiri, bisa SSH, systemd penuh.

```
 ┌───────────────────────────────────┐
 │  macOS (host)                     │
 │  ┌─────────────────────────────┐  │
 │  │ OrbStack (lightweight)      │  │
 │  │  ├── container runtime      │  │  ← docker run ...
 │  │  │   (compat Docker API)    │  │
 │  │  └── Linux Machine(s)      │  │  ← VM Ubuntu (kernel sendiri, systemd)
 │  │      (devbox, lab01, ...)   │  │     SSH, systemd, apt — server sungguhan
 │  └─────────────────────────────┘  │
 └───────────────────────────────────┘
```

```bash
orb version
docker version                 # kompatibel Docker API
orbctl status 2>/dev/null || orb status
```

## 2. Kenapa Ada "Machine" Selain Container?

Container (Fase 1.1) berbagi kernel host — tapi Mac **bukan Linux**. Untuk container Linux di Mac, dibutuhkan **kernel Linux**. OrbStack menjalankan kernel Linux ringan (machine) sebagai fondasi, dan container jalan di atasnya.

Lebih dari itu, **Machine** memberi VM Linux penuh berguna untuk:
- Simulasi **server on-prem** (systemd, SSH, static IP — seperti Fase 4 Ansible).
- Menjalankan service yang butuh systemd/VM (k3s, database dengan init system).
- Lab "production-like" yang tidak mungkin di container biasa.

**Aturan praktis:**
- **Container** (`docker run`) → untuk app, compose stack, k3d (Fase 1.1 & topik ini).
- **Machine** (`orb`/OrbStack UI) → untuk VM server, k3s, Ansible target (Fase 2.2 & 4).

## 3. Membuat & Mengelola Machine

```bash
# Buat machine (lewat UI OrbStack: Machines → New, atau CLI)
orb create -a ubuntu:24.04 devbox          # arch default = host (arm64)
orb ls                                       # daftar machine
orb start devbox
orb stop devbox
orb rm devbox                                # hapus (hati-hati)
```

Akses:
```bash
orb ssh devbox                               # SSH langsung (tanpa setup key)
# atau dari OrbStack UI: klik machine → Open terminal
```

Di dalam machine:
```bash
uname -a                      # Linux ... aarch64 GNU/Linux
systemctl is-system-running   # running (systemd penuh, beda dengan container)
sudo apt update && sudo apt install -y nginx
sudo systemctl enable --now nginx
```

**Machine ≠ container**: machine punya init system (systemd), boot lengkap, IP persisten. Container biasa tidak punya systemd (kecuali dibuat khusus).

## 4. SSH "Sungguhan" ke Machine

`orb ssh` adalah pintu cepat, tapi untuk Ansible/Fase 4 kita butuh **SSH standar** (key-based, seperti Modul 0.1 LAB-01):

```bash
# Dapatkan IP machine
orb ip devbox                                 # mis. 192.168.97.10

# Salin public key (dari ~/.ssh/id_ed25519.pub modul 0.1) ke machine
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@$(orb ip devbox)
# atau manual: orb ssh devbox, lalu paste ke ~/.ssh/authorized_keys

# Buat alias di ~/.ssh/config (Mac)
cat >> ~/.ssh/config <<EOF
Host devbox
    HostName $(orb ip devbox)
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
EOF

ssh devbox                                    # sekarang SSH biasa, tanpa orb
```

Machine OrbStack ini persis `lab01` dari Modul 0.1 LAB-01 — keduanya OrbStack Machine. Sekarang kita paham apa di baliknya.

## 5. Filesystem Sharing — Mac ↔ Machine/Container

OrbStack otomatis **share folder Mac** ke machine & container — nyaman untuk dev.

```bash
# Di machine: home Mac ter-mount
orb ssh devbox
ls /Users/<mac-user>/Developer         # folder Mac terlihat di machine (orbstack fs sharing)
# atau mount point explicit:
ls /mnt/mac                            # root filesystem Mac
```

```bash
# Container: bind mount folder Mac
docker run --rm -v "$PWD:/work" alpine ls /work
```

**Catatan:** filesystem sharing memudahkan edit kode di Mac, build/jalan di Linux. Tapi untuk data **production-like** (DB besar, I/O berat), pakai volume OrbStack (lebih cepat dari bind mount Mac).

## 6. Networking OrbStack — IP Stabil & DNS

Keunggulan OrbStack: IP stabil & DNS otomatis.

```bash
# Container: dapat IP di network OrbStack, resolve via DNS internal
docker run -d --name web alpine sleep 3600
docker inspect web --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# DNS: container saling resolve via nama (seperti compose, tapi global)
docker run --rm alpine ping -c1 web      # resolve nama container

# Machine: IP stabil, tidak berubah saat reboot
orb ip devbox
```

**IP stabil machine** = kenapa OrbStack bagus untuk simulasi **static IP on-prem** (Fase 2.3 MetalLB, Fase 4). Tidak seperti Docker Desktop yang IP berubah-ubah.

```bash
# Port forwarding machine/container → Mac
orb ssh -L 8080:localhost:80 devbox      # forward devbox:80 ke Mac:8080
# atau container:
docker run -p 8080:80 nginx              # Mac:8080 → container:80
```

## 7. Memilih OS & Arch Machine

```bash
orb create -a ubuntu:24.04 devbox        # -a arch; default host arch (arm64 di M5)
orb create -a debian:12 debox
# Arch spesifik (jarang butuh; machine emulasi lambat):
# orb create --arch amd64 ubuntu:24.04 amdbox
```

Untuk bootcamp: **arm64** (native, cepat). Simulasi amd64 production dilakukan lewat container multi-arch (Modul 1.1) atau CI runner, bukan machine emulasi.

## Latihan Cepat (15 menit)

```bash
# 1. Buat machine (kalau belum)
orb create -a ubuntu:24.04 devbox 2>/dev/null || orb start devbox
orb ls

# 2. Akses & cek systemd
orb ssh devbox 'systemctl is-system-running; uname -m'

# 3. IP stabil
orb ip devbox
orb stop devbox && orb start devbox && orb ip devbox     # IP sama (stabil)

# 4. Filesystem sharing
orb ssh devbox 'ls /mnt/mac/Users 2>/dev/null || ls /Users 2>/dev/null' | head

# 5. DNS container global
docker run -d --name orbweb alpine sh -c 'sleep 3600'
docker run --rm alpine ping -c1 orbweb
docker rm -f orbweb
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Buat VM Linux | `orb create -a ubuntu:24.04 <name>` |
| SSH cepat | `orb ssh <name>` |
| SSH standar | `ssh <name>` (setelah ssh-copy-id + config) |
| IP machine | `orb ip <name>` |
| Start/stop | `orb start/stop <name>` |
| Filesystem Mac di machine | `/mnt/mac`, `/Users/<user>` |
| Container DNS global | nama container resolve antar-container |

## Cek Pemahaman

1. Kenapa OrbStack butuh "Machine" (kernel Linux) padahal Mac sudah bisa `docker run`?
2. Kapan Anda pakai **container** vs **Machine** OrbStack? Beri contoh masing-masing dari bootcamp ini.
3. Kenapa IP stabil OrbStack Machine penting untuk simulasi on-prem/MetalLB?
4. Beda filesystem sharing (bind mount Mac) vs volume OrbStack untuk data berat?