# Kurikulum Bootcamp SRE 2026
### Dari Pemula → Siap Pegang Environment Production (On-Prem)

| Item | Keterangan |
|---|---|
| Durasi | 18 minggu (~4,5 bulan), ~15–20 jam/minggu |
| Level awal | Pemula (boleh tanpa background IT) |
| Perangkat latihan | MacBook Air M5 (Apple Silicon / ARM64) |
| Target akhir | Paham, lancar, dan berani "pegang" production on-prem |

**Stack yang dipelajari:** OrbStack · k3s/k3d · MetalLB · OpenTofu · Ansible · Helm · ArgoCD · GitLab.com · Prometheus · Alloy · Grafana · Mimir · Loki · Tempo

---

## 1. Profil Lulusan (Learning Outcomes)

Setelah lulus, peserta mampu:

1. Mengoperasikan Linux & container secara mandiri (troubleshoot tanpa panik).
2. Membangun dan mengelola cluster Kubernetes (k3s) untuk simulasi dan production on-prem, termasuk networking bare-metal dengan MetalLB.
3. Menyediakan infrastruktur dengan OpenTofu (IaC) dan mengelola state dengan aman.
4. Mengotomasi konfigurasi server dengan Ansible (patching, hardening, install k3s) — keterampilan kunci di lingkungan on-prem.
5. Menerapkan GitOps end-to-end: GitLab → CI → Helm → ArgoCD → cluster.
6. Membangun observability penuh: metrics (Prometheus → Mimir), logs (Loki via Alloy), traces (Tempo), dashboard & alerting (Grafana + Alertmanager).
7. Menjalankan praktik SRE: SLO/error budget, on-call, incident response, postmortem.
8. Memahami karakteristik khusus on-prem: jaringan fisik, LoadBalancer bare-metal, storage, backup/restore, keamanan dasar.

---

## 2. Peta Alat & Perannya di Production

| Alat | Peran di Production | Catatan untuk Mac M5 (ARM64) |
|---|---|---|
| OrbStack | Runtime Linux/container lokal (pengganti Docker Desktop, ringan di Apple Silicon) | Native ARM64, sangat hemat RAM |
| k3d | Cluster Kubernetes-in-Docker untuk latihan cepat & CI | Jalan di atas OrbStack |
| k3s | Distro Kubernetes ringan untuk production on-prem (single node atau HA) | Image & binary ARM64 tersedia |
| MetalLB | LoadBalancer bare-metal (L2/BGP) — pengganti cloud LB di on-prem | - |
| OpenTofu | IaC (fork open-source Terraform) untuk provisioning server/VM/infra | Binary universal macOS |
| Ansible | Konfigurasi server, patching, hardening, instalasi k3s (agentless via SSH) | Install via pip/brew di Mac |
| Helm | Package manager Kubernetes | - |
| ArgoCD | GitOps CD — source of truth ada di Git | - |
| GitLab.com | Repo + CI/CD pipeline + issue tracking | Free tier cukup untuk belajar |
| Prometheus | Standar de-facto metrics: scraping, PromQL, alerting rules | - |
| Alloy | Collector telemetry (metrics/logs/traces) — pengganti Promtail/Grafana Agent | - |
| Grafana | Visualisasi & alerting | - |
| Mimir | Metrics storage skala besar (long-term Prometheus) | - |
| Loki | Log aggregation (pasangan Alloy untuk logs) | - |
| Tempo | Distributed tracing backend | - |

**Alur alat di production on-prem:**
```
OpenTofu (provision VM/server) → Ansible (konfigurasi OS + install k3s)
  → Helm + ArgoCD (deploy aplikasi, GitOps) → Prometheus/Alloy (telemetry)
  → Mimir/Loki/Tempo (storage) → Grafana (visualisasi & alert)
```

---

## 3. Struktur Kurikulum — Ringkasan

