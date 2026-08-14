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
│   └── modul-1.2-orbstack-lab/             ✅ tersedia
│       ├── README.md                ← panduan modul & rencana 1 hari
│       ├── 01-orbstack-machine.md
│       ├── 02-resource-limit-vs-docker-desktop.md
│       ├── 03-k3d-on-orbstack.md
│       ├── lab/
│       │   └── LAB-01-k3d-cluster.md
│       └── evaluasi/
│           ├── latihan.md
│           └── kuis-dan-jawaban.md
├── fase-2-kubernetes/
│   ├── README.md                    ← overview fase (Minggu 4–5)
│   ├── modul-2.1-konsep-k3d/             ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 3 hari
│   │   ├── 01-arsitektur-k8s.md
│   │   ├── 02-objek-inti.md
│   │   ├── 03-k3d-latihan-harian.md
│   │   ├── 04-kubectl-survival-kit.md
│   │   ├── lab/
│   │   │   ├── LAB-01-pod-deployment-service.md
│   │   │   └── LAB-02-ingress-configmap-secret.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-2.2-k3s-production/          ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 3 hari
│   │   ├── 01-k3s-arsitektur-install.md
│   │   ├── 02-k3s-multi-node-ha.md
│   │   ├── 03-disable-komponen-ingress.md
│   │   ├── 04-topologi-onprem.md
│   │   ├── lab/
│   │   │   ├── LAB-01-k3s-single-node-vm.md
│   │   │   └── LAB-02-k3s-multi-node-topologi.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-2.3-metallb/                ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 2 hari
│   │   ├── 01-kenapa-metallb.md
│   │   ├── 02-l2-mode-arp.md
│   │   ├── 03-bgp-mode-konsep.md
│   │   ├── 04-konfigurasi-integrasi-k3s.md
│   │   ├── lab/
│   │   │   ├── LAB-01-metallb-l2-expose.md
│   │   │   └── LAB-02-troubleshooting-arp.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-2.4-operasi-troubleshooting/ ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 2 hari
│   │   ├── 01-debug-workload.md
│   │   ├── 02-resource-hpa.md
│   │   ├── 03-etcd-backup-restore-k3s.md
│   │   ├── 04-upgrade-k3s-rolling.md
│   │   ├── lab/
│   │   │   ├── LAB-01-chaos-debug-workload.md
│   │   │   └── LAB-02-etcd-backup-restore-rolling-upgrade.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
├── fase-3-opentofu/                 ✅ tersedia
│   ├── README.md                    ← overview fase (Minggu 6–7)
│   ├── modul-3.1-dasar-opentofu/   ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 2 hari
│   │   ├── 01-konsep-iac-opentofu.md
│   │   ├── 02-hcl-resource-variable-output-provider.md
│   │   ├── 03-workflow-init-plan-apply-destroy.md
│   │   ├── 04-state-remote-import-drift.md
│   │   ├── lab/
│   │   │   ├── LAB-01-tofu-docker-web-server.md
│   │   │   └── LAB-02-state-minio-import-drift.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-3.2-modul-pola-produksi/ ✅ tersedia
│   │   ├── README.md                ← panduan modul & rencana 3 hari
│   │   ├── 01-arsitektur-modul-reusable.md
│   │   ├── 02-environment-workspace-dan-state.md
│   │   ├── 03-foreach-count-conditional-data-source.md
│   │   ├── 04-secret-handling-dan-ci-plan.md
│   │   ├── lab/
│   │   │   ├── LAB-01-modul-reusable-web-server.md
│   │   │   └── LAB-02-environment-promotion-dan-secret-safety.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   └── modul-3.3-konteks-onprem/      ✅ tersedia
│       ├── README.md                ← panduan modul & rencana 3 hari
│       ├── 01-provider-onprem-dan-boundary.md
│       ├── 02-simulasi-lokal-docker-libvirt-mock.md
│       ├── 03-opentofu-ansible-k3s-handoff.md
│       ├── 04-production-readiness-dan-evidence.md
│       ├── lab/
│       │   ├── LAB-01-simulasi-provisioning-lokal.md
│       │   └── LAB-02-handoff-ke-ansible-dan-k3s.md
│       └── evaluasi/
│           ├── latihan.md
│           └── kuis-dan-jawaban.md
│   └── (Fase 3 lengkap — Modul 3.1, 3.2, dan 3.3 tersedia)
├── fase-4-ansible/                  ✅ tersedia
│   ├── README.md                    ← overview fase & boundary OpenTofu → Ansible → k3s
│   ├── modul-4.1-fundamental-ansible/ ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-arsitektur-inventory-ssh.md
│   │   ├── 02-ad-hoc-playbook-yaml.md
│   │   ├── 03-idempotency-variables-facts-handlers.md
│   │   ├── lab/
│   │   │   ├── LAB-01-inventory-ssh-ad-hoc.md
│   │   │   └── LAB-02-playbook-common-idempotent.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-4.2-pola-produksi-ansible/ ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-roles-collections-repository.md
│   │   ├── 02-jinja-variables-vault.md
│   │   ├── 03-check-diff-limit-error-handling-ci.md
│   │   ├── lab/
│   │   │   ├── LAB-01-role-common-hardening.md
│   │   │   └── LAB-02-vault-safe-ci-validation.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   └── modul-4.3-ansible-k3s-onprem/ ✅ tersedia
│       ├── README.md
│       ├── 01-opentofu-inventory-readiness.md
│       ├── 02-k3s-install-upgrade-role.md
│       ├── 03-hardening-patching-rolling.md
│       ├── 04-backup-health-evidence-rebuild.md
│       ├── lab/
│       │   ├── LAB-01-handoff-k3s-multinode.md
│       │   └── LAB-02-rolling-patching-readiness.md
│       └── evaluasi/
│           ├── latihan.md
│           └── kuis-dan-jawaban.md
├── fase-5-helm/                     ✅ tersedia
│   ├── README.md                    ← overview fase & boundary k3s → Helm → GitOps
│   ├── modul-5.1-helm-fundamental/ ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-struktur-chart-values-template.md
│   │   ├── 02-render-install-upgrade-rollback.md
│   │   ├── 03-repository-chart-oci-gitlab.md
│   │   ├── lab/
│   │   │   ├── LAB-01-chart-skeleton-render-lint.md
│   │   │   └── LAB-02-release-lifecycle-values.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   └── modul-5.2-chart-production/ ✅ tersedia
│       ├── README.md
│       ├── 01-custom-chart-environment-values.md
│       ├── 02-hooks-tests-notes-upgrade.md
│       ├── 03-production-chart-security-reliability.md
│       ├── lab/
│       │   ├── LAB-01-custom-app-chart.md
│       │   └── LAB-02-observability-oci-test-rollback.md
│       └── evaluasi/
│           ├── latihan.md
│           └── kuis-dan-jawaban.md
├── fase-6-gitops/                   ✅ tersedia
│   ├── README.md                    ← overview fase & boundary CI → GitOps → ArgoCD
│   ├── modul-6.1-gitlab-ci-cd/      ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-pipeline-stages-jobs-rules.md
│   │   ├── 02-runner-artifacts-cache-environment.md
│   │   ├── 03-iac-ansible-pipeline-approval.md
│   │   ├── lab/
│   │   │   ├── LAB-01-ci-lint-test-build-push.md
│   │   │   └── LAB-02-iac-plan-approval-lab-run.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-6.2-argocd/             ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-gitops-source-of-truth-repo-structure.md
│   │   ├── 02-argocd-install-application-sync.md
│   │   ├── 03-applicationset-secrets-drift-selfheal.md
│   │   ├── lab/
│   │   │   ├── LAB-01-argocd-application-sync.md
│   │   │   └── LAB-02-drift-selfheal-applicationset.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   └── modul-6.3-end-to-end-flow/    ✅ tersedia
│       ├── README.md
│       ├── 01-promotion-flow-evidence.md
│       ├── 02-progressive-delivery-rollback.md
│       ├── lab/
│       │   ├── LAB-01-end-to-end-gitops-flow.md
│       │   └── LAB-02-canary-blue-green-introduction.md
│       └── evaluasi/
│           ├── latihan.md
│           └── kuis-dan-jawaban.md
├── fase-7-observability/            ✅ tersedia
│   ├── README.md
│   ├── modul-7.1-prometheus/        ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-arsitektur-scrape-exporter-promql.md
│   │   ├── 02-rules-alertmanager-retention-cardinality.md
│   │   ├── lab/
│   │   │   ├── LAB-01-prometheus-node-blackbox-scrape.md
│   │   │   └── LAB-02-promql-rules-alertmanager.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-7.2-alloy-telemetry-pipeline/ ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-alloy-components-metrics-logs-traces.md
│   │   ├── 02-otel-pipeline-daemonset-helm-debugging.md
│   │   ├── lab/
│   │   │   ├── LAB-01-alloy-metrics-remote-write.md
│   │   │   └── LAB-02-alloy-logs-otlp-pipeline.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-7.3-mimir/              ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-arsitektur-mimir-remote-write-object-storage.md
│   │   ├── 02-retention-ha-query-recording-rules.md
│   │   ├── lab/
│   │   │   ├── LAB-01-mimir-remote-write-query.md
│   │   │   └── LAB-02-mimir-retention-capacity-recovery.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   ├── modul-7.4-loki-tempo/          ✅ tersedia
│   │   ├── README.md
│   │   ├── 01-loki-labels-logql-redaction-retention.md
│   │   ├── 02-tempo-otlp-traces-correlation.md
│   │   ├── lab/
│   │   │   ├── LAB-01-alloy-loki-log-pipeline.md
│   │   │   └── LAB-02-tempo-trace-log-metrics-correlation.md
│   │   └── evaluasi/
│   │       ├── latihan.md
│   │       └── kuis-dan-jawaban.md
│   └── modul-7.5-grafana-alerting-slo/ ✅ tersedia
│       ├── README.md
│       ├── 01-grafana-use-red-dashboard-as-code.md
│       ├── 02-alerting-contact-point-routing-notification.md
│       ├── 03-slo-error-budget-burn-rate-runbook.md
│       ├── lab/
│       │   ├── LAB-01-dashboard-data-source-evidence.md
│       │   └── LAB-02-alert-firing-notification-failure-injection.md
│       └── evaluasi/
│           ├── latihan.md
│           └── kuis-dan-jawaban.md
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
"}① 天天中彩票未functions.Edit Ադրբեջary  (...)  code 