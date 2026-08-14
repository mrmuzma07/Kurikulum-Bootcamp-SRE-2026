# 02 — Ad-hoc Command dan Playbook YAML

## Tujuan

Mengenali modul Ansible dan menyusun playbook kecil yang dapat direview sebelum mutation.

## 1. Ad-hoc untuk Preflight

```bash
ansible all -i inventories/lab/hosts.yml -m ping --limit worker1
ansible k3s_servers -i inventories/lab/hosts.yml -m ansible.builtin.setup --limit cp1
```

Gunakan ad-hoc untuk pertanyaan kecil dan observasi. Package/service mutation harus memakai playbook versioned, review, check mode, dan scope disposable.

## 2. Play, Task, Module

```yaml
---
- name: Pastikan baseline lab tersedia
  hosts: lab_nodes
  become: true
  gather_facts: true
  serial: 1
  tasks:
    - name: Pastikan paket curl ada
      ansible.builtin.package:
        name: curl
        state: present
      tags: [packages]

    - name: Buat marker konfigurasi
      ansible.builtin.copy:
        content: "managed-by=ansible\n"
        dest: /var/lib/sre-lab-marker
        owner: root
        group: root
        mode: "0644"
      notify: Catat perubahan baseline
      tags: [marker]

  handlers:
    - name: Catat perubahan baseline
      ansible.builtin.debug:
        msg: "baseline marker berubah; review evidence sebelum service reload"
```

Contoh hanya memakai nilai non-secret. `serial: 1` mengurangi blast radius, tetapi tidak menjamin availability bila aplikasi tidak memiliki replica/PDB.

## 3. Condition, Loop, Register

```yaml
- name: Pastikan package baseline
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop: "{{ baseline_packages }}"
  when: ansible_facts.os_family in ['Debian', 'RedHat']
```

Validasi `baseline_packages` melalui defaults/vars yang dapat direview. `register` boleh dipakai untuk keputusan berikutnya, tetapi jangan mencetak object yang mungkin mengandung credential. Gunakan `changed_when`/`failed_when` hanya bila semantics command dipahami.

## 4. Check, Diff, Limit, Tags

```bash
ansible-playbook -i inventories/lab/hosts.yml playbooks/common.yml \
  --check --diff --limit lab_nodes --tags packages
```

Sebelum runtime, review inventory graph, target, module behavior, dan kemungkinan diff sensitif. `--check` dapat berbeda dari eksekusi nyata dan beberapa module memiliki dukungan terbatas.

## Troubleshooting

- YAML indentation error: parse dengan linter/syntax check, jangan langsung menjalankan.
- Handler tidak berjalan: pastikan task memiliki `notify` dan perubahan benar-benar terjadi.
- Loop mengubah terlalu banyak: gunakan `--limit`, daftar kecil, dan review variable.
- `changed=0` tidak sesuai harapan: bedakan desired state, current state, dan check mode prediction.

## Acceptance Checklist

- [ ] Playbook memiliki name, hosts, scope, dan task module yang eksplisit.
- [ ] Handler hanya dipanggil saat perubahan relevan.
- [ ] Condition dan loop tidak membocorkan data.
- [ ] Command eksekusi memiliki check/diff/limit.
- [ ] Tidak ada `shell`/`command` yang dipakai tanpa alasan dan idempotency guard.

## Catatan SRE

Prefer module deklaratif daripada shell command. Jika command diperlukan, dokumentasikan idempotency, exit code, `changed_when`, dan rollback; command yang “berhasil” sekali dapat menjadi drift pada rerun.

## Kaitan

Lanjutkan ke [Idempotency, Variables, Facts, Handlers](03-idempotency-variables-facts-handlers.md) dan [LAB-01](lab/LAB-01-inventory-ssh-ad-hoc.md).
