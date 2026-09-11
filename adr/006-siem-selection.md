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

## Amendment — the sizing figures need re-checking before Phase 5

**The resource numbers above are Security Onion 2.4 figures, and 2.4 is at end of
life.** Confirmed: 2.4 reaches EOL on 1 October 2026, and the 3.x line has
shipped — 3.0.0 in March 2026, 3.1.0 in May, 3.2.0 in July. Ubuntu and Debian
were officially removed as supported base systems in 3.0; 3.x runs on Oracle
Linux 9 only.

A changed base OS and three feature releases mean the Eval node and Import node
requirements may no longer be what this ADR states — and those numbers are
precisely what the rejection of Security Onion as the standing platform rests on.

**Before building anything in Phase 5**, read the current hardware requirements
from the 3.x documentation and record what they actually say. If the Import node
has grown past what the budget allows, that is a decision to revisit rather than
discover mid-build.

The decision itself — Wazuh standing, Security Onion on demand — is not disturbed
by this. Wazuh's role is agent-based FIM and compliance scanning, which Security
Onion does not replace at any size. What could change is whether the on-demand
Import node remains affordable at all, and if it does not, the Phase 5 PCAP
analysis approach needs a different answer.

Flagged rather than resolved because Phase 5 is distant and the figures would go
stale again before then. The action is to verify at build time, not to write
numbers here that will be wrong by the time they are used.
