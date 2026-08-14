# 02 — Runbook MetalLB, Topology, Evidence, dan Postmortem

## 1. MetalLB IP Tidak Responding

- **Symptom:** LoadBalancer IP `<approved-service-ip>` tidak menjawab atau ARP tidak stabil.
- **Scope/severity:** service, VLAN, node/speaker, client segment, dan customer impact.
- **Preconditions:** context/cluster/service verified; target bukan production kecuali incident approval; current change/freeze checked.
- **Read-only first checks:** Service/Endpoints summary, speaker/controller status, address pool allocation summary, ARP table dari approved observer, VLAN/route/MTU, node health, recent changes, and external DNS reference.
- **Decision tree:**
  1. IP tidak dialokasikan → inspect Service/MetalLB policy dan pool ownership.
  2. IP allocated tetapi ARP tidak terlihat → inspect L2 VLAN/speaker/ARP path.
  3. ARP terlihat tetapi TCP gagal → inspect endpoints, kube-proxy/CNI, NetworkPolicy, health, and MTU.
  4. BGP mode → inspect peer/session/route policy tanpa mengubah router langsung.
- **Stop condition:** duplicate IP, unknown ownership, switch/firewall change tanpa approval, atau packet capture mengandung secret/PII.
- **Safe mitigation:** shift traffic ke approved endpoint, rollback reviewed pool/config revision, atau eskalasi network owner. Jangan mengosongkan pool atau menghapus service luas.
- **Rollback/data caveat:** perubahan route/advertisement dapat memutus lebih banyak service; external DNS/cache tidak pulih seketika.
- **Communication:** IC dan communications lead mengirim impact/update memakai status summary, bukan raw payload.
- **Post-check:** ARP/route, endpoint reachability, error/latency, alert recovery, and SLO window.
- **Evidence:** runbook revision, service/pool summary, query references, UTC timeline, decision, action, rollback, post-check.

## 2. Topology dan Dependency Documentation

Pisahkan tiga artefact:

- **As-built topology:** apa yang benar-benar terhubung sekarang, dengan source evidence dan timestamp.
- **Desired-state topology:** apa yang harus dikelola OpenTofu/Ansible/GitOps, dengan revision dan owner.
- **Dependency map:** DNS, NTP, registry, storage, identity, bastion, network, observability, database, dan external API.

Setiap artefact memiliki `owner`, `revision`, `last-reviewed`, `expires`, change reference, access/recovery boundary, dan evidence retention. Dokumentasi drift adalah signal; jangan melakukan manual production edit hanya agar diagram terlihat benar.

## 3. Evidence Retention dan Redaction

Evidence chain:

```text
policy/runbook revision
→ preflight/scope/approval
→ fault/action
→ telemetry/alert/paging
→ incident timeline/decision
→ mitigation/rollback/recovery
→ post-check/SLO outcome
→ postmortem/action-item verification
```

Simpan identifier, checksum summary, query reference, status, owner, dan timestamp. Redact token, password, kubeconfig, private key, PII, authorization header, raw logs, raw alert payload, raw backup, and rendered Secret. Retention harus memiliki access policy, expiry, dan deletion rule.

## 4. Postmortem Contract

```text
incident_id: <approved-incident-id>
revision: <git-revision>
status: <draft-approved-closed>
impact: <redacted-customer-and-service-impact>
detection: <alert-or-report-reference>
timeline_utc: <linked-redacted-timeline>
contributing_factors:
  - <system/process-factor>
what_went_well:
  - <observable-strength>
what_went_poorly:
  - <observable-gap>
missing_signal:
  - <telemetry/process-gap>
action_items:
  - id: <action-id>
    action: <specific-remediation>
    owner: <approved-team-or-role>
    due_date: <utc-date>
    verification: <test-or-evidence-reference>
slo_error_budget_decision: <approved-follow-up>
```

Postmortem harus blameless namun accountable. Jangan menulis nama individu sebagai root cause; identifikasi system/process, ownership action, due date, dan verification. Review apakah SLO, alert threshold, runbook, change policy, capacity, backup, atau training perlu diubah.

## Acceptance Criteria

- [ ] MetalLB runbook memiliki L2/BGP branch, ARP/VLAN/MTU checks, stop condition, rollback, dan data caveat.
- [ ] As-built, desired-state, dependency map, access boundary, retention, dan expiry dibedakan.
- [ ] Evidence chain dan redaction policy eksplisit.
- [ ] Postmortem memiliki impact, detection, timeline UTC, factors, signals, actions, owner, due date, verification, dan SLO decision.
