# LAB-02 — Troubleshooting MetalLB: Pending, ARP Conflict & Traffic

> **Skenario:** Service sudah dibuat, tetapi hasilnya tidak selalu sehat. Anda akan mendiagnosis kegagalan secara berurutan: allocation → speaker → ARP → TCP → Service/Pod.

## Tujuan

- Menggunakan pola debug berlapis, bukan menebak-nebak.
- Membedakan `EXTERNAL-IP: <pending>` dari kegagalan ARP.
- Mengidentifikasi IP pool habis/salah, advertisement tidak cocok, IP conflict, dan firewall/route.
- Mengumpulkan bukti yang cukup untuk laporan incident.
- Memulihkan konfigurasi lab tanpa operasi destruktif luas.

## Prasyarat dan Safety

- LAB-01 selesai atau instruktur menyediakan cluster k3s + MetalLB.
- Context k3s sudah diverifikasi.
- Satu Service `web` bertipe `LoadBalancer` tersedia.
- Peserta bekerja hanya pada namespace `metallb-lab` dan resource lab.
- Jangan menjalankan `kubectl delete -A`, mengubah gateway, atau mengubah router jaringan.
- Skenario conflict dan firewall hanya boleh dilakukan pada network lab yang diizinkan instruktur.

## Pola Debug Utama

```text
1. Allocation: apakah Controller memberi VIP?
2. Speaker: apakah Speaker aktif dan advertisement cocok?
3. ARP/NDP: apakah client menemukan MAC VIP?
4. TCP: apakah port Service dapat dijangkau?
5. HTTP/app: apakah endpoint dan aplikasi sehat?
```

Command baseline:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get svc web -n metallb-lab -o wide
kubectl get endpointslice -n metallb-lab -l kubernetes.io/service-name=web
kubectl get pods -n metallb-system -o wide
kubectl get events -n metallb-system --sort-by=.lastTimestamp
```

Simpan output baseline sebelum mengubah apa pun.

## Skenario A — `EXTERNAL-IP` Tetap `<pending>`

### Gejala

```text
NAME   TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)
web    LoadBalancer   10.43.x.x    <pending>     80:3xxxx/TCP
```

### Langkah Diagnosis

```bash
kubectl describe svc web -n metallb-lab
kubectl get ipaddresspool,l2advertisement -n metallb-system -o yaml
kubectl logs -n metallb-system deploy/controller --tail=120
```

Periksa:

1. Apakah controller `Running`?
2. Apakah `IPAddressPool` ada di namespace `metallb-system`?
3. Apakah `L2Advertisement` menunjuk nama pool yang benar?
4. Apakah pool masih memiliki alamat bebas?
5. Apakah `autoAssign: false` tanpa annotation pool pada Service?
6. Apakah `apiVersion` sesuai CRD yang terpasang?

### Latihan Perbaikan

Jika instruktur memberi pool khusus, koreksi manifest hanya setelah review:

```yaml
spec:
  type: LoadBalancer
  # Hanya jika pool autoAssign=false atau beberapa pool digunakan
  # annotations berada di metadata Service:
metadata:
  annotations:
    metallb.io/address-pool: lab-pool
