# 04 — Dasar Go (Golang) untuk SRE: Microservice, Probe, & Biner Multi-Arch

> **Ringkasan:** Memahami anatomi aplikasi **Go (Golang)** dari kacamata SRE: membaca konfigurasi lewat *environment variable*, menyediakan endpoint standar produksi (`/healthz`, `/readyz`, `/metrics`), menangani sinyal *graceful shutdown* (`SIGTERM`), menjalankan unit test (`go test`), serta mengompilasi biner statis lintas arsitektur (`arm64` dan `amd64`).

---

## 1. Konsep & Arsitektur: Mengapa Go Adalah "Bahasa Ibu" Cloud-Native?

Jika kamu membuka kode sumber **Docker, Kubernetes (k3s), OpenTofu, Helm, ArgoCD, Prometheus, Grafana Alloy, Mimir, Loki, atau Tempo**, semuanya ditulis menggunakan bahasa **Go**.

Bagi seorang SRE, kamu tidak dituntut menghafal seluruh fitur bahasa Go seperti seorang *backend engineer* penuh waktu. Namun, memahami cara kerja aplikasi Go memberikan 4 keuntungan besar di sepanjang bootcamp ini:
1. **Satu Biner Mandiri (*Static Binary*):** Dengan `CGO_ENABLED=0`, hasil `go build` adalah **satu file biner tunggal** yang sudah memuat semua kebutuhannya. Di Fase 1, kita bisa memasukkannya ke dalam container `distroless` berukuran hanya ~5–15 MB (bandingkan dengan image runtime bahasa lain yang bisa mencapai 150–800 MB).
2. **Cross-Compilation Instan (`arm64` ↔ `amd64`):** Dari MacBook Apple Silicon (`arm64`), kamu bisa mem-build biner Linux untuk server produksi `x86_64` (`amd64`) hanya dengan mengubah variabel `GOOS=linux GOARCH=amd64`.
3. **Standar Observabilitas & Siklus Hidup Pod Kubernetes:** Kamu bisa melihat secara transparan bagaimana sebuah aplikasi mengekspos `/healthz` (Liveness), `/readyz` (Readiness), `/metrics` (Prometheus format), dan menangani `SIGTERM` saat Pod di-restart oleh Kubernetes.
4. **Dasar Sintaks Template Helm (Fase 5):** Sintaks `{{ .Values.replicaCount }}` pada Helm Chart adalah paket `text/template` standar dari bahasa Go.

```text
Kode Sumber (main.go + go.mod)
        │
        ├──► go test ./...               (Unit Test di GitLab CI - Fase 6)
        │
        └──► CGO_ENABLED=0 go build      (Static Binary ~6 MB)
                   │
                   ├──► Endpoint /healthz & /readyz  (K8s Probes - Fase 2)
                   ├──► Endpoint /metrics            (Prometheus Scrape - Fase 7)
                   └──► Signal Handler SIGTERM       (Zero-Downtime Rollout - Fase 2 & 6)
```

---

## 2. Parameter Kunci & Anatomi Aplikasi Go Siap Produksi

| Konsep Go / SRE | Implementasi di Kode Go | Peran di Lingkungan Produksi (k3s) |
|---|---|---|
| `os.Getenv("PORT")` | Membaca konfigurasi dari *environment variable* | Memungkinkan 1 container image yang sama dipakai di `dev`, `staging`, dan `prod` lewat ConfigMap/Secret Kubernetes. |
| `/healthz` (*Liveness*) | Mengembalikan `HTTP 200 OK` selama proses tidak *deadlock* | Dipakai `livenessProbe` Kubernetes: jika gagal, k3s akan **me-restart Pod**. |
| `/readyz` (*Readiness*) | Mengembalikan `HTTP 200 OK` jika siap terima trafik (atau `503` jika belum siap/sedang shutdown) | Dipakai `readinessProbe` Kubernetes: jika gagal, Pod **dikeluarkan sementara dari LoadBalancer/Service** tanpa di-restart. |
| `/metrics` | Mengekspos penghitung (*counter*) teks berformat Prometheus | Di-scrape oleh Prometheus / Grafana Alloy setiap 15 detik di Fase 7. |
| `SIGTERM` & `server.Shutdown` | Menangkap sinyal mati dari OS/Kubernetes dan menyelesaikan request yang sedang berjalan | Mencegah error `502 Bad Gateway` pada pengguna saat terjadi *rolling update* Deployment. |

---

## 3. Hands-On Step-by-Step

