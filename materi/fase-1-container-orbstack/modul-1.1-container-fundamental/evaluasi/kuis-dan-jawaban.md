# Kuis & Kunci Jawaban — Modul 1.1 Container Fundamental

> **Cara kerja:** jawab sendiri tanpa buka catatan. Cek di bagian bawah. Target ≥ 80% (16 dari 20).

---

## Bagian A — Pilihan Ganda (10 soal)

**1. Yang membedakan container dengan VM secara mendasar adalah...
- A. Container punya kernel sendiri
- B. Container berbagi kernel host
- C. Container lebih besar dari VM
- D. VM start lebih cepat dari container

**2. Saat sebuah container dihapus, data yang ditulis ke writable layer akan...
- A. Pindah ke image
- B. Tetap ada di volume baru
- C. Hilang
- D. Disinkron ke host otomatis

**3. Kenapa `RUN apt install` dan `RUN apt clean` di Dockerfile **terpisah** tidak membuat image kecil?
- A. `apt clean` tidak berfungsi
- B. File masih ada di layer `install`, layer `clean` hanya menutupi
- C. Layer tidak bisa dihapus
- D. apt cache otomatis dibersihkan

**4. `EXPOSE 8080` di Dockerfile berfungsi sebagai...
- A. Membuka port 8080 ke host
- B. Dokumentasi port yang dipakai (tidak benar-benar membuka)
- C. Menjalankan service di port 8080
- D. Memblok port lain

**5. Keuntungan utama multi-stage build adalah...
- A. Lebih cepat build-nya
- B. Image final hanya berisi hasil build, tanpa toolchain → kecil & aman
- C. Tidak butuh base image
- D. Bisa tanpa Dockerfile

**6. Kenapa menjalankan container sebagai non-root user penting?
- A. Agar bisa bind port < 1024
- B. Kalau ada celah, attacker tidak dapat root di container
- C. Non-root lebih cepat
- D. Docker mewajibkan non-root

**7. Saat `docker pull` image multi-arch di Mac M5, yang terjadi adalah...
- A. Pull semua arsitektur (boros)
- B. Otomatis pilih layer arm64 sesuai host
- C. Gagal karena bukan amd64
- D. Emulasi semua

**8. Tag `:latest` di produksi tidak disarankan karena...
- A. Tidak bisa dipull
- B. Menunjuk image yang bisa berubah → tidak reproducible
- C. Lebih besar ukurannya
- D. Hanya untuk amd64

**9. Compose `depends_on` dengan `condition: service_healthy` berbeda dari `depends_on` biasa karena...
- A. Hanya menunggu container start, bukan sehat
- B. Menunggu service dependency **sehat** (healthcheck pass) sebelum start
- C. Membuat service dependen
- D. Menghapus service lain

**10. Build image amd64 di Mac ARM64 (M5) cenderung lambat karena...
- A. Mac tidak bisa build amd64
- B. Memakai emulasi QEMU
- C. Registry lambat
- D. amd64 lebih besar

---

## Bagian B — Perintah/Dockerfile (4 soal)

**11.** Tulis perintah untuk: menjalankan container `alpine` dengan limit memori 256MB dan 1 CPU, background, nama `limited`.

**12.** Tulis baris Dockerfile untuk: membuat user non-root bernama `appuser` (UID 10001) di **Alpine**, lalu set agar container jalan sebagai user itu.

**13.** Tulis perintah `docker buildx` untuk: build image tag `registry.gitlab.com/me/proj/app:v2` untuk **arm64 dan amd64**, lalu langsung push ke registry.

**14.** Tulis entry `volumes` dan `healthcheck` untuk service `db` (postgres) di compose.yaml agar: data bertahan di volume `pgdata`, dan healthcheck pakai `pg_isready`.

---

## Bagian C — Skenario (2 soal)

**15.** Anda menarik image publik `oldapp:v1` ke Mac M5, jalankan, dan dapat `exec format error`. Jelaskan langkah diagnosa (apa yang Anda cek) dan dua kemungkinan akar masalah beserta solusinya.

