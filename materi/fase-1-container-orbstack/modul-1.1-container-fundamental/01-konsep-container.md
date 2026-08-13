# 01 — Konsep Container

> Image vs container, layer filesystem, dan namespace & cgroup — apa sebenarnya yang berjalan.

## Tujuan
- Bisa menjelaskan image vs container dengan analogi yang tepat
- Paham layer filesystem & kenapa urutan instruksi Dockerfile memengaruhi ukuran
- Mengerti namespace (isolasi) & cgroup (batas sumber daya) secara konsep
- Bisa membedakan container vs VM vs proses biasa

## 1. Apa Itu Container?

Container = **proses Linux biasa** yang diberi dua hal:
1. **Namespace** — isolasi: proses "merasa" sendirian di sistem (punya filesystem, jaringan, PID sendiri)
2. **Cgroup** — pembatas: CPU, memori, I/O dibatasi

Bukan VM. Tidak ada kernel terpisah. Semua container berbagi **kernel host**. Inilah kenapa container ringan & cepat start (milidetik) dibanding VM (detik–menit).

```
 VM                          Container
 ┌───────────┐               ┌───────────┐ ┌───────────┐
 │ App       │               │ App       │ │ App       │
 │ Libs      │               │ Libs      │ │ Libs      │
 │ Kernel    │  ← full OS    │ ──────────│─│───────────│  ← satu kernel host
 │ Driver    │               │  namespace + cgroup (isolasi)  │
 └───────────┘               └────────────────────────────────┘
   berat, lambat                 ringan, cepat
```

```bash
# Bukti container = proses Linux biasa:
docker run -d --name demo alpine sleep 3600
docker exec demo ps aux        # di dalam: PID 1 = sleep (kelihatan sendirian)
ps aux | grep sleep            # di host: sleep juga muncul, tapi PID beda (namespace PID)
docker rm -f demo
```

## 2. Image vs Container

| | Image | Container |
|---|---|---|
| Apa | template read-only | instance yang berjalan |
| Analogi | resep/kelas OOP | hidangan/object |
| Status | tidak berjalan | proses hidup |
| Dari mana | `docker build`, `pull` | `docker run` dari image |
| Bisa berjalan? | tidak | ya |

```bash
docker images                       # daftar image di lokal
docker ps -a                        # daftar container (running + stopped)
docker run hello-world              # buat container dari image hello-world
```

Satu image → **banyak container**. Image tidak berubah saat container berjalan; perubahan disimpan di **layer writable** (tipis) di atas layer image.

## 3. Layer Filesystem — Kenapa Urutan Instruksi Penting

Image = tumpukan **layer** read-only. Tiap instruksi Dockerfile (`RUN`, `COPY`) = satu layer baru.

```
Image "app:v2"
┌────────────────────────────┐
│ Layer 4: CMD ["./app"]     │  ← instruksi terakhir
├────────────────────────────┤
│ Layer 3: COPY app /app      │
├────────────────────────────┤
│ Layer 2: RUN apt install X  │
├────────────────────────────┤
│ Layer 1: FROM ubuntu:24.04  │  ← base
└────────────────────────────┘
```

**Kenapa ini penting untuk ukuran:**
```dockerfile
# BURUK — 2 layer, hapus tidak efektif (file masih ada di layer bawah)
RUN apt update && apt install -y curl
RUN apt clean && rm -rf /var/lib/apt/lists/*    # layer baru "menghapus", tapi curl & cache masih di layer 2

# BAIK — 1 layer, hapus di instruksi yang sama
RUN apt update && apt install -y curl && apt clean && rm -rf /var/lib/apt/lists/*
```

Saat `rm` di layer baru, file **tidak benar-benar hilang** dari image — masih ada di layer lama, hanya di-"tutupi". Image tetap besar. Solusi: gabung instalasi + bersih-bersih di **satu `RUN`** (chain dengan `&&`).

```bash
# Inspeksi layer image:
docker history <image>                 # lihat tiap layer & ukurannya
dive <image>                           # (tool tambahan) lihat isi tiap layer
```

**Layer sharing:** image yang pakai base sama (mis. `ubuntu:24.04`) berbagi layer base → hemat disk & bandwidth. Pull image kedua yang base sama = cepat.

## 4. Writable Layer & Volume

Saat container jalan, ada satu **writable layer tipis** di atas layer image. Semua perubahan (file baru, edit) tersimpan di sini. **Hilang saat container dihapus.**

```bash
docker run -d --name demo alpine sh -c 'echo data > /tmp/x && sleep 3600'
docker exec demo cat /tmp/x        # "data"
docker rm -f demo
docker run --name demo2 alpine cat /tmp/x   # error: file tidak ada (container baru, writable layer kosong)
```

Untuk data yang **harus bertahan**: pakai **volume** atau bind mount.

```bash
# Volume (dikelola Docker/OrbStack, direkomendasikan)
docker volume create mydata
docker run -v mydata:/data alpine sh -c 'echo persistent > /data/x'
docker run -v mydata:/data alpine cat /data/x   # "persistent" — data bertahan

# Bind mount (folder host di-mount ke container)
docker run -v "$PWD:/src" alpine ls /src         # lihat folder kerja host dari dalam container
```

