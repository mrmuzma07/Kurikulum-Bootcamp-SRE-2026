# LAB-03 — Trace Masalah Koneksi (Troubleshooting Sistematis)

> **Target:** mendiagnosis 3 skenario koneksi gagal dengan urutan 7-langkah yang sistematis (link → ARP → routing → firewall → port → DNS → app), bukan nebak.

## Latar Belakang
Skenario produksi paling sering: "service tidak bisa diakses." Tanpa metode, SRE pemula akan `ping`-doa, `restart`-doa, lalu panik. Lab ini menanam **urutan cek** yang sama dipakai senior, sekaligus memakai semua tool dari modul 0.1 & 0.2.

## Prasyarat
- [ ] LAB-01 & LAB-02 modul 0.2 selesai (Caddy + dnsmasq jalan)
- [ ] Sudah baca [04-firewall-nat-arp](../04-firewall-nat-arp.md)
- [ ] Akses sudo di VM `lab01`

## Waktu
± 90 menit

## Kerangka 7-Langkah (referensi)

```
1. Fisik/link   → ip -br addr show
2. ARP/L2       → ip neigh show, arping
3. Routing      → ip route get <target>
4. Firewall     → ufw status, iptables -L -n -v
5. Port/listen  → ss -tlnp
6. DNS          → dig <name>
7. App          → curl -v, journalctl
```

Setiap skenario: **catat di laporan** output per langkah & kesimpulan.

---

## Skenario A — Port Tidak Listening

### 1. Buat Situasi
```bash
ssh lab01 <<'EOF'
# Bunuh service 1 (port 8080) dari LAB-01
pkill -f srv1.py || true
sleep 1
ss -tlnp | grep :8080 || echo "8080 tidak listening (sesuai skenario)"
EOF
```

### 2. Investigasi dari Mac
```bash
# Langkah 5 (port) — yang seharusnya pertama dicurigai
ssh lab01 'ss -tlnp | grep :8080'          # kosong

# Langkah 6 (DNS) — nama masih resolve (bukan sumber masalah)
dig app.lab.local +short

# Langkah 7 (app) — bukti
curl -v https://app.lab.local/             # Caddy balas 502 (backend hilang)
ssh lab01 'sudo journalctl -u caddy -n 20 --no-pager | grep -i error'
```

**Diagnosis:** DNS OK, Caddy OK, tapi backend port 8080 mati → `502 Bad Gateway`.

### 3. Pulihkan & Verifikasi
```bash
ssh lab01 'nohup python3 /tmp/srv1.py >/tmp/srv1.log 2>&1 &'
sleep 1
curl -k https://app.lab.local/             # "hello from SERVICE-1"
```

---

## Skenario B — Firewall Memblokir

### 1. Buat Situasi
```bash
ssh lab01 <<'EOF'
set -e
# Pastikan SSH tetap aman dulu!
sudo ufw allow 22/tcp
sudo ufw --force enable
# Blokir port 3000 (service 2) dari luar
sudo ufw deny 3000/tcp
sudo ufw status numbered | head
EOF
```

### 2. Investigasi
```bash
# Langkah 5 (port) — service masih dengar
ssh lab01 'ss -tlnp | grep :3000'          # ada, listening

# Langkah 4 (firewall) — temuan kunci
ssh lab01 'sudo ufw status | grep 3000'    # DENY 3000/tcp

# Langkah 7 — gejala
curl -v --max-time 5 http://<VM_IP>:3000/  # timeout / connection refused
```

**Diagnosis:** Service listening, tapi firewall menolak → konflik aturan ufw. (Catatan: karena proxy berjalan di VM yang sama dengan Caddy, akses via nama Caddy mungkin tetap jalan; akses langsung ke port terblokir.)

### 3. Pulihkan
```bash
ssh lab01 'sudo ufw delete deny 3000/tcp && sudo ufw reload'
curl --max-time 5 http://<VM_IP>:3000/     # OK
# Bersihkan ufw bila hanya eksplorasi:
ssh lab01 'sudo ufw --force disable'
```

---

## Skenario C — DNS Gagal (Nama Tidak Resolve)