**16.** Anda punya Dockerfile yang menghasilkan image 400MB. Sebutkan 4 teknik best practice untuk mengecilkan image itu, danjelaskan singkat kenapa masing-masing efektif.

---

## Bagian D — Troubleshooting (2 soal)

**17.** Compose stack app+postgres: app terus restart dengan log `ping err: connection refused` ke `db:5432`, padahal `docker compose ps` menunjukkan `db` running. Sebutkan 3 dugaan & cara ceknya.

**18.** Anda `docker push` ke GitLab registry dan dapat `denied: access forbidden`. Sebutkan 4 hal yang harus Anda verifikasi.

---

## Bagian E — Esai Singkat (2 soal)

**19.** Jelaskan koneksi konsep-konsep container (namespace, cgroup, volume, healthcheck) ke konsep Kubernetes yang setara (Pod, resource limits, PVC, probe). Mengapa belajar container dulu memudahkan masuk Kubernetes?

**20.** Anda akan deploy app ke production on-prem berbasis server **amd64**, sementara Anda develop di Mac **arm64**. Jelaskan strategi build & registry yang membuat image jalan di kedua arsitektur tanpa kejutan, dan di mana CI berperan (Fase 6).

---

## Kunci Jawaban

### A — Pilihan Ganda
1. **B** — container berbagi kernel host (tidak punya kernel sendiri)
2. **C** — writable layer hilang saat container dihapus (kecuali volume)
3. **B** — `clean` layer baru hanya menutupi; file masih di layer `install` → image tetap besar
4. **B** — `EXPOSE` dokumentasi; port ke host butuh `-p` di run
5. **B** — final stage hanya hasil (biner), toolchain build dibuang
6. **B** — limiting blast radius; celah tidak langsung jadi root
7. **B** — manifest list → pull otomatis pilih arch host
8. **B** — `:latest` mutable, tidak reproducible/traceable
9. **B** — `service_healthy` tunggu healthcheck pass (bukan sekadar start)
10. **B** — QEMU emulasi amd64 di ARM64 = lambat

### B — Perintah/Dockerfile
11. ```bash
    docker run -d --name limited --memory=256m --cpus=1.0 alpine sleep 3600
    ```
12. ```dockerfile
    RUN adduser -D -u 10001 appuser
    USER appuser
    ```
13. ```bash
    docker buildx build --platform linux/arm64,linux/amd64 \
      -t registry.gitlab.com/me/proj/app:v2 --push .
    ```
14. ```yaml
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
    # (dan volumes: pgdata: di bawah level services)
    ```

### C — Skenario
15. Diagnosa:
    1. Cek arsitektur image: `docker buildx imagetools inspect oldapp:v1 | grep linux/` (atau `docker inspect --format '{{.Architecture}}'`).
    2. Cek host arch: `uname -m` (Mac M5 = arm64).
    Dua kemungkinan akar:
    - **Image amd64-only** di host arm64 → `exec format error` (biner amd64 tidak bisa jalan native di arm64; jika pakai `--platform linux/amd64` akan emulasi QEMU lambat). Solusi: cari image multi-arch, atau build sendiri multi-arch, atau `docker run --platform linux/amd64` (emulasi, lambat).
    - **Image arm64 di host amd64** (jika target salah) → sama, `exec format error`. Solusi: build multi-arch manifest.
    Inti: mismatch arsitektur biner vs host kernel.

16. 4 teknik:
    1. **Multi-stage build** — build di stage besar, salin hanya biner ke stage final kecil → buang toolchain/compiler/source (paling efektif, ratusan MB → puluhan MB).
    2. **Base image kecil** (`alpine`/`distroless`/`-slim`) — bukan `ubuntu`/`golang` full → base lebih kecil dari awal.
    3. **Gabung `RUN` + bersih-bersih** (`RUN apt install && rm -rf /var/lib/apt/lists/*` di satu layer) — file dibuang di layer yang sama, tidak tertinggal di layer bawah.
    4. **`.dockerignore`** — jangan kirim `.git`, `target/`, `*.md`, `.env` ke build context → layer `COPY` lebih kecil & build cepat.
    (bonus) `-ldflags="-s -w"` strip simbol biner Go; `--no-install-recommends` apt.

