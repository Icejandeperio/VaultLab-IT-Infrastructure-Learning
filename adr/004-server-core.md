# ADR-004 — Windows Server: Core, not Desktop Experience

**Status:** Accepted

## Context

Usable VM memory is approximately 16 GB. Windows Server 2025 with Desktop
Experience consumes 5–6 GB idle; Server Core runs comfortably in 3 GB.

## Decision

All Windows Server instances use the Server Core installation option.

## Consequences

- Roughly 2–3 GB saved per server, which is the difference between running three
  and four VMs concurrently. On this host that is decisive, not a preference.
- Administration is PowerShell and remote management only. There is no Server
  Manager, no MMC, no Explorer. This is the intended outcome — it is how
  production servers are administered and it forces the skill.
- Disk footprint after install is approximately 6 GB versus 15+ GB.
- Some installers still render GUI windows on Core, because the Win32 graphics
  subsystem is partially present. Core means no desktop shell, not no graphics.
- Certain roles and third-party software require Desktop Experience. None
  currently planned do. If one appears, it gets its own VM rather than changing
  this decision.