### 1. Buat Situasi
```bash
# Hapus resolver per-domain di Mac
sudo rm -f /etc/resolver/lab.local
sudo sed -i '' '/lab.local/d' /etc/hosts   # hapus hosts manual juga
```

### 2. Investigasi
```bash
# Langkah 6 (DNS) — temuan langsung
dig app.lab.local +short                   # kosong / SERVFAIL
dig app.lab.local                          # lihat status: NXDOMAIN/SERVFAIL

# Bandingkan dengan yang masih jalan
dig example.com +short                     # publik OK → bukan konektivitas umum

# Langkah 5 (port) — service di VM sebenarnya sehat
ssh lab01 'ss -tlnp | grep -E ":8080|:3000"'   # listening
ssh lab01 'curl -s http://localhost:8080/'     # OK dari dalam VM
```

**Diagnosis:** Layanan sehat & reachable dari dalam VM, tapi nama tidak resolve dari Mac. Masalah di lapisan DNS (resolver hilang), bukan service. Beda diagnosa vs Skenario A (port mati) — walau gejala pengguna sama: "tidak bisa akses."

### 3. Pulihkan
```bash
VM_IP=$(ssh lab01 'hostname -I | awk "{print \$1}"')
sudo mkdir -p /etc/resolver
echo "nameserver $VM_IP" | sudo tee /etc/resolver/lab.local
dig app.lab.local +short                   # resolve lagi
curl -k https://app.lab.local/             # OK
```

---

## Laporan

Buat `lab/lab03-report.md` di repo dengan struktur:

```markdown
# LAB-03 — Trace Koneksi Report

## Skenario A: Port Tidak Listening
- Gejala: ...
- Langkah 5 (port): output `ss` ...
- Langkah 6 (DNS): output `dig` ...
- Langkah 7 (app): `curl -v` → 502
- Diagnosis: backend port 8080 mati, Caddy 502
- Perbaikan: restart srv1.py

## Skenario B: Firewall Memblokir
- Gejala: ...
- Langkah 4 (firewall): `ufw status` → DENY 3000
- Langkah 5 (port): service tetap listening
- Diagnosis: ...
- Perbaikan: ...

## Skenario C: DNS Gagal
- Gejala: ...
- Langkah 6 (DNS): `dig` → NXDOMAIN/SERVFAIL
- Langkah 5 (port): service sehat dari dalam VM
- Diagnosis: ...
- Perbaikan: ...

## Pelajaran
- Kenapa urutan 7-langkah penting (bandingkan A vs C: gejala sama, akar beda)
- Tool yang paling cepat menemukan akar di tiap skenario
```

## Acceptance Criteria

- [ ] 3 skenario selesai, masing-masing didiagnosis & dipulihkan
- [ ] Laporan berisi output nyata per langkah (bukan teori)
- [ ] Bisa menjelaskan kenapa Skenario A & C punya gejala sama tapi akar beda
- [ ] VM kembali ke keadaan sehat: `curl -k https://app.lab.local/` & `grafana.lab.local` jalan
- [ ] Firewall dikembalikan (disable atau rules bersih)
- [ ] `lab/lab03-report.md` tersimpan di repo GitLab

## Troubleshooting

| Gejala | Solusi |
|---|---|
| Terkunci SSH setelah ufw | Buka terminal baru; di VM console OrbStack: `sudo ufw disable` |
| `pkill -f srv1.py` membunuh terlalu banyak | Spesifikkan PID: `pgrep -f srv1.py` lalu `kill <PID>` |
| `dig` tetap resolve setelah hapus resolver | Cache macOS: `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` |
| `curl` hang tanpa pesan | Tambah `--max-time 5` agar tidak menunggu selamanya |
| 502 terus padahal service sudah restart | Caddy cache upstream down? `sudo systemctl reload caddy` |

## Quick Reference — Urutan Troubleshooting

```
1. ip -br addr show        link up? ada IP?
2. ip neigh show / arping  ARP ok? (L2)
3. ip route get <IP>       lewat interface benar?
4. ufw status / iptables   firewall blokir?
5. ss -tlnp                service dengar? 0.0.0.0?
6. dig <name>              nama resolve?
7. curl -v / journalctl    app error?
```

## Lanjut
[Evaluasi: Latihan & Kuis](../evaluasi/latihan.md)