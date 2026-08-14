# Kuis dan Jawaban Modul 3.3

## Petunjuk

- Pilih satu jawaban terbaik untuk nomor 1–20.
- Jawab esai 21–30 dengan alasan, boundary, evidence, dan stop condition.
- Nilai minimal 80%.
- Jangan menggunakan credential nyata atau menjalankan mutation production untuk menjawab kuis.
- Pelanggaran secret/destructive guardrail menggugurkan penilaian.

## Pilihan Ganda

1. Peran utama provider OpenTofu adalah ...
   - A. Menjadi hypervisor
   - B. Menerjemahkan deklarasi ke API/library platform
   - C. Menyimpan password Ansible
   - D. Menggantikan k3s

2. Data source umumnya digunakan untuk ...
   - A. Menghapus object boundary lain
   - B. Membaca object yang dimiliki boundary lain
   - C. Menyimpan kubeconfig
   - D. Menjalankan playbook

3. Konfigurasi endpoint dan identity provider sebaiknya dikendalikan oleh ...
   - A. Child module tersembunyi
   - B. Root module/environment
   - C. Output k3s
   - D. Container image

4. CloudInit paling tepat dipahami sebagai ...
   - A. Full configuration management jangka panjang
   - B. First-boot guest initialization
   - C. Backend state
   - D. Load balancer

5. Docker simulation tidak membuktikan ...
   - A. `for_each` address
   - B. Label resource
   - C. VM boot/systemd/SSH production
   - D. Output contract

6. Provider libvirt membutuhkan perhatian khusus pada ...
   - A. QEMU/KVM, daemon, image, architecture
   - B. Kubernetes Secret saja
   - C. Git branch saja
   - D. DNS TTL saja

7. HTTP/REST API internal perlu diverifikasi untuk ...
   - A. Idempotency, auth, schema, retry, delete semantics
   - B. Warna terminal
   - C. Nama file README
   - D. Pod replica saja

8. `sensitive = true` berarti ...
   - A. State otomatis terenkripsi
   - B. Nilai disamarkan pada presentasi tertentu, bukan encryption
   - C. Credential aman di Git
   - D. Token boleh dimasukkan output

9. Contoh metadata yang boleh dihandoff adalah ...
   - A. k3s token
   - B. SSH private key
   - C. hostname, address, role, environment
   - D. kubeconfig admin

10. Readiness gate sebelum Ansible harus memeriksa ...
    - A. Hanya ping
    - B. Network, time, SSH, OS, storage, firewall, dan security
    - C. Hanya nama VM
    - D. Hanya jumlah file

11. Jika provider apply timeout, tindakan pertama yang tepat adalah ...
    - A. Apply ulang berkali-kali
    - B. Periksa task/object remote, log teredaksi, dan state
    - C. Destroy semua environment
    - D. Menghapus state tanpa backup

12. Rename key `worker-1` menjadi `worker-a` dapat menyebabkan ...
    - A. Address churn/replacement
    - B. State encryption
    - C. DNS otomatis benar
    - D. Quorum meningkat

13. Ansible memiliki ownership utama untuk ...
    - A. Hypervisor state
    - B. OS bootstrap, hardening, package, dan service
    - C. Provider lock
    - D. Git commit SHA

14. k3s sebaiknya dipasang ...
    - A. Sebelum IP ditentukan
    - B. Setelah host readiness lulus
    - C. Sebelum time sync
    - D. Saat plan masih direview

15. Plan production sebaiknya ...
    - A. Disalin dari dev
    - B. Dibuat pada target state/context production
    - C. Diabaikan jika HCL sama
    - D. Disimpan bersama token

16. Jika readiness kritis `not-verified`, automation harus ...
    - A. Tetap lanjut
    - B. Stop dan kumpulkan evidence
    - C. Memakai `-auto-approve`
    - D. Menghapus host

17. Evidence chain yang baik menghubungkan ...
    - A. Commit, lock, backend, plan, approval, handoff, health
    - B. Password dan token
    - C. Raw state di Git
    - D. Screenshot tanpa context

18. `state rm` adalah ...
    - A. Penghapusan remote object otomatis
    - B. Mutasi control-plane state yang perlu prosedur
    - C. Backup aplikasi
    - D. Health check k3s

19. Jika plan ingin replace control-plane aktif, lakukan ...
    - A. Apply langsung
    - B. Review migration/quorum/drain/rejoin plan
    - C. Reset cluster
    - D. Destroy state

20. Pada Mac ARM64, image container harus ...
    - A. Selalu `amd64` tanpa verifikasi
    - B. Diverifikasi manifest/compatibility dan emulation impact
    - C. Menggunakan `latest` untuk reproduksi
    - D. Memuat private key