| Fase | Minggu | Modul |
|---|---|---|
| 0 | 1–2 | Fondasi: Linux, Networking, Git |
| 1 | 3 | Container & OrbStack |
| 2 | 4–5 | Kubernetes dengan k3d & k3s + MetalLB |
| 3 | 6–7 | Infrastructure as Code dengan OpenTofu |
| 4 | 8 | Konfigurasi Server dengan Ansible |
| 5 | 9 | Helm |
| 6 | 10–11 | GitOps: GitLab CI + ArgoCD |
| 7 | 12–14 | Observability: Prometheus, Alloy, Grafana, Mimir, Loki, Tempo |
| 8 | 15 | Praktik SRE & Kesiapan Production On-Prem |
| 9 | 16–18 | Capstone Project + Simulasi On-Call |

---

## 4. Detail Modul

### FASE 0 — Fondasi (Minggu 1–2)

#### Modul 0.1 — Linux & Shell (5 hari)
**Tujuan:** Nyaman hidup di terminal.

- Navigasi filesystem, permission (`chmod`, `chown`), user/group
- Manajemen proses: `ps`, `top`/`htop`, `kill`, systemd (`systemctl`, `journalctl`)
- Text processing: `grep`, `awk`, `sed`, `jq`
- Networking CLI: `curl`, `dig`, `ss`, `ping`, `ip`
- SSH: key-based auth, `~/.ssh/config`, tunneling dasar
- Bash scripting dasar + buat alias & dotfiles sendiri

**Lab:**
- [ ] Install OrbStack, buat VM Linux (Ubuntu), akses via SSH
- [ ] Tulis script backup direktori + cron/systemd timer
- [ ] Debug proses yang memakan CPU tinggi (skenario dibuat sendiri)

#### Modul 0.2 — Networking untuk SRE (3 hari)
- DNS (record A/CNAME/SRV, resolusi, `dig`), CIDR & subnetting dasar
- TCP/UDP, port, handshake, TLS secara konsep
- HTTP/HTTPS: status code, header, proxy/reverse proxy
- Firewall dasar (nftables/ufw), NAT, port forwarding
- **Pengantar ARP & routing** (bekal penting untuk MetalLB L2 mode)

**Lab:**
- [ ] Konfigurasi reverse proxy (Caddy/Nginx) di VM OrbStack
- [ ] Simulasi DNS lokal (`/etc/hosts` + dnsmasq)
- [ ] Trace masalah koneksi dengan `curl -v`, `ss -tulpn`

#### Modul 0.3 — Git & Kolaborasi (2 hari)
- Repo, branch, merge, rebase, conflict resolution
- Conventional commit, branch strategy (trunk-based)
- GitLab: project, MR (merge request), issue, milestone

**Lab:**
- [ ] Buat repo `sre-bootcamp` di gitlab.com, kerjakan semua lab via MR ke diri sendiri

---

### FASE 1 — Container & OrbStack (Minggu 3)

#### Modul 1.1 — Container Fundamental
- Konsep: image vs container, layer, namespace & cgroup (konsep)
- Dockerfile best practice: multi-stage build, non-root user, image kecil
- Registry: push/pull ke GitLab Container Registry
- **Catatan ARM64:** build multi-arch (`buildx`), waspadai image amd64-only di Mac M5

#### Modul 1.2 — OrbStack sebagai Lab Harian
- Machine Linux di OrbStack, filesystem sharing, networking mode
- Resource limit, perbandingan dengan Docker Desktop
- Menjalankan k3d di atas OrbStack

**Lab:**
- [ ] Containerisasi aplikasi web sederhana (Go/Python/Node) — multi-stage
- [ ] Push image multi-arch ke GitLab Container Registry
- [ ] Jalankan compose stack (app + database) di OrbStack

**Deliverable:** Aplikasi berjalan di container, image multi-arch di registry.

---

### FASE 2 — Kubernetes: k3d, k3s & MetalLB (Minggu 4–5)

#### Modul 2.1 — Konsep & k3d untuk Latihan
- Arsitektur: control plane, kubelet, container runtime, etcd
- Objek inti: Pod, Deployment, Service, Ingress, ConfigMap, Secret, PV/PVC
- k3d: buat/hapus cluster cepat, multi-node, port mapping
- `kubectl` survival kit: get, describe, logs, exec, port-forward, top

#### Modul 2.2 — k3s untuk Simulasi Production
- Install k3s single-node & multi-node (di VM OrbStack)
- Traefik bawaan vs ingress alternatif
- Disable komponen yang tidak perlu (`--disable traefik`, `--disable servicelb`)
- Simulasi topologi on-prem: static IP, external LoadBalancer

