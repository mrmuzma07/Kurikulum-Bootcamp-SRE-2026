# Materi Bootcamp SRE 2026

Repositori materi belajar — pendamping dokumen [`KURIKULUM-BOOTCAMP-SRE-2026.md`](../KURIKULUM-BOOTCAMP-SRE-2026.md).

## Struktur

```
materi/
├── fase-0-fondasi/
│   ├── README.md                    ← overview fase
│   ├── modul-0.1-linux-shell/       ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 5 hari
│   │   ├── 01-navigasi-filesystem.md
│   │   ├── 02-permission.md
│   │   ├── 03-proses-systemd.md
│   │   ├── 04-text-processing.md
│   │   ├── 05-networking-cli.md
│   │   ├── 06-ssh.md
│   │   ├── 07-bash-scripting.md
│   │   ├── lab/
│   │   │   ├── LAB-01-orbstack-vm.md
│   │   │   ├── LAB-02-backup-script.md
│   │   │   └── LAB-03-debug-proses.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-0.2-networking/        ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 3 hari
│   │   ├── 01-dns-cidr.md
│   │   ├── 02-tcp-udp-tls.md
│   │   ├── 03-http-proxy.md
│   │   ├── 04-firewall-nat-arp.md
│   │   ├── lab/
│   │   │   ├── LAB-01-reverse-proxy.md
│   │   │   ├── LAB-02-dns-lokal.md
│   │   │   └── LAB-03-trace-koneksi.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-0.3-git/               ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 2 hari
│   │   ├── 01-git-fundamental.md
│   │   ├── 02-git-workflow.md
│   │   ├── lab/
│   │   │   └── LAB-01-gitlab-repo.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   └── (modul-0.3 selesai — Fase 0 lengkap)
├── fase-1-container-orbstack/
│   ├── README.md                    ← overview fase
│   ├── modul-1.1-container-fundamental/   ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 4 hari
│   │   ├── 01-konsep-container.md
│   │   ├── 02-dockerfile-best-practice.md
│   │   ├── 03-registry-gitlab.md
│   │   ├── 04-multi-arch-arm64.md
│   │   ├── lab/
│   │   │   ├── LAB-01-containerisasi-app.md
│   │   │   └── LAB-02-compose-stack.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   └── modul-1.2-orbstack-lab/             ⏳ menyusul
├── fase-2-kubernetes/               ⏳ menyusul
├── fase-3-opentofu/                 ⏳ menyusul
├── fase-4-ansible/                  ⏳ menyusul
├── fase-5-helm/                     ⏳ menyusul
├── fase-6-gitops/                   ⏳ menyusul
├── fase-7-observability/            ⏳ menyusul
├── fase-8-sre-practices/            ⏳ menyusul
└── fase-9-capstone/                 ⏳ menyusul
```

## Konvensi Penamaan

| Pola | Arti |
|---|---|
| `NN-judul.md` | Materi teori + praktik (baca berurutan) |
| `lab/LAB-NN-judul.md` | Lab step-by-step dengan acceptance criteria |
| `evaluasi/latihan.md` | Latihan harian per topik |
| `evaluasi/kuis-dan-jawaban.md` | Kuis pemahaman + kunci jawaban |

## Cara Pakai

1. Baca `README.md` modul → ikuti rencana hariannya.
2. Baca materi sambil **langsung praktik di terminal** (jangan cuma dibaca).
3. Kerjakan lab sampai semua ✅ acceptance criteria terpenuhi.
4. Tutup dengan latihan + kuis. Nilai kuis minimal 80% sebelum lanjut modul berikutnya.
