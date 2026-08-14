# Latihan Modul 5.2 — Chart untuk Production

1. Rancang custom chart Deployment/Service/Ingress/ConfigMap non-secret.
2. Buat values staging dan production serta jelaskan setiap override.
3. Buat schema untuk image digest, replica, resources, dan required endpoint reference.
4. Review selector stability dan label contract.
5. Pilih securityContext, ServiceAccount, capabilities, dan NetworkPolicy secara defensible.
6. Rancang hook pre/post install/upgrade dengan weight dan deletion policy.
7. Buat `helm test` dan NOTES yang tidak membocorkan credential.
8. Susun upgrade/rollback matrix untuk migration, CRD, PDB, dan timeout.
9. Rancang OCI package/provenance/promotion evidence.
10. Analisis resource, storage, retention, dan scrape concerns untuk chart observability.

## Rubrik

Custom chart/values 25%, hooks/tests/upgrade 25%, security/reliability 25%, supply chain/evidence 25%. Minimal 80%; pelanggaran secret atau destructive guardrail menggugurkan.

## Kaitan

Review [Modul 5.1](../../modul-5.1-helm-fundamental/README.md), lalu lanjutkan ke Fase 6 untuk GitOps dan Fase 7 untuk observability.
