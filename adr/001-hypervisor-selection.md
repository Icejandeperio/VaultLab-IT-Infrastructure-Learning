# ADR-001 — Hypervisor: VMware Workstation Pro 26H1

**Status:** Accepted

## Context

The lab runs on a single Gigabyte G5 KF laptop that is also the daily driver
machine. A hypervisor is required to host 5–6 VMs across six network segments,
at zero cost.

## Decision

VMware Workstation Pro 26H1, a Type 2 hypervisor running on the existing
Windows 11 host.

## Alternatives considered

**Proxmox VE bare metal** — better performance and closer to production practice,
but the machine has a single NVMe and is needed for daily use. Dual-boot would
mean losing the working environment every time the lab runs.

**VirtualBox** — genuinely free and open source, but weaker snapshot management
and networking for a six-segment topology.

**Hyper-V** — available on Windows 11 Pro, but enabling it forces VMware and
VirtualBox to run on top of Microsoft's hypervisor layer, degrading both.
Choosing it would foreclose the others.

## Consequences

- Type 2 carries a small performance penalty from the host OS layer. Acceptable.
- Broadcom made Workstation free for commercial, educational, and personal use in
  November 2024, with no license key required. No cost, no license risk.
- Requires Hyper-V, VBS, WSL2, and Memory Integrity to be disabled on the host.
  This measurably reduces host security and is documented as an accepted trade-off
  for a dedicated lab machine.
- Proxmox experience is deferred. It can be run nested inside Workstation later,
  accepting the performance cost, or on separate hardware if acquired.