## Kunci Pilihan Ganda

| No. | Jawaban | Alasan singkat |
|---:|:---:|---|
| 1 | B | Provider adalah adapter ke API/library. |
| 2 | B | Data source membaca boundary lain. |
| 3 | B | Root/environment mengontrol context. |
| 4 | B | CloudInit berfokus pada first boot. |
| 5 | C | Container bukan bukti VM/guest behavior. |
| 6 | A | Libvirt bergantung pada host virtualization stack. |
| 7 | A | Lifecycle API harus dipahami sebelum mutation. |
| 8 | B | Sensitive bukan encryption. |
| 9 | C | Metadata itu non-secret minimum. |
| 10 | B | Ping saja tidak cukup. |
| 11 | B | Hindari duplicate mutation dan klasifikasikan state. |
| 12 | A | Key map menentukan address. |
| 13 | B | Ansible mengonfigurasi host. |
| 14 | B | Host harus siap lebih dahulu. |
| 15 | B | State/context target berbeda. |
| 16 | B | Readiness adalah gate. |
| 17 | A | Chain dapat diaudit lintas boundary. |
| 18 | B | State operation mengubah control plane. |
| 19 | B | Replacement memerlukan migration plan. |
| 20 | B | ARM64 compatibility harus diverifikasi. |

## Esai dan Panduan Jawaban

21. **Bandingkan Proxmox, vSphere, libvirt, dan HTTP/REST.**

   Jawaban baik menyebut use case, endpoint/API, identity/permission, version compatibility, storage/network semantics, import/update/delete behavior, dan failure mode. Provider production tidak boleh diklaim teruji tanpa evidence.

22. **Mengapa provider bukan jaminan infrastructure siap?**

   Provider hanya mengirim/ membaca API. Capacity, IP, DNS, VLAN, datastore, image, guest boot, firewall, dan readiness tetap harus diperiksa. API success tidak sama dengan service health.

23. **Apa batas Docker simulation?**

   Docker membuktikan module shape, resource address, labels, ports, dan output. Docker tidak membuktikan VM kernel/systemd, SSH, CloudInit, disk/storage, hypervisor, L2, quorum, atau k3s.

24. **Rancang metadata handoff non-secret.**

   Sertakan stable key, hostname, management address, role, environment, module/provider version, provisioning reference, owner/lifecycle. Tolak password, private key, token, kubeconfig, dan secret provider.

25. **Tulis readiness gate sebelum Ansible.**

   Minimal identity, management/node network, DNS/route, time sync, SSH, OS/kernel/resource, storage, firewall, security baseline. Setiap gate memiliki evidence dan stop condition.

26. **Apa yang dilakukan saat provisioning timeout?**

   Stop retry, periksa remote task/object, provider logs yang diredáksi, state/backend/lock, dan tentukan apakah object partial exists. Recovery/import hanya dengan backup, plan, scope, lock, review, dan approval.

27. **Mengapa Ansible dan OpenTofu tidak boleh mengambil alih satu sama lain?**

   OpenTofu memiliki lifecycle object infrastructure/state; Ansible memiliki host configuration. Pencampuran membuat drift, race, dan ownership tidak jelas. Metadata menjadi contract, bukan secret channel.

28. **Sebutkan k3s readiness dan operational checks.**

   Bahas topology server/agent, quorum/datastore, static/stable IP, time, firewall/API path, version pin, CNI/ingress/load balancer, backup, upgrade, drain/PDB, context safety, node/workload health. Jangan reset cluster aktif sebagai shortcut.

29. **Susun evidence chain promotion.**

   Commit → module/provider lock → target backend/state key → plan → policy/approval → apply → metadata → host readiness → Ansible result → k3s health/SLO. Raw state/plan dan credential tidak masuk artifact biasa.

30. **Kapan automation harus berhenti?**

   Saat environment/identity/backend tidak cocok, provider unavailable/permission salah, IP/DNS/network belum siap, replacement belum punya migration plan, state lock/backup/evidence tidak siap, readiness belum diverifikasi, atau artifact membocorkan secret.

## Penilaian

- Pilihan ganda: 20 × 2 = 40 poin.
- Esai: 10 × 6 = 60 poin.
- Lulus minimal 80 poin.
- Pelanggaran secret, destructive operation, atau klaim runtime tanpa evidence: **tidak lulus** dan wajib mengulang guardrail.

## Kaitan

Setelah lulus, review ulang [README Modul 3.3](../README.md), [LAB-01](../lab/LAB-01-simulasi-provisioning-lokal.md), dan [LAB-02](../lab/LAB-02-handoff-ke-ansible-dan-k3s.md). Fase 4 — Ansible masih menyusul untuk eksekusi playbook dan handoff nyata.
