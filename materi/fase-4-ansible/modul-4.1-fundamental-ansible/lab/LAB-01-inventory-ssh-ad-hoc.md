# LAB-01 — Inventory, SSH, dan Ad-hoc

## Tujuan

Menyusun inventory lab, memeriksa graph, dan membuktikan koneksi hanya pada target disposable yang disetujui.

## Guardrail

**Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.** Gunakan placeholder, jangan menyalin private key/password ke file atau output. Jangan menjalankan command pada host production.

## Jalur Statis

1. Buat `inventories/lab/hosts.yml` dengan group `k3s_servers` dan `k3s_agents`.
2. Isi `ansible_host` dengan `<management-address>` dan user dengan `<bootstrap-user>`.
3. Periksa duplicate key, role, environment, serta beda management/node address.
4. Prediksi hasil `ansible-inventory --graph` tanpa mengklaim command dijalankan.
5. Tulis readiness matrix: identity, SSH, sudo, Python, network, time, storage, firewall.

## Jalur Runtime Disposable

Preflight sebelum perubahan:

```bash
ansible-inventory -i inventories/lab/hosts.yml --graph
ansible all -i inventories/lab/hosts.yml -m ping --limit k3s_servers
ansible all -i inventories/lab/hosts.yml -m setup --limit k3s_servers --tree <redacted-evidence-dir>
```

Gunakan evidence directory di luar repository bila output berisi detail host. Pastikan target, environment, dan access path sudah diverifikasi. Task package/service belum dilakukan pada lab ini.

## Failure Drill

- ubah sementara group menjadi salah dan lihat apakah graph review menangkapnya;
- simulasikan host unreachable pada disposable target;
- klasifikasikan error sebagai route, SSH, sudo, interpreter, atau inventory.

Jangan melakukan retry massal atau mematikan host key checking sebagai shortcut.

## Acceptance Criteria

- [ ] Inventory graph dapat dibaca dan group role benar.
- [ ] Preflight dibatasi dengan `--limit`.
- [ ] Identity mismatch menghentikan lab.
- [ ] Evidence tidak berisi credential.
- [ ] Jika Ansible/SSH tidak tersedia, hasil ditulis **belum diverifikasi**.

## Troubleshooting dan Catatan SRE

Graph yang benar tidak menjamin connectivity. Perlakukan inventory sebagai input berisiko: review sebelum setiap playbook dan simpan ringkasan, bukan raw verbose output.