#### Modul 2.3 — MetalLB: LoadBalancer Bare-Metal (Pendalaman)
- **Kenapa butuh MetalLB:** di cloud, `Service type=LoadBalancer` otomatis dapat IP dari provider; di on-prem tidak ada — MetalLB mengisi kekosongan ini
- **L2 mode (advertisement via ARP/NDP):** cara kerja, IPAddressPool + L2Advertisement, cocok untuk lab & jaringan sederhana
- **BGP mode (konsep):** peering dengan router, kapan dipakai di production nyata
- Konfigurasi: pilih range IP yang tidak bentrok dengan DHCP/static IP lain
- Integrasi dengan k3s: disable ServiceLB (klipper) dulu, baru pasang MetalLB
- Troubleshooting umum: IP tidak ter-assign, ARP conflict, traffic tidak sampai (cek `arping`, firewall, strict ARP)

#### Modul 2.4 — Operasi & Troubleshooting
- Debug pola: CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled
- Resource requests/limits, HPA
- Backup & restore etcd pada k3s (snapshot)
- Upgrade k3s tanpa downtime (konsep rolling)

**Lab:**
- [ ] Cluster k3d 3 node + deploy app dengan Ingress
- [ ] Cluster k3s multi-node di VM OrbStack, disable servicelb, pasang MetalLB (L2 mode)
- [ ] Expose app via `Service type=LoadBalancer` → akses dari Mac lewat IP MetalLB
- [ ] Skenario chaos: matikan 1 node, amati perilaku app
- [ ] Backup & restore snapshot etcd k3s

**Deliverable:** Cluster k3s multi-node "mirip production" dengan LoadBalancer berfungsi.

---

### FASE 3 — Infrastructure as Code: OpenTofu (Minggu 6–7)

#### Modul 3.1 — Dasar OpenTofu
- Kenapa IaC; OpenTofu vs Terraform (lisensi, komunitas)
- HCL: resource, variable, output, provider, state
- Workflow: `init → plan → apply → destroy`
- State: local vs remote (S3-compatible/MinIO), locking, `state mv/rm/import`

#### Modul 3.2 — Modul & Pola Produksi
- Struktur modul reusable (modules/, environments/)
- Workspace vs direktori per environment
- `for_each`, `count`, conditional, data source
- Secret handling: jangan commit secret; integrasi dengan vault/SOPS (konsep)

#### Modul 3.3 — Konteks On-Prem
- Provider yang relevan on-prem: Proxmox, vSphere, libvirt, CloudInit, atau HTTP/REST API internal
- Simulasi lokal: pakai provider `docker`/`libvirt`/mock untuk latihan
- **Pola produksi on-prem:** OpenTofu provisioning VM/server → Ansible bootstrap & konfigurasi → install k3s (handoff ini dilatih nyata di Fase 4)

**Lab:**
- [ ] Tulis modul reusable "web server" + deploy via OpenTofu (provider docker/local)
- [ ] Remote state ke MinIO (S3-compatible) berjalan di OrbStack
- [ ] Import resource yang dibuat manual ke dalam state
- [ ] Simulasi drift detection & remediasi

**Deliverable:** Repo IaC dengan modul rapi + state remote + pipeline plan otomatis (lihat Fase 6).

---

### FASE 4 — Konfigurasi Server dengan Ansible (Minggu 8)

> **Kenapa Ansible penting untuk on-prem:** di production on-prem, Anda mengelola banyak server fisik/VM tanpa API cloud. Ansible (agentless, via SSH) adalah standar untuk konfigurasi konsisten, patching, hardening, dan instalasi Kubernetes.

#### Modul 4.1 — Fundamental Ansible
- Arsitektur agentless: control node → SSH → managed nodes
- Inventory (static & dynamic), groups, variables, `ansible.cfg`
- Ad-hoc commands: `ansible all -m ping`, `-m apt`, `-m systemd`
- Playbook YAML: tasks, handlers, conditionals (`when`), loops, tags
- Idempotency: konsep paling penting — playbook dijalankan berulang, hasil tetap sama