```

Apply dan amati:

```bash
kubectl apply -f web-loadbalancer.yaml
kubectl get svc web -n metallb-lab -w
```

**Jangan** langsung menghapus semua Service atau CRD. Bukti `<pending>` harus berasal dari `describe`, pool, dan controller logs.

### Acceptance Skenario A

- [ ] Anda menunjukkan bukti bahwa masalah terjadi sebelum tahap ARP.
- [ ] Pool/advertisement atau kapasitas pool diperiksa.
- [ ] Setelah koreksi, `EXTERNAL-IP` terisi tanpa mengubah namespace lain.

## Skenario B — VIP Ada, tetapi `arping` Timeout

### Gejala

```bash
VIP=$(kubectl get svc web -n metallb-lab -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
IFACE=$(route get default | awk '/interface:/{print $2}')
sudo arping -I "$IFACE" -c 3 "$VIP"
```

Tidak ada reply atau request timeout.

### Langkah Diagnosis

```bash
kubectl get pods -n metallb-system -l component=speaker -o wide
kubectl logs -n metallb-system -l component=speaker --tail=150
kubectl get l2advertisement -n metallb-system -o yaml
orb ip k3s-cp1
route -n get "$(orb ip k3s-cp1)"
ip route 2>/dev/null || true
```

Kemungkinan penyebab:

- Mac memakai interface/route yang salah.
- VIP tidak satu L2 subnet dengan client dan node.
- Speaker tidak `Ready` atau tidak eligible.
- `L2Advertisement` tidak menunjuk pool yang dialokasikan.
- OrbStack network tidak meneruskan ARP seperti asumsi lab.
- Firewall memblokir traffic atau network isolation aktif.
- VIP sudah dipakai perangkat lain sehingga hasil tidak konsisten.

Uji dari sisi VM yang berada pada subnet yang sama bila Mac tidak bisa melihat ARP:

```bash
# Jalankan di VM Linux yang punya interface pada subnet VIP
sudo arping -I eth0 -c 3 "$VIP"
ip neigh show | grep "$VIP" || true
```

### Perbaikan yang Aman

- Gunakan interface yang benar dari `route -n get <IP_VM>`.
- Pilih pool baru yang satu subnet dan sudah di-reserve.
- Pastikan `L2Advertisement` dan pool menggunakan nama yang sama.
- Pulihkan Speaker yang gagal setelah membaca `describe`/events.
- Jika network OrbStack tidak mendukung jalur ARP yang dibutuhkan, catat keterbatasan dan pindahkan lab ke topologi VM/network yang disetujui; jangan memalsukan acceptance criterion.

### Acceptance Skenario B

- [ ] Anda memisahkan masalah route/interface dari masalah Speaker.
- [ ] Anda menjalankan `arping` dari lokasi yang tepat.
- [ ] Anda menyimpan log Speaker dan status advertisement.
- [ ] Anda tidak menyimpulkan app rusak hanya karena ARP gagal.

## Skenario C — Simulasi IP Conflict (Hanya Network Lab)

> Jalankan hanya jika instruktur menyediakan perangkat/namespace uji yang aman. Jangan mengambil IP aktif milik DHCP atau perangkat rumah/kantor.

### Gejala

VIP sudah terisi dan `arping` mendapat jawaban yang berubah-ubah atau MAC bukan yang diharapkan. Dua pemilik IP dapat menyebabkan koneksi intermittent.

```bash
sudo arping -I "$IFACE" -c 10 "$VIP"
arp -a | grep "$VIP" || true
```

### Diagnosis

```bash
kubectl get svc web -n metallb-lab -o yaml
kubectl get ipaddresspool lab-pool -n metallb-system -o yaml
```

Bandingkan MAC reply dengan interface node yang diiklankan oleh Speaker. Jika ada perangkat kedua menjawab:

1. Hentikan request aplikasi ke VIP.
2. Catat waktu, VIP, MAC, dan interface.
3. Minta pemilik network memeriksa DHCP/static reservation.
4. Jangan memaksa MetalLB "menang" dengan manipulasi ARP.
5. Setelah IP dilepas dan cache bersih, ulangi `arping`.

### Acceptance Skenario C

- [ ] Anda menyebut IP conflict sebagai incident network, bukan masalah selector Pod.
- [ ] VIP ditarik dari penggunaan sampai ownership jelas.
- [ ] Pool diperbarui hanya setelah reservation network disepakati.

## Skenario D — ARP Berhasil, tetapi TCP/HTTP Tidak Sampai

### Gejala

```bash
sudo arping -I "$IFACE" -c 3 "$VIP"   # berhasil
curl -v --connect-timeout 5 "http://${VIP}/" # timeout/refused
```

### Langkah Diagnosis

```bash
kubectl describe svc web -n metallb-lab
kubectl get endpointslice -n metallb-lab -l kubernetes.io/service-name=web -o wide
kubectl get pods -n metallb-lab -o wide
kubectl get networkpolicy -n metallb-lab
kubectl get nodes -o wide
```

Periksa:

- Service `port` dan `targetPort` cocok dengan container.
- EndpointSlice berisi alamat Pod Ready.
- Readiness probe tidak membuat endpoint kosong.
- Node dan kube-proxy sehat.
- Firewall mengizinkan TCP port 80.
- VIP tidak hanya dapat di-ARP tetapi route return traffic juga benar.

Uji dari dalam cluster untuk memisahkan aplikasi dari external path:

```bash
kubectl run curl-debug -n metallb-lab --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  curl -v --connect-timeout 5 http://web.metallb-lab.svc.cluster.local/
```

Interpretasi:

- Internal curl gagal → fokus Deployment, Pod, Service, endpoints, atau NetworkPolicy.
- Internal curl berhasil + ARP gagal → fokus L2/network.
- Internal curl dan ARP berhasil + external curl gagal → fokus TCP firewall, route, kube-proxy, atau policy external.

### `externalTrafficPolicy`

```bash
kubectl get svc web -n metallb-lab -o jsonpath='{.spec.externalTrafficPolicy}{"\n"}'
```

Dengan `Local`, traffic yang masuk ke node tanpa endpoint lokal dapat gagal. Untuk lab, gunakan default `Cluster` kecuali sedang menguji source IP:

```bash
kubectl patch svc web -n metallb-lab -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

### Acceptance Skenario D

- [ ] Anda menguji Service dari dalam cluster.
- [ ] EndpointSlice dan readiness diperiksa.
- [ ] Anda menjelaskan perbedaan layer network vs application.
- [ ] Perubahan `externalTrafficPolicy` dicatat dan dapat di-rollback.

## Skenario E — Strict ARP / kube-proxy

MetalLB L2 memerlukan node menjawab traffic VIP dengan benar. Pada beberapa kombinasi kube-proxy/IPVS, strict ARP menjadi pertimbangan. Jangan mengubah setting secara membabi buta, terutama pada k3s yang umumnya menggunakan iptables.

Checklist:

```bash
kubectl get nodes -o wide
kubectl get configmap -A | grep -i kube-proxy || true
# Pada VM, review mode kube-proxy sesuai dokumentasi cluster
sudo k3s check-config 2>/dev/null || true
```

Jika instruktur sengaja memberi cluster IPVS, ikuti konfigurasi resmi versi cluster dan dokumentasikan before/after. Acceptance bukan "ubah strictARP sampai berhasil", tetapi:

- mode proxy diketahui;
- perubahan memiliki alasan dan rollback;
- hasil `arping`/curl dibandingkan sebelum dan sesudah;
- konfigurasi tidak merusak Service lain.

## Skenario F — Buat Incident Timeline

Isi tabel berikut pada `m2.3/lab-02-incident.md`:

| Waktu | Observasi | Command/bukti | Hipotesis | Tindakan | Hasil |
|---|---|---|---|---|---|
| | `<pending>` / timeout / refused | | | | |
| | | | | | |
| | | | | | |

Tambahkan:

- impact: Service/namespace/client yang terdampak;
- scope: hanya satu VIP atau semua Service;
- first known good;
- perubahan terakhir;
- root cause atau evidence yang belum cukup;
- tindakan pencegahan (pool reservation, alert, runbook).

## Acceptance Criteria Lab

- [ ] Baseline context, node, Service, endpoints, dan MetalLB tersimpan.
- [ ] Minimal dua skenario dianalisis dari lab yang ditentukan instruktur.
- [ ] Setiap diagnosis dimulai dari layer yang tepat.
- [ ] `EXTERNAL-IP` pending, ARP timeout, IP conflict, dan TCP timeout dibedakan.
- [ ] Tidak ada operasi cluster-wide yang tidak diperlukan.
- [ ] Incident timeline dan bukti command lengkap.
- [ ] Ada rekomendasi pencegahan yang dapat ditindaklanjuti.

## Ringkasan Pola Troubleshooting

```text
<pending>
  → Service describe / pool / controller logs
VIP assigned
  → speaker / L2Advertisement / subnet / route
ARP reply
  → Service / endpoints / kube-proxy / firewall TCP
HTTP response
  → app behavior, Ingress, DNS, TLS
```

## Kaitan dengan Modul Berikutnya

Pola ini dipakai ulang di Modul 2.4:

- `get` untuk scope;
- `describe` dan events untuk alasan;
- logs untuk proses;
- endpoint/probe untuk readiness;
- node/resource untuk scheduling;
- snapshot/restore untuk data control plane.
