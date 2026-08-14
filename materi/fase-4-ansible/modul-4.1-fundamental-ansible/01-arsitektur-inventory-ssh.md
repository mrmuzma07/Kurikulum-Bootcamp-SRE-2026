# 01 — Arsitektur, Inventory, dan SSH

## Tujuan

Memahami alur agentless dan menulis inventory yang membedakan identity host, management address, serta role.

## 1. Control Node dan Managed Node

```text
Mac ARM64 / CI runner (control node)
  └─ Ansible connection plugin → SSH → VM/server (managed node)
                                      ├─ OS/package/systemd
                                      └─ network/storage/firewall
```

Control node menyimpan playbook dan menjalankan modul melalui koneksi. Managed node tidak membutuhkan agent Ansible, tetapi membutuhkan SSH, Python/interpreter yang sesuai, user, dan privilege yang telah disiapkan. Keberadaan SSH tidak berarti host siap k3s.

## 2. Inventory Contract

Contoh placeholder YAML:

```yaml
all:
  vars:
    environment: lab
    ansible_user: <bootstrap-user>
    ansible_python_interpreter: /usr/bin/python3
  children:
    k3s_servers:
      hosts:
        cp1:
          ansible_host: <management-address-cp1>
          node_role: server
    k3s_agents:
      hosts:
        worker1:
          ansible_host: <management-address-worker1>
          node_role: agent
```

`ansible_host` adalah alamat koneksi management; `hostname` adalah identity guest; node IP/k3s advertise address dapat berbeda. Jangan mengisi private key, password, token, atau kubeconfig dalam inventory.

Inventory environment harus terpisah. Jangan memakai group `all` untuk mutation luas tanpa review. Metadata dari OpenTofu hanya diterima setelah stable key, role, environment, dan provisioning reference diverifikasi.

## 3. ansible.cfg

```ini
[defaults]
inventory = inventories/lab/hosts.yml
retry_files_enabled = false
interpreter_python = auto_silent
# host_key_checking tetap mengikuti kebijakan organisasi; jangan mematikan tanpa scope.

[privilege_escalation]
become = false
become_method = sudo
```

Konfigurasi di atas adalah contoh desain. Nilai privilege, SSH key, dan known-host policy harus disesuaikan dengan environment. Jangan menaruh secret di konfigurasi.

## 4. SSH Preflight

Static review memeriksa nama host, alamat, group, dan sumber credential. Runtime disposable dapat memulai dengan:

```bash
ansible-inventory -i inventories/lab/hosts.yml --graph
ansible all -i inventories/lab/hosts.yml -m ping --limit k3s_servers
```

Command tersebut belum dijalankan hanya karena ditulis. Catat target, context, waktu, dan output redacted jika benar-benar dieksekusi. Hentikan bila inventory menampilkan host production atau alamat yang tidak dikenali.

## 5. Failure Modes

| Gejala | Kemungkinan | Tindakan awal |
|---|---|---|
| UNREACHABLE | route/SSH/user/host key | verifikasi identity dan management path, jangan retry membabi buta |
| sudo gagal | policy/user/TTY | cek kontrak bootstrap, jangan menaruh password di log |
| Python missing | image minimal/interpreter | pilih interpreter yang disetujui atau bootstrap terbatas |
| host salah | inventory merge/group typo | stop, perbaiki inventory, ulangi graph |

## Acceptance Checklist

- [ ] Control node dan managed node dibedakan.
- [ ] Inventory memakai placeholder dan group role yang stabil.
- [ ] Management address tidak disamakan otomatis dengan node IP.
- [ ] Preflight memiliki `--limit` dan stop condition.
- [ ] Tidak ada credential literal.

## Catatan SRE

Koneksi adalah dependency, bukan health signal. `ping` Ansible hanya membuktikan modul dapat merespons; readiness OS, disk, time sync, firewall, dan k3s tetap memerlukan pemeriksaan terpisah.

## Kaitan

Lanjutkan ke [Ad-hoc dan Playbook YAML](02-ad-hoc-playbook-yaml.md) dan kontrak handoff [Modul 3.3](../../fase-3-opentofu/modul-3.3-konteks-onprem/03-opentofu-ansible-k3s-handoff.md).