#### Modul 4.2 — Pola Produksi
- Roles & collections (Ansible Galaxy): struktur role yang rapi
- Vault: enkripsi secret (password, key) dalam repo — `ansible-vault`
- Jinja2 templating untuk file konfigurasi (`template` module)
- Error handling, `--check` (dry-run), `--diff`, `--limit`
- Menjalankan Ansible dari GitLab CI (runner sebagai control node)

#### Modul 4.3 — Ansible untuk Lingkungan k3s/On-Prem
- Hardening server dasar: user non-root, SSH hardening, firewall (ufw), fail2ban, auto security updates
- Install & upgrade k3s via Ansible (role komunitas `xanmanning.k3s` atau buat sendiri)
- Patching OS terjadwal & rolling (satu per satu node agar cluster tetap sehat)
- Backup konfigurasi: seluruh playbook + inventory di Git (infra = code)

**Lab:**
- [ ] Setup inventory 3 VM OrbStack; jalankan ad-hoc & playbook pertama
- [ ] Tulis role `common` (hardening + paket dasar) — idempoten, jalankan 2x tanpa perubahan
- [ ] Install cluster k3s multi-node full via Ansible (server + agents)
- [ ] Simulasi patching rolling: update paket OS satu node per satu
- [ ] Enkripsi secret dengan ansible-vault

**Deliverable:** Repo Ansible (roles + inventory + vault) yang bisa membangun cluster k3s dari nol — "resep" rebuild production.

---

### FASE 5 — Helm (Minggu 9)

#### Modul 5.1 — Helm Fundamental
- Chart structure, values.yaml, template & fungsi Go-template
- `install/upgrade/rollback/uninstall`, `--set`, multiple values files
- Repository chart & OCI chart di GitLab registry

#### Modul 5.2 — Chart untuk Production
- Buat chart sendiri untuk aplikasi internal
- Pattern: values per environment (`values-staging.yaml`, `values-prod.yaml`)
- Hooks, tests, NOTES.txt
- Strategi upgrade & rollback yang aman

**Lab:**
- [ ] Deploy stack observability via Helm (persiapan Fase 7)
- [ ] Buat chart custom untuk app sendiri, versi semver, push ke GitLab OCI registry
- [ ] Uji `helm test` dan rollback

**Deliverable:** Chart custom versi 1.0.0 di registry, siap dipakai ArgoCD.

---

### FASE 6 — GitOps: GitLab CI + ArgoCD (Minggu 10–11)

#### Modul 6.1 — GitLab CI/CD
- `.gitlab-ci.yml`: stages, jobs, artifacts, cache, rules
- Runner: shared runner gitlab.com + self-hosted runner di OrbStack (simulasi on-prem runner)
- Pipeline: lint → test → build image → push registry
- Pipeline IaC: `tofu plan` di MR, `apply` di main (dengan approval)
- Pipeline Ansible: lint (`ansible-lint`) + jalankan playbook ke lab
- Protected branch & environment

#### Modul 6.2 — ArgoCD
- Konsep GitOps: Git = single source of truth, pull-based deployment
- Install ArgoCD (via Helm) ke cluster k3s
- Application & ApplicationSet, sync policy, self-heal, auto vs manual sync
- Struktur repo GitOps (pisah app repo & manifest repo)
- Secret management: Sealed Secrets / SOPS + age

#### Modul 6.3 — End-to-End Flow
```
Developer push → GitLab CI (build+push image) → update tag di GitOps repo
   → ArgoCD detect drift → sync ke cluster k3s → app ter-update
```
- Progressive delivery konsep: canary/blue-green (pengenalan Argo Rollouts)

**Lab:**
- [ ] Pipeline CI lengkap untuk app sendiri (build & push otomatis)
- [ ] Pipeline OpenTofu (plan di MR) & Ansible (lint + run) di GitLab CI
- [ ] ArgoCD mengelola semua deployment di cluster (termasuk ArgoCD mengelola dirinya sendiri)
- [ ] Simulasi drift manual (`kubectl edit`) → lihat ArgoCD self-heal
- [ ] Deploy via ApplicationSet ke "staging" dan "prod" (2 cluster k3d/k3s)

**Deliverable:** Sistem GitOps penuh yang berjalan otomatis.

---

### FASE 7 — Observability: Prometheus · Alloy · Grafana · Mimir · Loki · Tempo (Minggu 12–14)

