# Latihan Modul 2.3 — MetalLB

> Kerjakan setelah LAB-01 dan LAB-02. Gunakan cluster k3s VM OrbStack. Semua output sensitif (token/kubeconfig) harus disamarkan sebelum masuk laporan.

## Aturan Pengumpulan

- Simpan manifest dan laporan di branch feature.
- Buka Merge Request dengan judul Conventional Commit, misalnya `docs(metallb): complete l2 loadbalancer lab`.
- Sertakan output yang relevan, bukan secret.
- Jelaskan asumsi network: subnet node, DHCP range, VIP pool, interface Mac.
- Nilai minimal yang disarankan: 80% dan seluruh acceptance criteria lab terpenuhi.

## Latihan 1 — Network Inventory & Pool Design

1. Jalankan `orb ip` untuk semua Machine k3s dan `ip route` dari VM.
2. Tentukan subnet, gateway, DHCP range, dan 5–10 alamat VIP.
3. Verifikasi kandidat VIP dengan `arping`.
4. Buat tabel reservation dan jelaskan mengapa range tidak bentrok.

**Output:** `network-inventory.md` + diagram subnet.

## Latihan 2 — Allocation dan Advertisement

1. Buat `IPAddressPool` dengan `autoAssign: true`.
2. Buat `L2Advertisement` eksplisit yang menunjuk pool.
3. Deploy dua Service `type=LoadBalancer`.
4. Buktikan masing-masing mendapat VIP berbeda.
5. Ubah salah satu Service untuk meminta pool tertentu jika cluster memiliki lebih dari satu pool.

**Pertanyaan:** tahap mana dilakukan Controller dan tahap mana dilakukan Speaker?

## Latihan 3 — ARP Evidence

1. Dapatkan VIP dengan JSONPath.
2. Jalankan `arping` minimal lima kali dari client yang tepat.
3. Catat MAC reply dan bandingkan dengan node Speaker.
4. Periksa `arp -a` di Mac atau `ip neigh` di Linux.
5. Ambil log Speaker dan jelaskan leader election secara konseptual.

**Output:** bukti command + paragraf interpretasi, bukan hanya screenshot.

## Latihan 4 — Service Path

1. Akses `curl http://<VIP>` dari Mac.
2. Periksa `Endpoints/EndpointSlice`, readiness Pod, dan `Service port/targetPort`.
3. Jalankan curl dari Pod debug.
4. Ubah `externalTrafficPolicy` ke `Local`, uji dengan penempatan Pod yang terkontrol, lalu kembalikan ke `Cluster`.
5. Jelaskan dampak source IP dan reachability.

## Latihan 5 — Troubleshooting Drill

Pilih minimal tiga skenario berikut dan tulis diagnosis berurutan:

- Service tetap `<pending>` karena pool salah/habis.
- VIP terisi tetapi `arping` timeout.
- IP conflict menghasilkan MAC reply tidak konsisten.
- ARP berhasil tetapi TCP/HTTP timeout.
- `externalTrafficPolicy: Local` tidak punya endpoint di node penerima.
- Speaker tidak Running karena node/resource/image issue.

Untuk setiap skenario, isi:

```text
Gejala:
Impact:
Command pertama:
Evidence:
Hipotesis:
Perbaikan yang aman:
Validasi setelah perbaikan:
Pencegahan:
```

## Latihan 6 — BGP Design Review

Tanpa membuat peering ke router nyata:

1. Gambar topologi MetalLB Speaker → ToR router.
2. Tentukan private ASN, peer ASN, peer address, dan VIP prefix.
3. Jelaskan bagaimana ECMP membagi flow.
4. Tulis minimal lima route/security policy yang harus direview.
5. Bandingkan kapan desain Anda memakai L2 dan kapan memakai BGP.

## Latihan 7 — Runbook SRE

Tulis runbook satu halaman berjudul `runbook-metallb-vip.md` yang memuat:

- symptom dan impact;
- command verifikasi context;
- allocation → speaker → ARP → TCP → HTTP;
- rollback/cleanup;
- kapan eskalasi ke tim network;
- data yang tidak boleh dimasukkan ke tiket (token, kubeconfig, secret).

## Tantangan Opsional — Capacity & Failure

- Hitung berapa Service LB yang dapat ditampung pool Anda.
- Hapus satu Pod app dan buktikan Service tetap tersedia.
- Dengan instruktur, simulasi satu node speaker failover.
- Catat convergence time dan apakah koneksi aktif reset.
- Jelaskan mengapa L2 bukan active-active per VIP.

## Rubrik Singkat

| Area | Bobot |
|---|---:|
| Network inventory dan pool aman | 20% |
| Instalasi/configuration MetalLB | 20% |
| Bukti ARP dan akses VIP | 20% |
| Troubleshooting berlapis | 25% |
| BGP design + dokumentasi SRE | 15% |

## Kaitan dengan Modul 2.4

Latihan ini membangun kebiasaan observability dan incident response yang diperlukan untuk Modul 2.4: mengumpulkan evidence, menjaga context safety, menguji failure, serta membedakan symptom dari root cause.
