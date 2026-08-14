# 01 — Roles, Collections, dan Struktur Repository

## Tujuan

Membuat boundary konfigurasi yang dapat dipelihara, diuji, dan dipromosikan.

## 1. Struktur Repository

```text
ansible/
├── ansible.cfg
├── inventories/
│   ├── lab/hosts.yml
│   └── staging/hosts.yml
├── playbooks/
│   ├── common.yml
│   └── k3s.yml
├── roles/
│   ├── common/
│   │   ├── defaults/main.yml
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   └── meta/main.yml
│   └── k3s/
└── collections/requirements.yml
```

Role memiliki ownership yang jelas. Inventory environment tidak dicampur agar `--limit` dan approval dapat direview.

## 2. Role Contract

Dokumentasikan input, supported OS, privilege, side effect, handler, output evidence, dan rollback. `defaults` adalah API yang aman; `vars` sebaiknya tidak menjadi tempat menyembunyikan override penting.

```yaml
# roles/common/defaults/main.yml
common_packages:
  - ca-certificates
common_manage_firewall: false
common_timezone: UTC
```

Flag firewall default `false` hanya contoh untuk mencegah perubahan mendadak. Mengaktifkannya membutuhkan network path dan recovery access review.

## 3. Collections dan Pinning

```yaml
collections:
  - name: ansible.posix
    version: ">=1.0.0,<2.0.0"
```

Version range harus mengikuti kebijakan lock yang disetujui. Verifikasi module behavior dan platform support; nama collection yang valid tidak membuktikan install atau execution.

## 4. Dependency dan Testing

Role dependency harus eksplisit dan tidak menyebabkan side effect tersembunyi. Molecule atau test harness dapat menguji converge/verify pada container atau VM, tetapi container bukan bukti systemd, SSH, kernel, storage, atau k3s VM production.

## Acceptance Checklist

- [ ] Environment inventory terpisah.
- [ ] Role contract, supported OS, side effect, dan rollback tertulis.
- [ ] Collection version dipin dan provenance dapat diaudit.
- [ ] Test lane dibedakan dari production-like runtime.

## Catatan SRE

Role adalah unit ownership. Role yang mengubah terlalu banyak subsystem menyulitkan blast-radius analysis dan rollback; pecah berdasarkan boundary host yang dapat diuji.

## Kaitan

Lanjutkan ke [Jinja2, Variables, Vault](02-jinja-variables-vault.md) dan [LAB-01](lab/LAB-01-role-common-hardening.md).
