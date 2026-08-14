# LAB-01 — Dashboard, Data Source, dan Evidence

## Tujuan

Mereview dashboard USE/RED sebagai code dan, bila tersedia, membuktikan panel menampilkan data runtime.

## Prasyarat dan Guardrail

Baca [Dashboard as code](../01-grafana-use-red-dashboard-as-code.md).

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menulis Grafana password, datasource credential, raw dashboard export, atau PII ke repository/evidence.

## Lane A — Static Simulation

1. Buat panel inventory USE dan RED dengan query/reference, unit, threshold, variable allowlist, owner, runbook.
2. Review datasource UID/tenant reference, dashboard version, JSON schema, and GitOps ownership.
3. Periksa query cardinality, time range, no-data display, error display, and misleading aggregations.
4. Tulis evidence schema yang membedakan provisioned, rendered, data returned, and SLO relevant.

## Lane B — Optional Disposable Runtime

1. Verify Grafana context/namespace, datasource reference, tenant, and synthetic workload.
2. Provision dashboard via approved GitOps/Helm path after diff review.
3. Verify dashboard JSON loaded and each selected panel query returns bounded data.
4. Capture panel title/query reference/time range/series summary/timestamp; redact values if sensitive.
5. Check cross-signal link where configured; cleanup/rollback disposable dashboard.

Dashboard loaded tanpa panel data tetap **belum diverifikasi**.

## Evidence Template

```text
dashboard_revision: <reference>
datasource: <type-and-redacted-reference>
panel_results: <bounded-summary>
render_status: <summary>
query_window: <bounded-window>
status: <verified-or-belum-diverifikasi>
```

## Acceptance Criteria

- [ ] USE/RED panel inventory lengkap.
- [ ] Datasource/tenant/variable bounded.
- [ ] JSON non-secret dan ownership jelas.
- [ ] Panel data evidence terpisah dari provisioning.

## Troubleshooting

Panel kosong: cek datasource/tenant/query/time range/ingestion. Panel lambat: cek cardinality/range/step. UI accessible bukan SLO evidence.