Mari kita bangun aplikasi mikro Go pertama kita dari nol di terminal VM `lab01` (atau macOS).

### 3.1 Inisialisasi Modul Go (`go.mod`)
Setiap proyek Go dimulai dengan file `go.mod` yang mencatat nama modul dan versi Go:

```bash
mkdir -p ~/latihan-m0.4/go-dasar && cd ~/latihan-m0.4/go-dasar

# Inisialisasi modul Go
go mod init sre-bootcamp/demo-app
cat go.mod
```

### 3.2 Menulis Aplikasi HTTP Siap Produksi (`main.go`)
Buat file `main.go` yang memiliki 4 endpoint (`/`, `/healthz`, `/readyz`, `/metrics`), pencatatan log terstruktur (JSON), dan *graceful shutdown*:

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
	reqCount uint64
	isReady  atomic.Bool
	startAt  = time.Now()
)

func getenv(key, fallback string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return fallback
}

func newMux(appName, version string) *http.ServeMux {
	mux := http.NewServeMux()

	// 1. Endpoint utama aplikasi
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		atomic.AddUint64(&reqCount, 1)
		w.Header().Set("Content-Type", "application/json")
		resp := map[string]any{
			"app":        appName,
			"version":    version,
			"uptime_sec": int(time.Since(startAt).Seconds()),
		}
		_ = json.NewEncoder(w).Encode(resp)
	})

	// 2. Liveness Probe: apakah proses masih hidup?
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "ok")
	})

	// 3. Readiness Probe: apakah siap menerima trafik dari Service/Ingress?
	mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
		if !isReady.Load() {
			w.WriteHeader(http.StatusServiceUnavailable)
			fmt.Fprintln(w, "not ready")
			return
		}
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "ready")
	})

	// 4. Prometheus Metrics Endpoint (format teks standar Prometheus)
	mux.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
		total := atomic.LoadUint64(&reqCount)
		uptime := time.Since(startAt).Seconds()
		w.Header().Set("Content-Type", "text/plain; version=0.0.4")
		fmt.Fprintf(w, "# HELP app_http_requests_total Total HTTP requests ke endpoint utama.\n")
		fmt.Fprintf(w, "# TYPE app_http_requests_total counter\n")
		fmt.Fprintf(w, "app_http_requests_total{app=%q,version=%q} %d\n", appName, version, total)
		fmt.Fprintf(w, "# HELP app_uptime_seconds Waktu hidup aplikasi dalam detik.\n")
		fmt.Fprintf(w, "# TYPE app_uptime_seconds gauge\n")
		fmt.Fprintf(w, "app_uptime_seconds{app=%q} %.2f\n", appName, uptime)
	})

	return mux
}

func main() {
	port := getenv("PORT", "8080")
	appName := getenv("APP_NAME", "sre-demo-app")
	version := getenv("APP_VERSION", "0.1.0")

	isReady.Store(true)

	srv := &http.Server{
		Addr:         ":" + port,
		Handler:      newMux(appName, version),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	// Tangkap sinyal SIGINT (Ctrl+C) dan SIGTERM (saat Kubernetes mematikan Pod)
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	go func() {
		fmt.Printf(`{"level":"info","msg":"server listening","port":%q,"app":%q,"version":%q}`+"\n", port, appName, version)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			fmt.Fprintf(os.Stderr, `{"level":"error","msg":"server failed","err":%q}`+"\n", err.Error())
			os.Exit(1)
		}
	}()

	// Tunggu sampai ada sinyal berhenti
	<-ctx.Done()
	isReady.Store(false)
	fmt.Println(`{"level":"info","msg":"shutting down gracefully..."}`)

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	_ = srv.Shutdown(shutdownCtx)
	fmt.Println(`{"level":"info","msg":"server stopped cleanly"}`)
}
EOF
```

### 3.3 Menulis Unit Test Bawaan Go (`main_test.go`)
Di Fase 6 (GitLab CI), setiap perubahan kode harus melewati tahap pengujian otomatis (`go test`). Mari tambahkan file `main_test.go`:

```bash
cat > main_test.go <<'EOF'
package main

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestHealthAndMetrics(t *testing.T) {
	isReady.Store(true)
	mux := newMux("test-app", "1.0.0")

	// 1. Uji /healthz harus 200 OK
	req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
	rec := httptest.NewRecorder()
	mux.ServeHTTP(rec, req)
	if rec.Code != http.StatusOK {
		t.Fatalf("ekspektasi status 200 di /healthz, dapat %d", rec.Code)
	}

	// 2. Uji / lalu pastikan counter di /metrics bertambah
	mux.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest(http.MethodGet, "/", nil))

	reqMetrics := httptest.NewRequest(http.MethodGet, "/metrics", nil)
	recMetrics := httptest.NewRecorder()
	mux.ServeHTTP(recMetrics, reqMetrics)
	body := recMetrics.Body.String()
	if !strings.Contains(body, `app_http_requests_total{app="test-app",version="1.0.0"}`) {
		t.Fatalf("output /metrics tidak mengandung metrik yang diharapkan:\n%s", body)
	}
}
EOF

