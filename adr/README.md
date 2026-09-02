# Architecture Decision Records

Each record captures a decision that would be expensive to reverse, the alternatives
considered, and the trade-offs accepted.

Format: **Context** (the situation), **Decision** (what was chosen), **Consequences**
(what it costs), **Status** (proposed / accepted / superseded).

Write the ADR before implementing, not after. The value is in forcing the reasoning
to be explicit while alternatives are still open.

| # | Decision | Status |
|---|---|---|
| 001 | Hypervisor: VMware Workstation Pro 26H1 | Accepted |
| 002 | Address space: 10.10.0.0/16 | Accepted |
| 003 | Domain namespace: corp.vaultlab.net | Accepted |
| 004 | Windows Server: Core, not Desktop Experience | Accepted |
| 005 | Firewall platform: OPNsense | Accepted |
| 006 | SIEM: Wazuh standing, Security Onion on demand | Accepted |
| 007 | Host management path on CORE (known deviation) | Accepted, to be superseded in Phase 3 |
