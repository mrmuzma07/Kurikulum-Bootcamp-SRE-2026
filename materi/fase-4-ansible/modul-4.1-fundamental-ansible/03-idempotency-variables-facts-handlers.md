# 03 — Idempotency, Variables, Facts, dan Handlers

## 1. Idempotency

Playbook idempotent memetakan current state ke desired state. Run pertama boleh `changed`; run berikutnya seharusnya tidak mengubah apa pun bila kondisi tetap sama. Idempotency tidak berarti task aman untuk semua host dan tidak berarti service sehat.

```yaml
- name: Pastikan service chrony aktif
  ansible.builtin.service:
    name: chrony
    enabled: true
    state: started
```

Nama service berbeda antar image. Validasi OS dan package manager terlebih dahulu; jangan menyalin contoh ke production tanpa mapping.

## 2. Variables dan Facts

Gunakan defaults untuk nilai aman dan dapat dioverride per environment:

```yaml
# roles/common/defaults/main.yml
common_packages:
  - ca-certificates
  - curl
common_timezone: UTC
```

Facts membantu memilih task:

```yaml
- name: Tampilkan fakta non-sensitif untuk evidence
  ansible.builtin.debug:
    msg:
      os_family: "{{ ansible_facts.os_family }}"
      architecture: "{{ ansible_facts.architecture }}"
```

Jangan mencetak seluruh facts pada log karena dapat memuat alamat, environment detail, atau data yang tidak diperlukan.

## 3. Handlers

Handler ideal untuk reload/restart setelah file konfigurasi berubah. Hindari restart tanpa perubahan karena dapat menyebabkan outage yang tidak perlu.

```yaml
- name: Tulis konfigurasi marker
  ansible.builtin.template:
    src: marker.conf.j2
    dest: /etc/sre-marker.conf
    mode: "0644"
  notify: Reload service lab

handlers:
  - name: Reload service lab
    ansible.builtin.service:
      name: sre-lab
      state: reloaded
```

Pastikan service memang mendukung reload dan siapkan fallback/stop condition bila reload gagal.

## 4. Error Behavior

`when`, `failed_when`, dan `changed_when` harus merepresentasikan semantics nyata. `ignore_errors: true` bukan recovery strategy; gunakan `block/rescue` pada Modul 4.2 dan simpan evidence redacted.

## 5. Verifikasi Rerun

Urutan yang aman pada disposable target:

```text
inventory graph → syntax check → check mode → approval → first run
→ health check → rerun → bandingkan changed/result → catat evidence
```

Jika runtime tidak tersedia, tulis hasil sebagai predicted behavior, bukan “berhasil idempotent”.

## Acceptance Checklist

- [ ] Desired state dan current state dijelaskan.
- [ ] Variables memiliki boundary environment.
- [ ] Facts yang dikumpulkan minimum dan non-secret.
- [ ] Handler tidak memicu restart tanpa perubahan.
- [ ] Error dan rerun memiliki stop condition.

## Catatan SRE

`changed=0` adalah sinyal convergence untuk task tertentu. SRE tetap membutuhkan probe service, dependency, latency, dan workload health untuk menyatakan sistem siap.

## Kaitan

Lanjutkan ke [LAB-02](lab/LAB-02-playbook-common-idempotent.md) dan Modul 4.2 tentang role/templating.
