# LAB-02 — Compose Stack: App + Database di OrbStack

> **Target:** menjalankan stack multi-container (app web + database) dengan Docker Compose, lengkap dengan network, volume persisten, healthcheck, dan environment variable — miniatur arsitektur yang nanti jadi Deployment + StatefulSet di Kubernetes.

## Latar Belakang
Aplikasi nyata jarang berdiri sendiri: ada app + database, mungkin + cache. Compose mendeskripsikan seluruh stack dalam **satu file YAML** — `up` satu perintah, semuanya jalan. Ini model mental untuk Kubernetes manifest (beberapa Pod + Service + PVC). Di sini kita latih network, volume, healthcheck, dan env — konsep yang sama dipakai di k3s.

## Prasyarat
- [ ] LAB-01 selesai (image app di registry, `docker compose` tersedia via OrbStack)
- [ ] Sudah baca [01-konsep-container](../01-konsep-container.md)
- [ ] Repo `sre-bootcamp` + image app dari LAB-01

## Waktu
± 2 jam

## Langkah

### 1. Struktur Folder

```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
git switch -c m1.1-lab02
mkdir -p m1.1/compose && cd m1.1/compose
```

### 2. Aplikasi yang Bicara Database

Kita butuh app yang membaca/tulis DB. Buat versi sederhana yang catat "visit count" ke PostgreSQL.

```bash
mkdir app && cat > app/main.go <<'EOF'
package main

import (
	"database/sql"
	"fmt"
	"net/http"
	"os"

	_ "github.com/lib/pq"
)

func main() {
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		dbURL = "postgres://app:secret@db:5432/app?sslmode=disable"
	}
	db, err := sql.Open("postgres", dbURL)
	if err != nil {
		fmt.Fprintln(os.Stderr, "connect err:", err)
		os.Exit(1)
	}
	if err := db.Ping(); err != nil {
		fmt.Fprintln(os.Stderr, "ping err:", err)
		os.Exit(1)
	}
	_, _ = db.Exec(`CREATE TABLE IF NOT EXISTS visits (id serial PRIMARY KEY, at timestamp default now())`)

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		_, err := db.Exec("INSERT INTO visits DEFAULT VALUES")
		if err != nil {
			http.Error(w, err.Error(), 500)
			return
		}
		var n int
		_ = db.QueryRow("SELECT count(*) FROM visits").Scan(&n)
		fmt.Fprintf(w, "hello from %s — visit #%d\n", os.Getenv("APP_NAME"), n)
	})
	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		if err := db.Ping(); err != nil {
			http.Error(w, "unhealthy", 503)
			return
		}
		fmt.Fprintln(w, "ok")
	})
	fmt.Println("listening on :8080")
	http.ListenAndServe(":8080", nil)
}
EOF

cat > app/go.mod <<'EOF'
module compose/app
go 1.22
require github.com/lib/pq v1.10.9
EOF

cat > app/.dockerignore <<'EOF'
*.md
.env
EOF
```

### 3. Dockerfile App (Multi-Stage)

```bash
cat > app/Dockerfile <<'DOCKERFILE'
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /out/app .

FROM alpine:3.20
RUN adduser -D -u 10001 appuser
COPY --from=builder /out/app /app/app
ENV APP_NAME=compose-app PORT=8080
USER appuser
EXPOSE 8080
ENTRYPOINT ["/app/app"]
DOCKERFILE
```

### 4. `compose.yaml`

```bash
cat > compose.yaml <<'YAML'
# Compose v2 (OrbStack) — file bernama compose.yaml (standar baru) atau docker-compose.yml
services:
  app:
    build: ./app
    image: compose-app:local
    environment:
      APP_NAME: compose-app
      DATABASE_URL: postgres://app:secret@db:5432/app?sslmode=disable
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

volumes:
  db-data:
YAML
```

**Yang perlu diperhatikan:**
- **Network:** Compose otomatis buat network; service saling panggil via **nama service** (`db:5432`), bukan IP. Ini persis seperti Service DNS di Kubernetes.
- **`depends_on` + `condition: service_healthy`:** app tunggu DB sehat dulu (via healthcheck) sebelum start — hindari race condition "app start sebelum DB siap".
- **Volume `db-data`:** data DB bertahan saat `down`. Tanpa ini, data hilang tiap restart.
- **`restart: unless-stopped`:** auto-restart container kalau crash (kecuali dihentikan manual).

### 5. Jalankan Stack