## 5. Namespace — Isolasi

Namespace = membuat proses "merasa" punya sumber daya sendiri. Yang diisolasi:

| Namespace | Yang diisolasi |
|---|---|
| `pid` | proses ID (container punya PID 1 sendiri) |
| `net` | interface, port, routing (container punya IP sendiri) |
| `mnt` | mount point / filesystem |
| `uts` | hostname |
| `ipc` | shared memory, semaphore |
| `user` | UID mapping (di container UID 0, di host UID biasa) |

```bash
# Lihat namespace sebuah container:
docker run -d --name demo alpine sleep 3600
ls -l /proc/$(docker inspect -f '{{.State.Pid}}' demo)/ns/
# ^^^ tiap file = satu namespace; bandingkan PID ns host vs container
docker rm -f demo
```

**Ini dasar Kubernetes:** Pod = sekumpulan container yang **berbagi namespace `net` & `ipc`** (satu IP, satu port space) — sehingga container dalam Pod bisa `localhost` satu sama lain.

## 6. Cgroup — Batas Sumber Daya

Cgroup (control group) = membatasi & menghitung sumber daya. Tanpa ini, satu container bisa habiskan seluruh CPU/RAM host.

```bash
docker run -d --name limited --memory="256m" --cpus="1.0" alpine sleep 3600
docker inspect limited | grep -A5 Memory
docker rm -f limited
```

| Batas | Flag | Catatan |
|---|---|---|
| Memori | `--memory`, `--memory-swap` | OOMKilled kalau melebihi |
| CPU | `--cpus`, `--cpu-shares` | `--cpus=1.0` = 1 core |
| Disk I/O | `--device-read-bps` | jarang dipakai di lab |

```bash
# Simulasi OOMKilled:
docker run --name oomtest --memory="20m" alpine sh -c 'dd if=/dev/zero bs=1M count=50 | head -c 50m > /tmp/big'
docker inspect oomtest --format '{{.State.OOMKilled}}'   # true
docker rm oomtest
```

Ini persis yang terjadi di Kubernetes saat Pod melebihi `resources.limits.memory` → status `OOMKilled` (akan dilatih di Modul 2.4).

## 7. Container vs VM vs Proses Biasa

| Aspek | Proses biasa | Container | VM |
|---|---|---|---|
| Kernel | host | host (berbagi) | sendiri |
| Isolasi | minim | namespace+cgroup | penuh (kernel terpisah) |
| Startup | instan | ms | detik–menit |
| Overhead | nol | sangat kecil | besar (OS penuh) |
| Portabilitas | terikat host | image portable | image disk penuh |

**Kenapa SRE pakai container:** portabel (jalan sama di laptop, CI, server on-prem), ringan (banyak di satu host), isolasi (tidak ganggu tetangga), dan **image = unit deploy** yang reproducible — semua yang kita pelajari selanjutnya (Kubernetes, Helm, ArgoCD) membangun di atas konsep ini.

## Latihan Cepat (15 menit)

```bash
# 1. Image vs container
docker pull alpine
docker images
docker run -d --name c1 alpine sleep 3600
docker ps
docker exec c1 hostname             # beda dengan host
docker rm -f c1

# 2. Layer & history
docker history alpine
docker pull nginx
docker history nginx | head

# 3. Volume (data bertahan)
docker volume create test-vol
docker run --rm -v test-vol:/d alpine sh -c 'echo hello > /d/x'
docker run --rm -v test-vol:/d alpine cat /d/x    # "hello"
docker volume rm test-vol

# 4. Cgroup — limit memori
docker run -d --name lim --memory=50m --cpus=0.5 alpine sleep 3600
docker inspect lim --format 'Mem={{.HostConfig.Memory}} CPU={{.HostConfig.NanoCpus}}'
docker rm -f lim

# 5. Container = proses host
docker run -d --name p1 alpine sleep 3600
PID=$(docker inspect -f '{{.State.Pid}}' p1)
ps -p $PID                          # proses biasa di host, hanya beda view
docker rm -f p1
```

## Ringkasan

| Mau... | Pakai |
|---|---|
| Lihat image lokal | `docker images` |
| Lihat container | `docker ps -a` |
| Lihat layer image | `docker history <image>`, `dive` |
| Data persisten | `docker volume`, `-v` |
| Batas sumber daya | `--memory`, `--cpus` |
| Cek OOM | `docker inspect --format '{{.State.OOMKilled}}'` |
| Buktikan container=proses | `docker inspect -f '{{.State.Pid}}'` + `ps` |

## Cek Pemahaman

1. Kenapa container start jauh lebih cepat dari VM? Apa yang tidak ada di container?
2. Kenapa `RUN apt install` dan `RUN apt clean` di Dockerfile terpisah tidak membuat image kecil?
3. Apa yang terjadi pada data di writable layer saat container dihapus? Bagaimana mencegahnya?
4. Apa beda namespace `pid` vs `net`, dan kenapa Pod Kubernetes berbagi keduanya secara berbeda?
5. Saat sebuah Pod Kubernetes berstatus `OOMKilled`, mekanisme apa (dari konsep ini) yang sedang bekerja?