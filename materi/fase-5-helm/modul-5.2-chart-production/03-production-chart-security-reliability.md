# 03 — Security, Reliability, dan Observability Preparation

## Tujuan

Menilai chart sebagai bagian dari supply chain dan reliability boundary, bukan hanya kumpulan template YAML.

## Security Review

Periksa setiap release:

- image repository dan digest berasal dari source yang disetujui;
- chart version, dependency, dan provenance dapat ditelusuri;
- ServiceAccount memiliki permission minimal;
- Pod Security, securityContext, capabilities, filesystem, dan network policy sesuai policy;
- ConfigMap tidak dipakai untuk credential;
- secret reference menunjuk mekanisme yang disetujui;
- rendered output dan release metadata tidak menjadi artifact publik.

`helm template` dapat menghasilkan object Secret jika chart memakai secret reference. Perlakukan hasil render sebagai data sensitif dan redaksi sebelum penyimpanan.

## Reliability Review

Periksa:

- replica dan PDB tidak saling bertentangan;
- requests/limits sesuai node capacity;
- readiness/startup/liveness probe memiliki tujuan berbeda;
- `terminationGracePeriodSeconds`, preStop, dan rollout strategy sesuai shutdown behavior;
- Service selector dan port stabil;
- storage class, PVC, backup, dan restore boundary jelas;
- topology spread/anti-affinity dipakai bila availability memerlukannya;
- external dependency memiliki timeout dan failure behavior.

Jangan menambahkan replica atau PDB tanpa capacity dan quorum review.

## Observability Preparation

Chart dapat menyediakan labels/annotations untuk scrape, ServiceMonitor, dashboard reference, atau logging metadata. Ini belum mengaktifkan telemetry jika collector/operator belum tersedia.

Sebelum deploy stack observability, review:

- resource footprint dan node capacity;
- storage, retention, compaction, dan backup;
- cardinality dan label policy;
- network path dan authentication;
- namespace dan ownership;
- upgrade/rollback behavior;
- impact terhadap workload existing.

Deployment observability harus memakai cluster disposable/approved dan evidence terpisah. Fase 7 membahas stack dan SLO secara lebih mendalam.

## CI Gate Konseptual

```text
chart lint → schema validation → template per environment
→ dependency/version/provenance check → rendered diff review
→ helm test disposable → package/sign/publish
→ approval → promotion reference immutable
```

Artifact CI harus minimum dan redacted. Jangan menyimpan merged secret values, kubeconfig, registry token, atau raw `helm get all`.

## Release Evidence

Evidence minimum yang aman:

```text
commit SHA
chart version/app version
image digest reference
values file identifier (bukan secret value)
target environment/namespace/release
render/lint/test status
revision/status
health gate summary
approval/change reference
```

Failure evidence harus mencantumkan class dan scope tanpa membocorkan data. “Deployed” tanpa target/time/output bukan evidence yang cukup.

## Rollback Decision

Rollback dipertimbangkan jika:

- readiness atau smoke test gagal;
- error rate melewati threshold;
- resource saturation atau scheduling failure;
- security regression;
- dependency incompatibility.

Jangan rollback buta ketika migration data sudah irreversible. Hentikan promotion, lindungi data, dan ikuti recovery runbook.

## Acceptance Criteria

- [ ] Security, supply chain, resource, probe, storage, PDB, dan dependency review tersedia.
- [ ] Observability footprint dan status runtime dibedakan.
- [ ] CI gate menghasilkan artifact minimum/redacted.
- [ ] Release evidence dapat ditelusuri tanpa secret.
- [ ] Rollback decision mempertimbangkan data migration dan external side effect.

## Catatan SRE

Reliability chart adalah kontrak antara aplikasi dan platform. Default yang tampak nyaman dapat menjadi outage ketika capacity, failure mode, atau dependency berubah.

## Kaitan

Fase 6 membawa artifact ini ke GitOps; Fase 7 menyediakan telemetry yang diperlukan untuk memvalidasi reliability.

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**
