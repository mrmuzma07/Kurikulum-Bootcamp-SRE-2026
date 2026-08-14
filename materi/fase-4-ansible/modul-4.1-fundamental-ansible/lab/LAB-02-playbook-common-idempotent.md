# LAB-02 — Playbook Common Idempotent

## Tujuan

Membuat baseline non-secret yang dapat diprediksi, menjalankan check mode, lalu merancang pembuktian rerun.

## Playbook Contoh

```yaml
---
- name: Baseline lab terbatas
  hosts: lab_nodes
  become: true
  gather_facts: true
  serial: 1
  tasks:
    - name: Pastikan marker
      ansible.builtin.copy:
        content: "managed-by=ansible\n"
        dest: /var/lib/sre-lab-marker
        mode: "0644"
      tags: [marker]

    - name: Pastikan package non-secret
      ansible.builtin.package:
        name: ca-certificates
        state: present
      tags: [packages]
```

## Langkah

1. Syntax review dan cek host/group.
2. Jalankan atau prediksi `--check --diff --limit <approved-lab-host>`.
3. Review perubahan dan approval.
4. Jalankan first run hanya pada disposable host.
5. Health check marker/package.
6. Rerun dengan limit yang sama dan bandingkan `changed`.

`changed=0` pada rerun membuktikan convergence task tertentu, bukan service atau k3s health.

## Failure Drill

- marker destination tidak writable;
- package manager unavailable;
- host kedua gagal setelah host pertama berubah.

Stop, catat partial state, dan jangan langsung menjalankan semua host ulang.

## Acceptance Criteria

- [ ] Playbook mempunyai scope, serial, tag, dan module state.
- [ ] Check mode dilakukan/didesain sebelum mutation.
- [ ] Rerun dan expected `changed=0` dijelaskan.
- [ ] Failure path memiliki stop condition.
- [ ] Tidak ada shell command yang menyimpan credential.

## Status Runtime

Jika `ansible-playbook` atau VM tidak tersedia, seluruh first run/rerun berstatus **belum diverifikasi**; cukup lakukan static review.