#### Modul 7.1 — Fondasi Metrics: Prometheus (Minggu 12)
> **Kenapa mulai dari Prometheus:** Mimir, Alloy, dan Grafana semuanya bicara "bahasa" Prometheus. Paham Prometheus = paham seluruh stack metrics.

- Arsitektur Prometheus: pull/scrape model, TSDB, service discovery
- Scrape config, targets, jobs & instances
- Exporter: **node_exporter** (metrics server fisik/VM — wajib di on-prem), blackbox_exporter (probe HTTP/TCP/DNS)
- Tipe metric: counter, gauge, histogram, summary
- **PromQL dari nol:** selector, `rate()`, `irate()`, aggregations (`sum by`), `histogram_quantile()`, recording rules
- Alerting rules di Prometheus → **Alertmanager**: grouping, routing, silencing, inhibition
- Batas Prometheus single-node (retensi & skala lokal) → kenapa butuh Mimir

#### Modul 7.2 — Grafana Alloy & Pipeline Telemetry (Minggu 12–13)
- Tiga pilar: metrics, logs, traces; OpenTelemetry konsep
- Alloy: komponen, pipeline, `alloy run`, debugging config
- Scrape metrics (Prometheus-format), tail & parse logs, receiver OTLP
- Alloy sebagai "pengganti Prometheus agent": scrape → remote-write ke Mimir
- Deploy Alloy sebagai DaemonSet + via Helm

#### Modul 7.3 — Metrics Skala Production: Mimir (Minggu 13)
- Arsitektur Mimir: distributor, ingester, compactor, store-gateway; object storage
- Remote-write dari Alloy → Mimir; long-term retention & query
- Recording rules & alerting rules (format Prometheus, dikelola sebagai code)
- Pola produksi: Prometheus/Alloy di tiap cluster → Mimir terpusat → Grafana

#### Modul 7.4 — Logs & Traces: Loki + Tempo (Minggu 13)
- Loki: label strategy (jangan over-label!), LogQL dasar (`|=`, `| json`, `rate()`)
- Tempo: OTLP ingest, penyimpanan traces
- Correlation: log ↔ trace ↔ metrics via traceID di Grafana

#### Modul 7.5 — Grafana & Alerting Terpadu (Minggu 14)
- Dashboard design yang bermakna (USE untuk infra, RED untuk service)
- Alert rule di Grafana + routing notifikasi (contact point)
- Integrasi Alertmanager dengan notifikasi nyata (webhook/Telegram/Slack)
- Dashboard sebagai code (JSON di Git, deploy via ArgoCD)

**Lab:**
- [ ] Install Prometheus + node_exporter + Alertmanager via Helm; scrape semua node k3s
- [ ] Tulis 10 query PromQL penting (CPU, memori, disk, error rate, p95 latency)
- [ ] Full LGTM-stack via Helm di cluster k3s: Alloy + Grafana + Mimir + Loki + Tempo
- [ ] App contoh mengeluarkan metrics + structured logs + traces (OTel SDK)
- [ ] Tulis 5 query LogQL penting
- [ ] Buat dashboard USE (infra) + RED (app)
- [ ] Buat alert "error rate > 5%", "p95 latency > 500ms", "disk > 85%", "node down" — uji firing & notifikasi

**Deliverable:** Stack observability lengkap + dashboard + alert yang terbukti berfungsi.

---

### FASE 8 — Praktik SRE & Kesiapan Production On-Prem (Minggu 15)

#### Modul 8.1 — Praktik SRE
- SLI / SLO / error budget — tulis SLO nyata untuk app sendiri
- Toil identification & automation mindset
- On-call: rotasi, escalation, runbook
- Incident response: deteksi → triage → mitigasi → resolusi → postmortem blameless
- Change management: change freeze, rollback plan, CAB ringan

