# ADR-005 — Firewall platform: OPNsense

**Status:** Accepted

## Context

The lab needs a router and firewall terminating six segments, enforcing
inter-segment policy, and providing DHCP, DNS forwarding, and NTP.

## Decision

OPNsense 26.7 (FreeBSD-based), deployed as FW01 with six virtual NICs.

## Alternatives considered

**pfSense CE** — the more widely known option, and what most tutorials assume.
Rejected because the feature gap between the free Community Edition and the
commercial Plus edition continues to widen, Plus costs $129/year on non-Netgate
hardware, and pfSense CE requires AES-NI where OPNsense does not.

**VyOS** — excellent for routing and configuration-as-code, but the rolling
release is the only free build and the learning curve is steeper than warranted
for Phase 1.

**A Linux VM with nftables** — maximum learning value, minimum convenience. Kept
in reserve as a later exercise rather than the foundation.

## Consequences

- Fully open under a 2-clause BSD licence. No feature gating, no upgrade pressure.
- Two-week update cadence.
- Web UI accelerates Phase 1 at the cost of hiding some mechanics. Mitigated by
  documenting rule rationale in `docs/firewall-policy.md` rather than only
  clicking through.
- Suricata IDS is available in-platform for Phase 4 without additional cost.
