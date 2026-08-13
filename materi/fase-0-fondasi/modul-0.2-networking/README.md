# Modul 0.2 — Networking untuk SRE

> **Tujuan akhir:** paham apa yang terjadi di balik kabel saat sebuah service "tidak reachable", dan bisa mengkonfigurasi DNS, reverse proxy, serta firewall di VM OrbStack tanpa nebak-nebak.

## Capaian Modul (Wajib)

- [ ] Bisa menjelaskan DNS record `A`/`AAAA`/`CNAME`/`SRV` & alur resolusi, serta mengkonfigurasi DNS lokal
- [ ] Bisa menghitung subnet CIDR & memilih range IP yang tidak bentrok (bekal MetalLB)
- [ ] Bisa menjelaskan TCP handshake, perbedaan TCP vs UDP, dan konsep TLS (sertifikat, handshake, terminasi)
- [ ] Bisa membaca HTTP status code, header, dan menjelaskan peran reverse proxy
- [ ] Bisa mengkonfigurasi & menjalankan reverse proxy (Caddy) dengan TLS otomatis
- [ ] Bisa menjelaskan NAT, port forwarding, dan mengkonfigurasi firewall dasar (ufw)
- [ ] Bisa menjelaskan ARP & routing — kenapa ini penting untuk MetalLB L2 mode
- [ ] Bisa men-trace masalah koneksi dengan urutan yang sistematis (`curl -v` → `ss` → `dig` → `tcpdump`)

## Rencana 3 Hari

| Hari | Materi | Lab/Evaluasi |
|---|---|---|
| 1 | [01-dns-cidr](01-dns-cidr.md), [02-tcp-udp-tls](02-tcp-udp-tls.md) | [Latihan:Dns-Tcp-Tls](evaluasi/latihan.md) |
| 2 | [03-http-proxy](03-http-proxy.md), [04-firewall-nat-arp](04-firewall-nat-arp.md) | [LAB-01](lab/LAB-01-reverse-proxy.md), [LAB-02](lab/LAB-02-dns-lokal.md) |
| 3 | Review & integrasi | [LAB-03](lab/LAB-03-trace-koneksi.md) + [Kuis](evaluasi/kuis-dan-jawaban.md) |

## Prasyarat

- Modul 0.1 selesai — terutama [05-networking-cli](../modul-0.1-linux-shell/05-networking-cli.md) (tool: `ping`, `dig`, `curl`, `ss`, `ip`, `tcpdump`)
- VM `lab01` OrbStack bisa di-SSH tanpa password (LAB-01 modul 0.1)
- Sudah membaca [Fase 0 README](../README.md)

## Deliverables Modul

1. **Reverse proxy Caddy** berjalan di VM `lab01`, melayani ≥ 2 situs ( salah satunya dengan HTTPS otomatis).
2. **DNS lokal (dnsmasq)** yang menyelesaikan nama internal (mis. `app.lab.local`) ke IP VM.
3. **Repo `sre-bootcamp/m0.2`** di GitLab yang berisi:
   - `Caddyfile` & konfigurasi dnsmasq di `lab/`
   - Aturan ufw yang dipakai di `lab/ufw-rules.txt`
   - Laporan trace koneksi di `lab/lab03-report.md`
4. **Nilai kuis ≥ 80%**

## Cara Memulai

Modul 0.1 mengajarkan **tool**-nya (`dig`, `curl`, `ss`, `tcpdump`). Modul ini mengajarkan **konsep & konfigurasi** di balik tool itu. Buka materi di satu tab, VM `lab01` di tab lain, dan **jalankan setiap perintah**. Networking adalah ilmu yang paling sering dihafal-padahal-harusnya-dipahami: tiap topik ditutup dengan "kenapa ini penting untuk MetalLB/production".

## Kaitan dengan Modul Berikutnya

Topik di modul ini bukan teori jauh — langsung dipakai:
- **DNS & CIDR** → pilih range IP MetalLB yang tidak bentrok (Modul 2.3)
- **ARP & routing** → cara kerja MetalLB L2 mode (Modul 2.3)
- **Reverse proxy & TLS** → Ingress & sertifikat di Kubernetes (Modul 2.1)
- **Firewall/NAT** → hardening server & expose service di on-prem (Modul 4.3, 8.2)