#### Modul 8.2 — Karakteristik Production On-Prem
| Topik | Yang harus dikuasai |
|---|---|
| Jaringan | Static IP pool, VLAN, DNS internal, **MetalLB (L2/BGP) sebagai LoadBalancer**, tidak ada cloud LB |
| Storage | Local PV, NFS, atau SAN; StorageClass; backup data persistent |
| Hardware | Monitoring host (node_exporter), disk failure, kapasitas |
| Konfigurasi server | Semua perubahan OS via Ansible — tidak ada SSH manual untuk perubahan |
| Akses | Bastion/jump host, SSH hardening, tidak ada IAM cloud |
| Backup | Velero untuk K8s objects + PV; etcd snapshot; uji restore berkala |
| Update | Patch OS (via Ansible), upgrade k3s terkontrol, maintenance window |
| Keamanan | RBAC, network policy, secret rotation, image scanning (Trivy) |
| Disaster Recovery | RTO/RPO, dokumentasi runbook DR |

#### Modul 8.3 — Runbook & Dokumentasi
- Tulis runbook untuk incident paling umum (node down, disk full, certificate expired, app crashloop, MetalLB IP tidak responding)
- Dokumentasi arsitektur & topologi

**Lab:**
- [ ] Tulis SLO + dashboard error budget untuk app sendiri
- [ ] Install Velero + uji backup/restore namespace
- [ ] Trivy scan image di pipeline CI
- [ ] Tulis 2 runbook + 1 postmortem (dari insiden yang disimulasikan)

---

### FASE 9 — Capstone Project + Simulasi On-Call (Minggu 16–18)

#### Capstone: "Production-Like Platform di Laptop"
Bangun sistem lengkap yang meniru production on-prem:

```
┌──────────────────────────────────────────────────────────────┐
│  MacBook Air M5 (OrbStack)                                   │
│                                                              │
│  VM "infra": OpenTofu state (MinIO), GitLab runner lokal,    │
│              Ansible control node                            │
│                                                              │
│  VM "servers": dibangun & dikonfigurasi via Ansible          │
│    → cluster k3s "production" (multi-node)                   │
│                                                              │
│  Cluster k3s "production":                                   │
│    ├── App demo (chart Helm sendiri, image dari GitLab)      │
│    ├── ArgoCD (GitOps, self-heal)                            │
│    ├── MetalLB (L2, IP pool dedicated) + Ingress             │
│    ├── Prometheus + Alertmanager + node_exporter             │
│    ├── Alloy (collector)                                     │
│    ├── Mimir + Loki + Tempo (telemetry backend)              │
│    └── Grafana (dashboard + alert)                           │
│                                                              │
│  Cluster k3d "staging" (target deploy kedua)                 │
└──────────────────────────────────────────────────────────────┘
```

**Kriteria kelulusan capstone:**
- [ ] Server/VM cluster dibangun ulang dari nol via OpenTofu + Ansible (uji: destroy & rebuild)
- [ ] Semua deploy aplikasi via GitOps (tidak ada `kubectl apply` manual di "prod")
- [ ] Pipeline CI: lint → test → build multi-arch → push → update GitOps repo
- [ ] MetalLB berfungsi: service LoadBalancer dapat IP dari pool dedicated
- [ ] Observability lengkap: metrics (Prometheus→Mimir), logs, traces, ter-correlate
- [ ] Minimal 5 alert meaningful + routing notifikasi terbukti sampai
- [ ] SLO terdefinisi & dashboard error budget
- [ ] Backup Velero + etcd snapshot berjalan otomatis & pernah di-restore
- [ ] Dokumentasi: arsitektur, runbook (min. 3), postmortem (min. 1)

#### Simulasi On-Call (Game Day)
Skenario kegagalan yang diinjeksi mentor (atau diri sendiri), harus diselesaikan dengan runbook & observability:
1. Node worker mati → app harus tetap jalan, investigasi via Grafana
2. Disk penuh di node → alert firing, mitigasi
3. Deploy rusak (bad release) → rollback via ArgoCD < 5 menit
4. Certificate ingress expired → deteksi & perpanjangan
5. Latency naik 10x → root cause via traces (Tempo)
6. MetalLB IP tidak responding (ARP conflict) → diagnosa & perbaikan
7. Rebuild total: destroy semua VM → bangunkan lagi hanya dengan repo (OpenTofu + Ansible + GitOps)

**Output:** postmortem blameless per insiden.

---

## 5. Rutinitas Mingguan

