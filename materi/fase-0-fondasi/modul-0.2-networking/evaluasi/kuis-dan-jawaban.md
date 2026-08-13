# Kuis & Kunci Jawaban — Modul 0.2 Networking untuk SRE

> **Cara kerja:** jawab sendiri tanpa buka catatan. Cek di bagian bawah. Target ≥ 80% (16 dari 20).

---

## Bagian A — Pilihan Ganda (10 soal)

**1.** Record DNS `CNAME` digunakan untuk...
- A. Memetakan nama ke IPv6
- B. Membuat alias dari satu nama ke nama lain
- C. Menyimpan port & prioritas service
- D. Verifikasi kepemilikan domain

**2.** Berapa usable host di subnet `10.10.0.0/22`?
- A. 254
- B. 510
- C. 1022
- D. 2046

**3.** Saat migrasi IP server, kenapa TTL DNS diturunkan jauh hari sebelumnya?
- A. Agar query lebih cepat
- B. Agar resolver lebih hemat memori
- C. Agar perubahan IP menyebar cepat saat hari-H
- D. Agar authoritative server tidak overload

**4.** TCP three-way handshake yang benar adalah...
- A. SYN → ACK → FIN
- B. SYN → SYN-ACK → ACK
- C. SYN → FIN → SYN-ACK
- D. ACK → SYN → SYN-ACK

**5.** Kenapa `0.0.0.0:8080` bisa diakses dari host lain tapi `127.0.0.1:8080` tidak?
- A. `0.0.0.0` pakai IPv6
- B. `127.0.0.1` hanya dengar di interface loopback (localhost)
- C. `0.0.0.0` butuh privilege lebih rendah
- D. `127.0.0.1` selalu diblok firewall

**6.** HTTP status `502 Bad Gateway` berarti...
- A. Client tidak terautentikasi
- B. Reverse proxy tidak dapat respons valid dari upstream
- C. Service sengaja menolak karena maintenance
- D. Upstream tidak menjawab dalam batas waktu

**7.** Beda `401 Unauthorized` vs `403 Forbidden`...
- A. Keduanya sama, hanya nama beda
- B. 401 = "siapa kamu?", 403 = "kamu siapa tahu, tapi tidak boleh"
- C. 401 = error server, 403 = error client
- D. 401 = permanent, 403 = temporary

**8.** Saat konfigurasi firewall via SSH, aturan emasnya adalah...
- A. Matikan firewall dulu, baru konfigurasi
- B. Izinkan port 22 SEBELUM `default deny incoming`
- C. Selalu pakai `iptables` langsung, bukan `ufw`
- C. Hapus semua aturan lalu restart SSH

**9.** MetalLB L2 mode "mengklaim" IP LoadBalancer dengan cara...
- A. Mengubah tabel routing router
- B. Mengirim Gratuitous ARP ke jaringan
- C. Meminta IP ke DHCP server
- D. Membuka tunnel TCP ke semua node

**10.** Saat IP MetalLB berpindah node (failover), kenapa ada downtime singkat?
- A. Pod harus restart dulu
- B. Cache ARP host lain belum kedaluarsa, trafik masih ke MAC lama
- C. Sertifikat TLS harus diperbarui
- D. DNS harus di-flush

---

## Bagian B — Perintah (4 soal)

**11.** Tulis perintah untuk: melihat semua koneksi TCP yang sedang **established**, beserta proses pemiliknya.

**12.** Tulis perintah untuk: memeriksa subject, issuer, dan tanggal kadaluarsa sertifikat `https://example.com`.

**13.** Tulis konfigurasi `dnsmasq` (1 baris `srv-host`) untuk SRV record: service `_metrics._tcp.lab.local` → target `prometheus.lab.local` port `9090`, priority 10, weight 10.

**14.** Tulis aturan `ufw` untuk: mengizinkan akses port 6443 (Kubernetes API) **hanya** dari subnet `192.168.97.0/24`.

---

