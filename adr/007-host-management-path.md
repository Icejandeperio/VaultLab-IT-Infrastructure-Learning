# ADR-007 — Host management path on CORE

**Status:** Accepted — technical debt, to be superseded in Phase 3

## Context

Managing the lab requires reaching the OPNsense web UI and the servers from the
Windows host. The host can either sit on a lab segment or reach the lab through
the firewall.

## Decision

The Windows host keeps a virtual adapter on **VMnet2 (CORE) only**, addressed
`10.10.10.5`. Host adapters are disconnected on VMnet3–VMnet6.

## Consequences

- This is an out-of-band management path that bypasses the firewall entirely. It
  would not exist in a production design.
- It is deliberately limited to one segment. A host adapter on every segment
  would make the firewall decorative — any host-to-VM traffic would never
  traverse it, and segmentation testing would produce false passes.
- The host adapter has **no default gateway** configured. Adding one would place
  a second default route on the laptop and break normal internet access.

## Supersession plan

Phase 3 replaces this with a WireGuard tunnel terminating on FW01's LAN
interface. The host adapter is then disconnected from VMnet2, all management
traffic passes through the firewall, and inter-segment policy applies uniformly.
This ADR is marked superseded at that point.