```bash
# Build & start seluruh stack
docker compose up -d --build

# Lihat status (harus: app healthy, db healthy)
docker compose ps

# Lihat log gabungan
docker compose logs -f --tail=20
# Ctrl+C untuk keluar (tidak menghentikan stack)

# Uji endpoint
curl http://localhost:8080/             # "hello from compose-app — visit #1"
curl http://localhost:8080/             # visit #2 (counter naik)
curl http://localhost:8080/health       # ok

# Buktikan data persisten di volume
docker compose down                     # hentikan & hapus container (volume tetap)
docker compose up -d
curl http://localhost:8080/             # visit #3 (counter lanjut, tidak reset)
```

### 6. Inspeksi Network & Volume

```bash
# Network yang dibuat Compose (service resolve via nama):
docker network ls | grep compose
docker inspect <project>_default | grep -A10 Containers

# Volume:
docker volume ls | grep db-data
docker volume inspect <project>_db-data
# Data disimpan di OrbStack volume; bertahan lintas down/up

# Buktikan app → db via nama service:
docker compose exec app sh -c 'echo > /dev/tcp/db/5432 && echo "db reachable" || echo "unreachable"'
# (distroless tanpa shell — di lab ini pakai alpine, sh tersedia)
```

### 7. Simulasi Kegagalan & Restart

```bash
# Bunuh DB, amati app (harus restart/unhealthy)
docker compose stop db
curl --max-time 5 http://localhost:8080/health    # 503 atau error (DB hilang)
sleep 6
docker compose ps                                  # dbExited, app mungkin restart loop

# Pulihkan
docker compose start db
sleep 6
curl http://localhost:8080/health                 # ok lagi
```

Ini miniatur "Pod restart karena dependency mati" — di Kubernetes akan jadi CrashLoopBackOff (Modul 2.4).

### 8. Bersihkan & Commit

```bash
# Hentikan & hapus container + network (volume tetap, simpan data)
docker compose down

# Hapus volume juga bila ingin bersih total (DATA HILANG):
# docker compose down -v

# Commit ke repo
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp
git add m1.1/compose/
git commit -m "feat(m1.1): compose stack app + postgres

- app web catat visit count ke postgres
- network otomatis (resolve via nama service)
- volume db-data persisten
- healthcheck pg_isready + depends_on condition service_healthy
- simulasi kegagalan DB & restart

Closes #<issue-m1.1-lab02>"
git push -u origin m1.1-lab02
```

Buat MR → squash & merge.

## Acceptance Criteria

- [ ] `docker compose up -d --build` berhasil tanpa error
- [ ] `docker compose ps` menunjukkan app & db **healthy**
- [ ] `curl http://localhost:8080/` mengembalikan visit counter yang **meningkat**
- [ ] `curl /health` balas `ok`
- [ ] Data visit counter **bertahan** setelah `docker compose down` + `up` (volume bekerja)
- [ ] App resolve DB via **nama service** `db` (bukan IP)
- [ ] Simulasi `docker compose stop db` → `/health` 503, lalu pulih setelah `start db`
- [ ] `compose.yaml` ter-commit di repo via MR

## Troubleshooting

| Gejala | Solusi |
|---|---|
| App crash loop "ping err" | DB belum siap; `depends_on condition: service_healthy` atau cek `docker compose logs db` |
| `dial tcp: lookup db: no such host` | Network Compose tidak jalan; pastikan service di stack yang sama; cek `docker network ls` |
| Data reset setelah restart | Volume tidak dipakai; pastikan `volumes: - db-data:/var/lib/postgresql/data` ada & `down` tanpa `-v` |
| `port already in use 8080` | Container lain pakai 8080: `docker ps`, stop; atau ubah mapping `18080:8080` |
| healthcheck `pg_isready` never healthy | Password/user/db salah; cek env DB sama dengan `POSTGRES_*` |
| `compose` command not found | OrbStack/Docker Compose v2; `docker compose` (spasi) bukan `docker-compose` (lama) |
| App start terlalu cepat sebelum DB | `depends_on` tanpa `condition` hanya tunggu container start, bukan siap — pakai `service_healthy` |

## Catatan SRE
- **Network via nama service** = persis Service DNS Kubernetes. `db:5432` di sini = `db-service:5432` di K8s.
- **Volume** = PVC di Kubernetes. Tanpa itu, Pod restart = data hilang.
- **Healthcheck** = readiness/liveness probe di Kubernetes (Modul 2.4).
- **depends_on healthcheck** = init container / Pod startup ordering di K8s.
- Compose bagus untuk dev; production pakai Kubernetes (Fase 2). Tapi YAML compose = jembatan mental ke manifest K8s.

## Lanjut
[Evaluasi: Latihan & Kuis](../evaluasi/latihan.md)