## Bagian C — Skenario (2 soal)

**15.** Sebuah Service `grafana.lab.local` tiba-tiba tidak bisa diakses dari Mac. Dari dalam VM `lab01`, `curl http://localhost:3000` berjalan normal. Tulis urutan investigasi (7-langkah) dan di langkah mana kamu akan menemukan akar masalah untuk kasus ini.

**16.** Kamu akan memasang MetalLB di jaringan `192.168.1.0/24`. DHCP server meng-assign `192.168.1.100-192.168.1.200`. Ada 5 server fisik static IP `192.168.1.10-192.168.1.15`. Tulis pool IP MetalLB yang aman + jelaskan kenapa, dan apa yang terjadi kalau kamu malah memilih `192.168.1.150` (dalam DHCP).

---

## Bagian D — Troubleshooting (2 soal)

**17.** `curl -k https://app.lab.local/` mengembalikan `502 Bad Gateway`. Tapi `ssh lab01 'ss -tlnp | grep 8080'` menunjukkan service tetap listening. Sebutkan 3 dugaan penyebab & cara ceknya.

**18.** Setelah `sudo ufw enable`, koneksi SSH Mac→VM terputus dan tidak bisa masuk lagi. Bagaimana cara masuk kembali, dan apa yang seharusnya dilakukan sebelumnya agar tidak terjadi?

---

## Bagian E — Esai Singkat (2 soal)

**19.** Jelaskan kenapa ARP adalah "jantung" MetalLB L2 mode, dan apa konsekuensinya bagi pemilihan topologi jaringan (single-subnet vs multi-subnet).

**20.** Di production on-prem, kamu memutuskan terminasi TLS di reverse proxy (bukan di aplikasi). Sebutkan 2 keuntungan dan 1 risiko, serta syarat jaringan agar risiko itu dapat diterima.

---

## Kunci Jawaban

### A — Pilihan Ganda
1. **B** — CNAME = alias nama → nama lain
2. **C** — /22 → 32-22=10 bit host → 2^10=1024, minus network+broadcast = 1022
3. **C** — TTL pendek = cache cepat kedaluarsa = perubahan IP cepat menyebar saat hari-H
4. **B** — SYN → SYN-ACK → ACK
5. **B** — `127.0.0.1` bind loopback saja; `0.0.0.0` bind semua interface
6. **B** — 502 = proxy tidak dapat respons valid dari upstream
7. **B** — 401 "siapa kamu" (belum login/token salah), 403 "tahu siapa, tidak boleh"
8. **B** — izinkan 22 dulu sebelum deny; (catatan: soal sengaja punya dua opsi "C", yang benar B)
9. **B** — L2 mode pakai Gratuitous ARP mengumumkan kepemilikan IP
10. **B** — cache ARP host lain belum expire, trafik nyasar ke MAC node lama

### B — Perintah
11. ```bash
    ss -tnp state established
    ```
12. ```bash
    echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
      | openssl x509 -noout -subject -issuer -dates
    ```
13. ```bash
    srv-host=_metrics._tcp.lab.local,prometheus.lab.local,9090,10,10
    ```
14. ```bash
    sudo ufw allow from 192.168.97.0/24 to any port 6443 proto tcp
    ```

### C — Skenario
15. Urutan 7-langkah (ref: 04-firewall-nat-arp):
    1. **Link**: `ip -br addr show` — interface up?
    2. **ARP/L2**: `ip neigh`, `arping <VM_IP>` — satu subnet? cache OK?
    3. **Routing**: `ip route get <VM_IP>` — lewat interface benar?
    4. **Firewall**: `ufw status`, `iptables -L -n -v` — port 3000 diblok?
    5. **Port/listen**: `ss -tlnp | grep 3000` — service dengar? (soal menyebut dari-dalam-VM jalan → listening OK, tapi cek `0.0.0.0` vs `127.0.0.1`)
    6. **DNS**: `dig grafana.lab.local` — **di sini kemungkinan besar ketemu** (nama tidak resolve dari Mac, padahal VM sehat). Karena `curl localhost:3000` di VM OK, masalah bukan service.
    7. **App**: `curl -v`, log — konfirmasi.

    Akar paling mungkin ditemukan di **langkah 6 (DNS)** atau **langkah 5 (bind 127.0.0.1)** — tergantung gejala. Kunci: dari-dalam-VM OK tapi dari-Mac gagal mempersempit ke DNS/firewall/bind-address, bukan aplikasi.

