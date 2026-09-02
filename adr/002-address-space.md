# ADR-002 — Address space: 10.10.0.0/16

**Status:** Accepted

## Context

The lab needs six isolated segments with predictable, readable addressing that
will not collide with the host's real network.

## Decision

`10.10.0.0/16`, subdivided into one `/24` per segment. Third octet identifies the
segment; final octet identifies the host.

## Alternatives considered

**192.168.x.x** — rejected. Philippine ISP-supplied routers commonly occupy
`192.168.1.0/24` or `192.168.254.0/24`. A collision between the lab and the home
network produces routing failures that are difficult to diagnose because both
sides appear correctly configured.

**172.16.0.0/12** — functionally equivalent and equally valid. `10.10` was chosen
purely for readability.

## Consequences

- Segment identity is visible in the address. `10.10.30.20` is unambiguously the
  SEC segment without consulting a table.
- A `/16` allocation leaves 250+ unused segments for future expansion.
- VMware assigns itself `.1` on host-only networks by convention, colliding with
  the firewall's gateway address. Resolved by moving the host adapter to
  `10.10.10.5`. See `docs/troubleshooting-log.md`, entry 01.