| Hari | Aktivitas |
|---|---|
| Sen–Rab | Materi + lab terstruktur |
| Kam | Review & perbaiki lab, tulis catatan/runbook |
| Jum | Mini-project / eksplorasi bebas |
| Akhir pekan | Review kumulatif + latihan troubleshooting (1–2 jam) |

**Aturan emas:**
1. Semua konfigurasi masuk Git — tidak ada snowflake.
2. Setiap lab diakhiri dengan "bagaimana ini gagal, dan bagaimana saya tahu?"
3. Biasakan `kubectl describe` dan logs sebelum bertanya/panic.

---

## 6. Checklist Kesiapan Pegang Production

Sebelum diizinkan pegang production, peserta harus bisa (diuji mentor):

- [ ] Jelaskan arsitektur cluster production & komponen kritisnya
- [ ] Cek kesehatan cluster & node dalam < 2 menit
- [ ] Jelaskan cara kerja MetalLB di jaringan on-prem & troubleshoot IP yang tidak responding
- [ ] Jalankan playbook Ansible untuk patching/hardening dengan aman (dry-run dulu)
- [ ] Baca dashboard & jelaskan arti tiap panel utama
- [ ] Tulis query PromQL/LogQL dasar untuk investigasi
- [ ] Rollback deployment via ArgoCD
- [ ] Restore backup (etcd & Velero) dari nol
- [ ] Rebuild satu server dari nol via OpenTofu + Ansible
- [ ] Menangani 3 insiden simulasi dengan runbook
- [ ] Jelaskan SLO service & apa yang dilakukan saat error budget habis
- [ ] Tahu escalation path & siapa yang dihubungi

---

## 7. Referensi & Sumber Belajar

| Topik | Sumber |
|---|---|
| Linux | "The Linux Command Line" (William Shotts) |
| Kubernetes | kubernetes.io/docs, "Kubernetes in Action" |
| k3s/k3d | docs.k3s.io, k3d.io |
| MetalLB | metallb.universe.tf (konsep L2/BGP, troubleshooting) |
| OpenTofu | opentofu.org/docs |
| Ansible | docs.ansible.com, "Ansible: Up and Running" (O'Reilly) |
| Helm | helm.sh/docs |
| ArgoCD | argo-cd.readthedocs.io |
| GitLab CI | docs.gitlab.com/ci |
| Prometheus | prometheus.io/docs, "Prometheus: Up & Running" (O'Reilly) |
| Observability | grafana.com/docs (Alloy, Mimir, Tempo, Loki), "Observability Engineering" (O'Reilly) |
| SRE | "Site Reliability Engineering" & "The Site Reliability Workbook" (gratis di sre.google/books) |
| PromQL/LogQL | prometheus.io/docs query basics, grafana.com/docs/loki/latest/query |

---

## 8. Catatan Khusus MacBook Air M5

1. **ARM64 everywhere** — pastikan semua image mendukung `linux/arm64`; gunakan `docker buildx` untuk multi-arch saat push ke registry (production on-prem biasanya amd64).
2. **OrbStack > Docker Desktop** — lebih ringan; aktifkan limit memori (mis. 8 GB) agar Mac tetap responsif.
3. **Simulasi multi-node** — k3d cluster multi-node jauh lebih ringan daripada banyak VM; gunakan k3d untuk latihan harian, VM OrbStack untuk k3s + Ansible yang lebih "nyata".
4. **Ansible dari Mac** — install via `brew install ansible` atau pip; Mac bertindak sebagai control node, VM OrbStack sebagai managed nodes (pastikan SSH antar keduanya lancar).
5. **MetalLB di OrbStack** — alokasikan subnet IP yang tidak bentrok dengan jaringan OrbStack/Wi-Fi; uji dengan `arping` dari dalam VM.
6. **Storage** — SSD Mac cepat tapi kapasitas terbatas; bersihkan image/container rutin (`docker system prune`, `k3d cluster delete`).
7. **Jaringan** — OrbStack menyediakan DNS & IP stabil untuk VM; manfaatkan untuk simulasi static IP on-prem.

---

*Dokumen ini adalah kurikulum hidup — sesuaikan urutan & kedalaman dengan kecepatan belajar peserta. Target bukan menyelesaikan semua lab, tetapi mampu mengoperasikan dan men-troubleshoot sistem secara mandiri.*