16. Pool aman contoh: `192.168.1.240-192.168.1.250` (di luar DHCP 100-200 dan di luar static 10-15). Alasan: tidak tumpang tindih dengan DHCP (tidak akan di-assign ke device lain) maupun IP server fisik. Jika memilih `192.168.1.150` (dalam DHCP): DHCP bisa memberi IP itu ke laptop/printer lain → **ARP conflict** — dua host mengklaim IP yang sama, trafik nyasar, LoadBalancer tidak reliable. Ini persis skenario troubleshooting MetalLB "IP tidak responding".

### D — Troubleshooting
17. Dugaan + cek:
    - **Backend mati/baru restart** walau `ss` sempat listening: cek ulang `ss -tlnp` saat ini + `journalctl` backend. Mungkin service crash-loop.
    - **Bind ke `127.0.0.1` bukan `0.0.0.0`**: Caddy (di VM yang sama) mungkin masih bisa, tapi jika Caddy di host lain → gagal. Cek `ss -tlnp` kolom alamat.
    - **Caddy config salah upstream / port**: `cat /etc/caddy/Caddyfile` + `sudo journalctl -u caddy` cari "dial" error.
    - (bonus) **Firewall antara Caddy & backend**: `ufw status` — walau jarang antar-proses-satu-VM.
    - (bonus) **Backend return non-HTTP/garbage**: `curl -v http://localhost:8080` langsung dari VM lihat response valid.

18. Masuk kembali: buka **console VM OrbStack** langsung (bukan SSH), jalankan `sudo ufw disable` atau `sudo ufw allow 22/tcp`. Seharusnya dilakukan sebelumnya: (a) `sudo ufw allow 22/tcp` **sebelum** `ufw enable`/`default deny`; atau (b) jalankan dengan timeout `sudo ufw enable && sleep 60 && sudo ufw disable` + uji dari terminal SSH kedua sebelum menutup sesi.

### E — Esai
19. ARP = jembatan IP→MAC. MetalLB L2 memilih satu node leader lalu mengirim **Gratuitous ARP** "IP 192.168.1.240 sekarang di MAC-ku" agar semua host di subnet mengirim trafik ke node itu. Konsekuensi topologi: ARP broadcast **tidak lewat router**, jadi L2 mode hanya berfungsi dalam **satu subnet/L2 domain**. Untuk multi-subnet/production besar → butuh **BGP mode** (peering router, routing IP-level, bukan ARP). Juga: saat failover, cache ARP lama → downtime singkat sampai expire/flush.

20. Keuntungan: (1) **satu titinjau sertifikat** — mudah rotasi/monitor kadaluarsa, app fokus logika; (2) **offload enkripsi** dari app → CPU app turun, bisa HTTP murni internal. Risiko: trafik proxy→app **tidak terenkripsi (HTTP)** → jika jaringan internal tidak tepercaya, bisa disadap. Syarat diterima: jaringan internal terisolasi/terkontrol (VLAN terpisah, network policy, tidak ada host sembarangan di segmen itu), atau terapkan mTLS end-to-end untuk service sensitif.

---

## Penilaian

| Benar | Skor |
|---|---|
| 18–20 | Expert — lanjut Modul 0.3 (Git) |
| 16–17 | Lulus — boleh lanjut, perbaiki yang salah |
| 12–15 | Belum lulus — ulang materi, kerjakan ulang latihan & lab |
| < 12 | Ulangi semua materi, lanjut mentor |