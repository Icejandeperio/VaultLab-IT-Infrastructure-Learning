# ADR-006 — SIEM: Wazuh standing, Security Onion on demand

**Status:** Accepted

## Context

Phase 4 requires log aggregation, file integrity monitoring, and compliance
scanning. The two obvious candidates have very different resource profiles.

## Decision

Wazuh all-in-one as the standing SIEM at 2 vCPU / 6 GB. Security Onion deployed
as an **Import node** (2 cores / 4 GB / 50 GB) only when PCAPs need analysis.

## Alternatives considered

**Security Onion as the standing platform** — the more capable option, with
Suricata, Zeek, and Elastic integrated. Rejected on resources: the minimum Eval
node is 4 cores, 8 GB, 200 GB and two NICs; a Standalone node wants 24 GB. On a
16 GB budget with 450 GB of storage that would consume the lab.

**Splunk Free** — capped at 500 MB/day ingest with no alerting, and the licensing
posture is unattractive for a portfolio project.

**Elastic stack assembled manually** — highest learning value, highest time cost.
Deferred.

## Consequences

- Wazuh provides agent-based FIM, log collection, MITRE ATT&CK mapping, and
  built-in CIS benchmark SCA — which serves the compliance-evidence objective
  directly.
- Network-based detection (Suricata, Zeek) is not standing. It arrives via the
  Import node workflow: capture during an exercise, then analyse.
- Security Onion 2.4 reaches end of life 1 October 2026; the 3.x branch runs on
  Oracle Linux 9 only. Plan against 3.x.
