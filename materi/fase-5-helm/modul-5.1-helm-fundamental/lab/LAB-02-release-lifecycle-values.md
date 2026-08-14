# LAB-02 — Release Lifecycle dan Values

## Tujuan

Menyusun dan, bila disetujui, menjalankan lifecycle install/upgrade/rollback secara terbatas dengan values environment.

## Static Lane

1. Buat matriks `values.yaml`, `values-staging.yaml`, dan `values-prod.yaml` tanpa secret.
2. Bandingkan rendered output per environment untuk replica, resources, image digest, Service, probes, dan PDB.
3. Rancang command install dan upgrade dengan context, namespace, `--wait`, dan timeout placeholder.
4. Rancang health gate: rollout, Pod condition, Service/Ingress, events, smoke test, dan telemetry.
5. Rancang revision history, rollback revision, data migration decision, dan uninstall boundary.

Contoh workflow yang belum dianggap execution evidence:

```bash
helm upgrade <release-name> <chart-path> \
  --namespace <approved-namespace> \
  --values values-staging.yaml \
  --wait --timeout <approved-timeout> --atomic
helm history <release-name> --namespace <approved-namespace>
helm rollback <release-name> <approved-revision> \
  --namespace <approved-namespace> --wait --timeout <approved-timeout>
```

## Runtime Lane Opsional

Sebelum mutation:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl -n <approved-namespace> get deploy,pod,svc
```

Lakukan hanya pada release disposable dengan approval dan access recovery. Setelah install/upgrade, kumpulkan status dan health evidence yang telah diredáksi. Jika upgrade pertama gagal, hentikan promotion dan klasifikasikan failure sebelum rollback.

## Guardrail

- Jangan memakai `--set` untuk secret.
- Jangan menggunakan context yang tidak diverifikasi.
- Jangan menganggap `deployed` sebagai SLO.
- Jangan menjalankan uninstall pada namespace yang ownership-nya tidak jelas.
- Jangan memakai `--force` atau destructive shortcut sebagai default.

## Failure Drill

- timeout readiness;
- image pull failure;
- immutable selector change;
- PDB menghalangi rollout;
- migration tidak kompatibel dengan rollback.

## Acceptance Criteria

- [ ] Values per environment memiliki diff yang dapat dijelaskan.
- [ ] Runbook memiliki preflight, approval, timeout, health gate, dan rollback.
- [ ] Revision dan artifact dapat ditelusuri.
- [ ] Runtime yang tidak dijalankan diberi status belum diverifikasi.

## Catatan SRE

Rollback release tidak otomatis mengembalikan data. Selalu bedakan artifact rollback, application rollback, dan data recovery.
