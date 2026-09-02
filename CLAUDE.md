# Context for Claude Code

Read this before making changes in this repository.

## What this is

VAULTLAB is a segmented virtualization lab built on a single Windows 11 laptop
using VMware Workstation Pro. It is a learning and portfolio project. The owner is
a self-described beginner who explicitly wants to understand **what and why**, not
just receive commands to paste.

## How to work here

- **Explain reasoning before instructions.** Every procedure should say what it
  achieves and why that approach was chosen. Steps without explanation are a
  failure mode, not a shortcut.
- **Never invent file paths, cmdlet names, or menu locations.** Verify against the
  filesystem or ask. Three separate faults in this project came from asserted
  paths that did not exist.
- **Correct the docs when reality differs.** OPNsense enumerated interfaces as
  `em0`–`em5`, not `vmx0`–`vmx5`; the runbook was wrong and was amended. Do the
  same for any future divergence rather than leaving stale instructions.
- **Never commit secrets.** See `.gitignore`. The OPNsense `config.xml` export is
  the highest-risk file in the project — it contains password hashes and private
  keys. It must never enter git history.

## Hard constraints

| Constraint | Value |
|---|---|
| Usable VM RAM | ~16 GB — see `docs/resource-budget.md` |
| Lab storage ceiling | ~450 GB |
| Budget | Zero. Free tooling only, no subscriptions, no one-time purchases. |
| Windows Server 2025 eval | 180 days, **one** rearm available — see `docs/licensing-clock.md` |

VMs run in named profiles, never all at once. Check the resource budget before
proposing anything that adds a standing VM.

## Conventions

- Hostnames: `[ROLE][##]` — `FW01`, `DC01`, `WS01`, `SIEM01`. Max 15 chars, no hyphens.
- Addressing: `10.10.0.0/16`, one /24 per segment. See `docs/address-plan.md`.
- Domain: `corp.vaultlab.net`, NetBIOS `VAULTLAB`.
- Architectural decisions go in `adr/` before implementation, not after.
- Commit messages explain **why**, not what. See `docs/git-workflow.md`.
