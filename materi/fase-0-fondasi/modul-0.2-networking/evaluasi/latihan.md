# Latihan — Modul 0.2 Networking untuk SRE

Lakukan latihan **di VM `lab01`** (atau dari Mac ke VM), memanfaatkan setup Caddy + dnsmasq dari LAB-01 & LAB-02. Target: pemahaman konsep + muscle memory tool.

> **Aturan:** setiap soal wajib pakai **pipeline atau perintah terminal**, bukan klik GUI. Catat output penting di `lab/log-latihan-m0.2.md`.

---

## Hari 1 — DNS & CIDR, TCP/UDP/TLS

### 1.1 DNS
```bash
# 1. Cari IPv4 & IPv6 google.com
dig google.com A +short
dig google.com AAAA +short

# 2. Cari MX & TXT record gitlab.com
dig gitlab.com MX +short
dig gitlab.com TXT +short

# 3. Trace resolusi example.com dari root
dig example.com +trace | tail -20

# 4. Reverse lookup IP 1.1.1.1
dig -x 1.1.1.1 +short

# 5. Tanyakan ke dnsmasq lokal (dari LAB-02)
dig @127.0.0.1 app.lab.local +short
dig @127.0.0.1 _kube._tcp.lab.local SRV +short
```

### 1.2 CIDR
```bash
# 1. Berapa usable host di 10.20.30.0/23? hitung manual lalu cek
ipcalc 10.20.30.0/23

# 2. Pecah 192.168.50.0/24 jadi 4 subnet sama besar. Sebutkan tiap range.
ipcalc 192.168.50.0/26

# 3. Pilih range MetalLB yang aman bila:
#    DHCP = 192.168.50.100-192.168.50.200
#    server fisik = 192.168.50.10-192.168.50.30
# Tulis jawaban + alasan kenapa aman.
```

### 1.3 TCP/UDP/TLS
```bash
# 1. Tangkap TCP handshake (2 terminal):
#   T1: sudo tcpdump -i any -n -c 12 'tcp port 80' &
#   T2: curl http://example.com >/dev/null
# Identifikasi baris SYN, SYN-ACK, ACK di output.

# 2. Hitung koneksi TIME-WAIT
ss -tn state time-wait | wc -l
ss -tn state established | wc -l

# 3. Bandingkan DNS via UDP vs TCP
dig @8.8.8.8 example.com +stats | grep time
dig +tcp @8.8.8.8 example.com +stats | grep time

# 4. Periksa sertifikat gitlab.com: subject, issuer, kadaluarsa
echo | openssl s_client -connect gitlab.com:443 -servername gitlab.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# 5. Cek port listening di VM
ssh lab01 'ss -tlnp | grep -E ":80|:443|:8080|:3000"'
# Mana yang dengar di 0.0.0.0 vs 127.0.0.1? Kenapa beda?
```

---

## Hari 2 — HTTP/Proxy & Firewall/NAT/ARP

### 2.1 HTTP
```bash
# 1. Jalankan server uji: ssh lab01 'python3 -m http.server 8080 &'
# Lihat semua header response:
curl -s -D - -o /dev/null http://app.lab.local/

# 2. Paksa method POST dan lihat status code:
curl -o /dev/null -s -w "%{http_code}\n" -X POST http://app.lab.local/

# 3. Spoof Host header (simulasi virtual host lain):
curl -H 'Host: grafana.lab.local' -k https://app.lab.local/
# Apakah masih ke service 1? Kenapa? (Caddy routing by Host)

# 4. Kirim X-Forwarded-For dan lihat apakah service mencatatnya:
curl -k -H 'X-Forwarded-For: 10.0.0.99' https://app.lab.local/
```

### 2.2 Firewall
```bash
# 1. (via SSH — izinkan 22 DULU) aktifkan ufw, izinkan 22 & 443
ssh lab01 'sudo ufw allow 22/tcp && sudo ufw allow 443/tcp && sudo ufw --force enable'
ssh lab01 'sudo ufw status verbose'

# 2. Blokir 8080, uji dari Mac, lalu buka lagi
ssh lab01 'sudo ufw deny 8080/tcp'
curl --max-time 5 http://<VM_IP>:8080/      # gagal?
ssh lab01 'sudo ufw delete deny 8080/tcp && sudo ufw reload'

# 3. Bersihkan (disable) bila hanya eksplorasi
ssh lab01 'sudo ufw --force disable'
```

### 2.3 ARP & Routing
```bash
# 1. Lihat tabel ARP VM
ssh lab01 'ip neigh show'

# 2. Flush cache ARP lalu akses sesuatu, lihat cache terbangun
ssh lab01 'sudo ip neigh flush dev eth0 && ip neigh show'
ssh lab01 'ping -c 1 <gateway-IP> && ip neigh show'

# 3. Lihat rute ke 8.8.8.8
ssh lab01 'ip route get 8.8.8.8'

# 4. Arping gateway — apakah menjawab?
ssh lab01 'sudo arping -c 3 <gateway-IP>'

# 5. Jelaskan dengan 1 kalimat: kenapa saat IP MetalLB pindah node, host lain butuh waktu untuk sadar?
```

---

## Hari 3 — Integrasi & Troubleshooting

### 3.1 Latihan Terstruktur
Kerjakan ulang mini-skenario dari [LAB-03](../lab/LAB-03-trace-koneksi.md) tanpa lihat panduan:
1. Bunuh service 1 → diagnosa → pulihkan
2. Blokir port via ufw → diagnosa → pulihkan
3. Hapus resolver DNS → diagnosa → pulihkan

Catat di laporan: **urutan langkah** yang kamu ambil & di langkah ke-berapa akar masalah ketemu.

### 3.2 Soal Refleksi
Tulis jawaban singkat di `lab/log-latihan-m0.2.md`:
1. Kenapa `dig app.lab.local` dari Mac kosong setelah `/etc/resolver/lab.local` dihapus, padahal `ssh lab01 'curl localhost:8080'` tetap jalan?
2. Bandingkan diagnosa "port tidak listening" (Skenario A LAB-03) vs "DNS gagal" (Skenario C) — kenapa gejala pengguna identik tapi langkah penyelesaian beda?
3. Saat memilih subnet MetalLB, apa risiko memilih range yang di dalam DHCP pool?

---

## Catatan Performa

- [ ] Semua latihan dilakukan di terminal (tidak ada GUI)
- [ ] Output penting disimpan di repo `sre-bootcamp` di `lab/log-latihan-m0.2.md`
- [ ] Bisa menjelaskan tiap perintah yang dipakai & kenapa urutannya begitu
- [ ] Bisa menghubungkan tiap topik ke MetalLB/production (jadi bukan teori)