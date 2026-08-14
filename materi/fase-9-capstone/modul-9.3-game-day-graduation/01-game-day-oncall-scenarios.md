# 9.3.1 — Game Day dan Skenario On-Call

## Incident Command Card

Setiap drill menunjuk:

- primary dan secondary on-call;
- incident commander (IC);
- communications lead;
- operations lead;
- service/platform owner;
- severity, acknowledgement target, escalation target;
- maintenance window, stop condition, dan rollback authority.

Timeline memakai UTC. IC menjaga scope dan keputusan; operations lead menjalankan action yang disetujui; communications lead mengirim status yang telah disaring; secondary menjaga handoff dan evidence.

## Siklus Incident

```text
detection
→ declaration + severity
→ read-only triage
→ runbook selection
→ bounded mitigation
→ rollback/recovery
→ telemetry/post-check
→ communication + closure
→ blameless postmortem
→ action-item verification
```

Tidak ada perubahan sebelum target, symptom, access, blast radius, dan stop condition dipahami. Satu iterasi hanya menguji satu fault.

## Scenario Matrix

| ID | Fault | Signal awal | Mitigasi bounded | Stop condition |
|---|---|---|---|---|
| GD-01 | worker node mati | NodeNotReady, pod pending | cordon/recovery sesuai runbook, jaga capacity | quorum/capacity terancam |
| GD-02 | disk penuh | disk pressure, write failure | hentikan growth terkontrol, cleanup yang disetujui | data deletion belum terverifikasi |
| GD-03 | bad release | error/latency naik setelah promotion | pause dan rollback desired state | migration/data caveat tidak jelas |
| GD-04 | certificate ingress expired | TLS failure/expiry alert | gunakan certificate recovery procedure | private key/access tidak tersedia |
| GD-05 | latency 10x | percentile latency dan trace dependency | isolate dependency/traffic, rollback bila perlu | load/traffic di luar scope |
| GD-06 | MetalLB/ARP conflict | LoadBalancer IP tidak responding | verify pool/ARP/VLAN, move scoped IP bila approved | conflict menyentuh network lain |
| GD-07 | destroy/rebuild | platform unavailable pada disposable scope | rebuild dari repository dan validate | target bukan disposable/backup hilang |

GD-03 memiliki target rollback kurang dari lima menit, tetapi hasil hanya boleh dicatat setelah waktu declaration, Argo/rollout, traffic, telemetry, dan recovery benar-benar diukur.

## Read-Only First Checks

Gunakan dashboard/query dan status summary untuk memastikan target, namespace, revision, node, dependency, alert, traffic, dan recent change. Jangan menampilkan raw logs, token, headers, PII, atau kubeconfig dalam timeline.

## Evidence dan Closure

Catat incident ID, revision, target, UTC timeline, decision, query reference, alert/paging receipt, mitigation, rollback/recovery, post-check, SLO/error-budget outcome, dan cleanup. Setiap incident yang benar-benar dijalankan menghasilkan postmortem redacted; tabletop diberi label tabletop.

## Static dan Runtime

Static lane menguji role play, decision tree, scenario card, dan postmortem template. Runtime lane hanya disposable dengan approval, one-fault scope, timeout, rollback, cleanup, dan evidence redaction. Jangan melakukan chaos pada production.

## Guardrail

> **Secret, token, PAT, kubeconfig, dan kredensial lain tidak boleh disimpan atau di-commit sebagai plain text.**

Jangan menyalin raw alert payload, raw log, PII, password, private key, webhook, atau credential ke incident channel maupun postmortem.

## Kaitan

- Readiness dan postmortem ada di [9.3.2](02-readiness-review-postmortem.md).
- Drill ada di [LAB-01](lab/LAB-01-game-day-drill.md).
- Rebuild/graduation ada di [LAB-02](lab/LAB-02-destroy-rebuild-graduation.md).