### D — Troubleshooting
17. Dugaan + cek:
    - **App start sebelum DB siap menerima koneksi** (container running ≠ DB ready): `depends_on: condition: service_healthy` + healthcheck `pg_isready`. Cek `docker compose logs db` saat app connect.
    - **Network/nama service salah**: app connect ke `db` tapi service namanya `postgres` → `dial tcp: lookup db: no such host` atau `connection refused`. Cek nama service di compose.yaml & env `DATABASE_URL`.
    - **DB listen di localhost saja / auth salah**: cek `docker compose logs db` cari "database system is ready" & "authentication"; pastikan `POSTGRES_USER/PASSWORD/DB` cocok dengan connection string.
    - (bonus) **Port salah** di connection string (5432?); (bonus) DB restart loop sendiri → `docker compose ps` status db.

18. Verifikasi:
    1. **Login valid**: `docker login registry.gitlab.com` sukses (PAT scope `write_registry`, username = gitlab username bukan email).
    2. **Path benar**: `registry.gitlab.com/<username>/<project>/...` — project name case-sensitive, sesuai namespace GitLab.
    3. **Hak push ke project**: akun punya write access ke project itu (bukan hanya read).
    4. **Registry aktif di project**: GitLab project → Settings → General → visibility/registry; beberapa setup harus enable Container Registry.
    (bonus) tag tidak kosong; (bonus) rate/quota.

### E — Esai
19. Koneksi konsep:
    - **namespace** (isolasi proses/net) → **Pod**: satu atau lebih container berbagi namespace net (satu IP). Pod = container + shared namespace.
    - **cgroup** (limit CPU/mem) → **resources.limits/requests** di Pod; melampaui limit memory → `OOMKilled` (sama seperti cgroup OOM).
    - **volume** (data persisten) → **PVC/PV** (PersistentVolumeClaim); tanpa PVC, Pod restart = data hilang (sama seperti tanpa volume).
    - **healthcheck** → **liveness/readiness probe** Kubernetes; compose `healthcheck` ≈ probe.
    Mengapa memudahkan: Kubernetes tidak menciptakan ulang konsep ini — hanya mengorkestrasi banyak container dengan konsep yang sama. Paham "container = proses dibatasi cgroup" → langsung paham "Pod melebihi limit = OOMKilled". Paham volume compose → paham PVC. Belajar container dulu = belajar vocab Kubernetes tanpa orchestration yang membingungkan.

20. Strategi:
    - **Build multi-arch** dengan `docker buildx --platform linux/arm64,linux/amd64` + `--push` → satu tag (manifest list) melayani kedua arch; pull di masing-masing host otomatis dapat versi cocok. Tidak ada `exec format error` kejutan.
    - **Registry terpusat** (GitLab Container Registry) sebagai single source; bukan build lokal di tiap server.
    - **CI (GitLab CI, Fase 6)** sebagai tempat build multi-arch yang sebenarnya: build di runner multi-arch (atau amd64 native + arm64 native, atau QEMU) sehingga developer tidak perlu build amd64 lemot di Mac ARM64. CI juga memastikan setiap tag semver immutable & ter-scan.
    - **Tag semver + git SHA**, bukan `:latest`, agar reproducible & traceable (tag menunjuk build commit mana).
    Inti: developer build arm64 lokal (cepat), CI build & push multi-arch resmi; server amd64 & Mac arm64 pull tag sama tanpa masalah.

---

## Penilaian

| Benar | Skor |
|---|---|
| 18–20 | Expert — lanjut Modul 1.2 (OrbStack Lab Harian) |
| 16–17 | Lulus — boleh lanjut, perbaiki yang salah |
| 12–15 | Belum lulus — ulang materi, kerjakan ulang lab |
| < 12 | Ulangi semua materi, lanjut mentor |