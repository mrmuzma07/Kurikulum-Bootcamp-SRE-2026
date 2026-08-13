# 02 — Resource Limit & OrbStack vs Docker Desktop

> Mengatur RAM/CPU agar Mac tetap responsif, dan kenapa OrbStack dipilih di Apple Silicon.

## Tujuan
- Bisa mengatur resource limit OrbStack (memori/CPU)
- Paham dampak limit terhadap container/Machine & Mac
- Bisa menjelaskan perbedaan OrbStack vs Docker Desktop & trade-off-nya
- Bisa memantau penggunaan resource OrbStack

## 1. Kenapa Resource Limit Penting di Laptop

Bootcamp ini akan menjalankan **banyak hal sekaligus** di satu Mac: beberapa OrbStack Machine, cluster k3d, nanti cluster k3s multi-node, observability stack (Prometheus, Grafana, Loki, ...). Tanpa limit, OrbStack/VM bisa habiskan seluruh RAM → **Mac hang**, semua berhenti.

**Aturan praktis:** alokasikan ~60–70% RAM Mac untuk OrbStack, sisanya untuk macOS + editor + browser. Contoh M5 16GB → set OrbStack ~8GB.

## 2. Mengatur Limit OrbStack

```bash
# Lihat konfigurasi & resource saat ini
orb info 2>/dev/null || orb status

# Set limit via UI: OrbStack → Settings → Resources
#   - Memory: 8 GB
#   - CPUs:  4 (atau biarkan "auto")
#   - Disk:  64 GB (SSD Mac terbatas)
```

```bash
# Lihat pemakaian resource live (di machine/container)
orb stats                       # atau OrbStack UI: dashboard resource
docker stats                    # per-container CPU/mem/net/io
```

| Setting | Pengaruh |
|---|---|
| Memory limit | total RAM yang boleh dipakai semua container+machine OrbStack |
| CPU | berapa core host dialokasikan (auto = semua, pakai konteks) |
| Disk | ukuran virtual disk OrbStack (image/layer/volume simpan di sini) |

**Catatan:** memory limit OrbStack = total **semua** workload di dalamnya. Kalau 5 container masing butuh 2GB, butuh ≥10GB limit. Set individual container limit (`--memory`) juga (Modul 1.1) untuk mencegah satu container rakus.

## 3. Per-Container & Per-Machine Limit

Selain limit global OrbStack, set limit per workload:

```bash
# Container (dari Modul 1.1)
docker run -d --name app --memory=512m --cpus=0.5 alpine sleep 3600
docker stats --no-stream

# Machine: set resource saat create
orb create -a ubuntu:24.04 -c 2 -m 2g smallbox      # 2 CPU, 2GB RAM (cek flag: orb create --help)
```

**Hierarki limit:**
```
 Mac RAM (16GB)
 └─ OrbStack global limit (8GB)      ← tidak boleh dilampaui total
    ├─ machine devbox (2GB)          ← per-machine
    ├─ container app (512MB)         ← per-container
    └─ container db (1GB)
```

## 4. Memantau & Membersihkan Resource

SSD Mac cepat tapi terbatas. Image & volume menumpuk.

```bash
# Disk usage OrbStack
docker system df                         # image/container/volume size
docker images                            # daftar image (sering ada yang lupa)
docker volume ls

# Bersih-bersih (hati-hati — hapus yang tidak terpakai)
docker system prune                       # hapus container stop + dangling image + unused network
docker system prune -a                    # + semua image tidak terpakai (agresif)
docker volume prune                       # hapus volume tidak terpakai (data hilang!)
# Cluster k3d (topik 03) punya image banyak:
# k3d cluster delete <name>   → hapus cluster; image ikut bersih nanti via prune
```

**Rutin:** setiap selesai fase besar, jalankan `docker system prune` agar tidak penuh disk. Cek `docker system df` dulu sebelum `-a`.

## 5. OrbStack vs Docker Desktop

| Aspek | OrbStack | Docker Desktop |
|---|---|---|
| Platform | macOS (Apple Silicon & Intel) | cross-platform (Mac/Win/Linux) |
| Arch | **native ARM64** (cepat di M-series) | ARM64 via qemu/VM (lebih berat) |
| RAM usage | ringan (~1GB idle) | berat (~3–4GB idle) |
| Startup | cepat (~2–3 detik) | lambat (~10–30 detik) |
| Linux VM (Machine) | native, ringan, IP stabil | "Docker Desktop VM" tersembunyi, IP berubah |
| Filesystem sharing | cepat (native FUSE/virtiofs) | lebih lambat |
| Docker API compat | ✅ (docker CLI jalan) | ✅ (original) |
| Harga | free untuk personal (cek license terkini) | free personal / berbayar tim besar |
| Enterprise policy | lisensi berbeda — cek kebutuhan | lisensi Docker Inc. |

**Kenapa OrbStack untuk bootcamp ini (Mac M5):**
1. **Native ARM64** → container Linux arm64 jalan cepat (tidak emulasi untuk dev).
2. **Hemat RAM** → bisa jalankan banyak VM/cluster tanpa Mac hang.
3. **Machine dengan IP stabil** → ideal simulasi on-prem (static IP, MetalLB).
4. **`docker` CLI kompatibel** → semua perintah Modul 1.1 jalan tanpa ubah.

**Trade-off / kapan Docker Desktop mungkin lebih cocok:**
- Tim besar dengan policy lisensi Docker Inc. wajib.
- Butuh kompatibilitas feature Docker Desktop spesifik (scout, extension).
- Cross-platform sama persis di Win/Linux (OrbStack Mac-only).

## 6. Memantau dari Sisi Mac

```bash
# Resource Mac secara keseluruhan
top -l 1 | grep -E "PhysMem|CPU"          # memori & CPU host
# atau Activity Monitor → lihat proses "OrbStack"/"com.docker" / qemu
```

Jika Mac lag saat OrbStack berjalan: turunkan memory limit OrbStack, atau hentikan machine/cluster tidak terpakai.

## Latihan Cepat (10 menit)

```bash
# 1. Lihat resource OrbStack
orb info 2>/dev/null || orb status
docker stats --no-stream

# 2. Jalankan container ber-memori & amati
docker run -d --name hog --memory=1g alpine sh -c 'yes > /dev/null & sleep 3600'
docker stats hog
docker rm -f hog

# 3. Disk usage
docker system df
docker images | wc -l

# 4. Bersih-bersih ringan
docker system prune -f             # buang yang tidak terpakai (aman)
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Set limit global OrbStack | Settings → Resources (UI) |
| Limit per container | `docker run --memory --cpus` |
| Pantau resource live | `docker stats`, `orb stats` |
| Cek disk OrbStack | `docker system df` |
| Bersih-bersih | `docker system prune` (-a agresif) |
| Bandingkan | OrbStack = native ARM64, ringan, IP stabil |

## Cek Pemahaman

1. Kenapa memory limit global OrbStack harus lebih besar dari jumlah limit per-container?
2. Anda punya Mac 16GB. Berapa perkiraan limit OrbStack yang aman, dan kenapa tidak 14GB?
3. Sebutkan 3 alasan OrbStack dipilih untuk bootcamp ini di Mac M5.
4. Kapan `docker system prune -a` berbahaya dibanding `prune` biasa?