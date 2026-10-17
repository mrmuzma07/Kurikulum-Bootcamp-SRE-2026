# LAB-02 — Membangun Service HTTP Go dengan Health Probe & Metrics

> **Target:** Membangun aplikasi microservice HTTP dalam bahasa **Go** di `m0.4/app-go` yang menyediakan endpoint `/`, `/healthz`, `/readyz`, dan `/metrics` (format Prometheus), dilengkapi unit test (`go test`), *graceful shutdown* (`SIGTERM`), serta berhasil di-*cross-compile* menjadi biner statis untuk `linux/arm64` dan `linux/amd64` sebagai fondasi langsung **Fase 1 (Containerization)**.

## Latar Belakang & Skenario Nyata
Di **Fase 1 (`modul-1.1-container-fundamental`)**, kamu akan membungkus sebuah aplikasi web ke dalam container *distroless* multi-arsitektur, men-deploy-nya ke **k3s (Fase 2)**, mem-package-nya dengan **Helm (Fase 5)**, dan memantaunya dengan **Prometheus & Grafana (Fase 7)**.

Agar sebuah aplikasi layak dijalankan di cluster Kubernetes produksi, SRE mensyaratkan kontrak operasional berikut pada aplikasi:
1. Konfigurasi port dan nama aplikasi dibaca dari *environment variable* (`PORT`, `APP_NAME`, `APP_ENV`).
2. Memiliki endpoint Liveness (`/healthz`) dan Readiness (`/readyz`).
3. Mengekspos metrik internal di `/metrics` tanpa perlu mengubah kode saat di-scrape Prometheus.
4. Menangani sinyal `SIGTERM` agar tidak memutus koneksi pengguna saat Pod di-rolling-update.

## Prasyarat
- [ ] Sudah membaca [04-go-microservice-dasar](../04-go-microservice-dasar.md)
- [ ] `LAB-01` selesai dikerjakan
- [ ] Compiler `go` sudah terpasang (`go version`)

## Estimasi Waktu
± 60–90 menit

---

## Langkah Pelaksanaan

### 1. Siapkan Direktori `m0.4/app-go` di Repo `sre-bootcamp`
```bash
cd ~/Developer/Playgrounds/devops/sre/training01/sre-bootcamp   # sesuaikan path repo
mkdir -p m0.4/app-go
cd m0.4/app-go

# Inisialisasi modul Go
go mod init gitlab.com/sre-bootcamp/app-go
```

Pastikan direktori hasil kompilasi biner (`dist/` atau `*.bin`) tidak ikut ter-commit ke Git:
```bash
grep -qxF 'dist/' ../../.gitignore 2>/dev/null || echo 'dist/' >> ../../.gitignore
```

*Verifikasi:*
```bash
cat go.mod
```

---

### 2. Buat Kode Utama `main.go`
Tulis kode `main.go` berikut:

```bash
cat > main.go <<'EOF'
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"sync/atomic"
	"syscall"
	"time"
)

var (
	httpRequestsTotal uint64
	readyState        atomic.Bool
	startedAt         = time.Now()
)

func envOrDefault(key, def string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return def
}

func setupRouter(appName, appEnv, version string) *http.ServeMux {
	mux := http.NewServeMux()

	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path != "/" {
			http.NotFound(w, r)
			return
		}
		atomic.AddUint64(&httpRequestsTotal, 1)
		w.Header().Set("Content-Type", "application/json")
		_ = json.NewEncoder(w).Encode(map[string]any{
			"service":    appName,
			"env":        appEnv,
			"version":    version,
			"uptime_sec": int(time.Since(startedAt).Seconds()),
		})
	})

	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte(`{"status":"ok"}` + "\n"))
	})

	mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		if !readyState.Load() {
			w.WriteHeader(http.StatusServiceUnavailable)
			_, _ = w.Write([]byte(`{"status":"not_ready"}` + "\n"))
			return
		}
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte(`{"status":"ready"}` + "\n"))
	})

	mux.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
		count := atomic.LoadUint64(&httpRequestsTotal)
		uptime := time.Since(startedAt).Seconds()
		w.Header().Set("Content-Type", "text/plain; version=0.0.4")
		fmt.Fprintf(w, "# HELP sre_app_requests_total Total HTTP requests pada root endpoint.\n")
		fmt.Fprintf(w, "# TYPE sre_app_requests_total counter\n")
		fmt.Fprintf(w, "sre_app_requests_total{service=%q,env=%q,version=%q} %d\n", appName, appEnv, version, count)
		fmt.Fprintf(w, "# HELP sre_app_uptime_seconds Waktu aktif service dalam detik.\n")
		fmt.Fprintf(w, "# TYPE sre_app_uptime_seconds gauge\n")
		fmt.Fprintf(w, "sre_app_uptime_seconds{service=%q} %.2f\n", appName, uptime)
	})

	return mux
}

func main() {
	port := envOrDefault("PORT", "8080")
	appName := envOrDefault("APP_NAME", "sre-bootcamp-api")
	appEnv := envOrDefault("APP_ENV", "development")
	version := envOrDefault("APP_VERSION", "0.1.0")

	readyState.Store(true)

	srv := &http.Server{
		Addr:         ":" + port,
		Handler:      setupRouter(appName, appEnv, version),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	go func() {
		fmt.Printf(`{"level":"info","msg":"starting http server","port":%q,"service":%q,"env":%q}`+"\n", port, appName, appEnv)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			fmt.Fprintf(os.Stderr, `{"level":"error","msg":"listen failed","error":%q}`+"\n", err.Error())
			os.Exit(1)
		}
	}()

	<-ctx.Done()
	readyState.Store(false)
	fmt.Println(`{"level":"info","msg":"SIGTERM/SIGINT received, draining connections..."}`)

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	_ = srv.Shutdown(shutdownCtx)
	fmt.Println(`{"level":"info","msg":"server stopped gracefully"}`)
}
EOF
```