# Jalankan unit test
go test -v ./...
```
*Output yang diharapkan:*
```text
=== RUN   TestHealthAndMetrics
--- PASS: TestHealthAndMetrics (0.00s)
PASS
ok  	sre-bootcamp/demo-app	0.004s
```

### 3.4 Cross-Compile Multi-Architecture (`arm64` vs `amd64`)
Sesuai prinsip **Multi-Architecture Awareness** di bootcamp ini (laptop Mac M5 adalah `arm64`, sedangkan server bare-metal produksi umumnya `amd64`), mari kita buktikan betapa mudahnya Go mengompilasi biner untuk kedua arsitektur tersebut:

```bash
mkdir -p dist

# 1. Build biner statis untuk Linux ARM64 (OrbStack / Apple Silicon VM)
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -ldflags="-s -w" -o dist/app-linux-arm64 .

# 2. Build biner statis untuk Linux AMD64 (Server Production x86_64)
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o dist/app-linux-amd64 .

# 3. Periksa ukuran dan tipe arsitektur kedua file biner
ls -lh dist/
file dist/app-linux-arm64 dist/app-linux-amd64
```
*Output yang diharapkan:*
```text
dist/app-linux-arm64: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, stripped
dist/app-linux-amd64: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```
Perhatikan kata **`statically linked, stripped`**:
- `CGO_ENABLED=0` memastikan biner **tidak** bergantung pada library C (`glibc`) milik OS host, sehingga langsung bisa jalan di container kosong (`scratch` atau `distroless`).
- `-ldflags="-s -w"` membuang tabel simbol debug sehingga ukuran biner menyusut drastis (~30% lebih kecil).

---

## 4. Pola Kegagalan (Failure Mode) & Diagnosa

1. **`standard_init_linux.go: exec user process caused: no such file or directory` di Container Alpine/Distroless:**
   - **Penyebab:** Kamu mem-build biner Go **tanpa** `CGO_ENABLED=0` di Ubuntu (yang memakai `glibc`), lalu menyalin binernya ke container `alpine` atau `distroless/static` yang tidak memiliki `glibc`.
   - **Diagnosa:** Jalankan `file <nama-biner>` atau `ldd <nama-biner>`. Jika tertulis `dynamically linked`, berarti `CGO_ENABLED=0` terlupa.
2. **`exec format error` Saat Container / Biner Dijalankan:**
   - **Penyebab:** Arsitektur biner (`GOARCH=amd64`) tidak cocok dengan arsitektur CPU mesin (`arm64`) atau sebaliknya.
   - **Diagnosa:** Cek dengan `uname -m` di mesin target dan `file <nama-biner>`.
3. **Request Pengguna Putus (`502 Bad Gateway`) Saat Pod Di-restart:**
   - **Penyebab:** Aplikasi tidak menangani sinyal `SIGTERM` dan tidak mengubah status `/readyz` menjadi `503` saat mulai berhenti.
   - **Perbaikan:** Terapkan pola `signal.NotifyContext(..., syscall.SIGTERM)` dan `srv.Shutdown(ctx)` seperti pada contoh di atas.

---

## 5. Ringkasan & Langkah Selanjutnya

- Aplikasi yang ramah SRE selalu membaca konfigurasi dari *environment variable*, mengekspos `/healthz`, `/readyz`, dan `/metrics`, serta berhenti dengan rapi saat menerima `SIGTERM`.
- Selalu gunakan `CGO_ENABLED=0` saat mem-build biner Go untuk container produksi.
- Sekarang saatnya menguji kemampuan praktikmu di dua skenario laboratorium nyata:
  1. [LAB-01 — Membuat CLI Auditor YAML & HTTP Endpoint dengan Python](lab/LAB-01-script-audit-api.md)
  2. [LAB-02 — Membangun Service HTTP Go Siap Containerisasi](lab/LAB-02-http-service-health.md)
