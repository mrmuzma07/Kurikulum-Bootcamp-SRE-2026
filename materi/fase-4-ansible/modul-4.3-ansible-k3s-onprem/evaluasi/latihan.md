# Latihan Modul 4.3 — Ansible k3s dan On-Prem

1. Rancang metadata handoff non-secret.
2. Buat readiness matrix sembilan gate dan stop condition.
3. Jelaskan boundary OpenTofu, Ansible, dan k3s.
4. Susun topology tiga server dan dua agent dengan quorum note.
5. Rancang role k3s tanpa menulis token.
6. Tulis sequencing server bootstrap/join agent/health.
7. Buat hardening plan user, SSH, firewall, fail2ban, updates.
8. Susun rolling patch dengan `serial: 1` dan PDB/quorum gate.
9. Analisis failure worker setelah patch.
10. Buat evidence chain rebuild dan backup boundary.

## Rubrik

Handoff/readiness 25%, k3s topology 25%, hardening/rolling 25%, recovery/evidence/guardrail 25%. Minimal 80%; pelanggaran secret/destructive guardrail menggugurkan.

## Kaitan

Lanjutkan ke [Kuis](kuis-dan-jawaban.md) dan review Modul 2.2/2.4.