---

### 3. Buat Unit Test `main_test.go` & Jalankan Pengujian
```bash
cat > main_test.go <<'EOF'
package main

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestEndpoints(t *testing.T) {
	readyState.Store(true)
	router := setupRouter("sre-api", "test", "1.0.0")

	tests := []struct {
		path       string
		wantStatus int
		wantBody   string
	}{
		{"/", http.StatusOK, `"service":"sre-api"`},
		{"/healthz", http.StatusOK, `"status":"ok"`},
		{"/readyz", http.StatusOK, `"status":"ready"`},
		{"/metrics", http.StatusOK, `sre_app_requests_total{service="sre-api",env="test",version="1.0.0"} 1`},
		{"/tidak-ada", http.StatusNotFound, "404"},
	}

	for _, tc := range tests {
		req := httptest.NewRequest(http.MethodGet, tc.path, nil)
		rec := httptest.NewRecorder()
		router.ServeHTTP(rec, req)

		if rec.Code != tc.wantStatus {
			t.Fatalf("GET %s: dapat status %d, ingin %d", tc.path, rec.Code, tc.wantStatus)
		}
		if !strings.Contains(rec.Body.String(), tc.wantBody) {
			t.Fatalf("GET %s: body %q tidak mengandung %q", tc.path, rec.Body.String(), tc.wantBody)
		}
	}
}
EOF
```

*Verifikasi Unit Test:*
```bash
go test -v ./...
```
Pastikan muncul `--- PASS: TestEndpoints` dan `PASS`.

---

### 4. Build Biner Statis Multi-Arch (`arm64` & `amd64`) dan Uji Runtime
```bash
mkdir -p dist

# Build biner untuk Linux ARM64 dan Linux AMD64
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -ldflags="-s -w" -o dist/sre-app-linux-arm64 .
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o dist/sre-app-linux-amd64 .

# Build biner untuk OS saat ini (untuk langsung diuji)
CGO_ENABLED=0 go build -ldflags="-s -w" -o dist/sre-app-local .

# Jalankan di background pada port 18082
PORT=18082 APP_NAME=sre-capstone APP_ENV=staging ./dist/sre-app-local &
APP_PID=$!
sleep 1
```

*Verifikasi Endpoint Runtime & Auditor Python (`LAB-01`):*
```bash
curl -s http://127.0.0.1:18082/ | jq .
curl -s http://127.0.0.1:18082/healthz
curl -s http://127.0.0.1:18082/readyz
curl -s http://127.0.0.1:18082/metrics

# Kirim sinyal SIGTERM dan amati log graceful shutdown
kill -TERM "$APP_PID"
wait "$APP_PID" 2>/dev/null || true
```

---

### 5. Commit & Merge ke `main` via Merge Request di GitLab
```bash
git add go.mod main.go main_test.go ../../.gitignore
git status
git commit -m "feat(m0.4): tambah service http go dengan healthz readyz dan metrics"
git push
```
Buka GitLab UI → buat **Merge Request** ke branch `main`, pastikan deskripsi MR jelas, lalu lakukan **Squash and Merge**.

---

## Acceptance Criteria (Kriteria Keberhasilan)
- [ ] `go test -v ./...` lulus 100% tanpa error.
- [ ] Perintah `file dist/sre-app-linux-arm64 dist/sre-app-linux-amd64` membuktikan kedua biner adalah *statically linked* untuk `ARM aarch64` dan `x86-64`.
- [ ] Endpoint `/`, `/healthz`, `/readyz`, dan `/metrics` merespons `HTTP 200` dengan payload yang sesuai.
- [ ] Ketika proses dikirimi `kill -TERM <PID>`, aplikasi mencetak log JSON `draining connections...` dan `server stopped gracefully`.
- [ ] Seluruh kode `m0.4/` (`audit-tool` dan `app-go`) sudah ter-merge ke branch `main` di GitLab tanpa mengikutsertakan `.venv/` atau `dist/`.

## Troubleshooting & Tips
- **Error `go: go.mod file not found`:** Pastikan kamu berada di direktori `m0.4/app-go` saat menjalankan `go test` atau `go build`.
- **Error `bind: address already in use`:** Ubah nilai variabel `PORT=18089` saat menjalankan `./dist/sre-app-local`.
