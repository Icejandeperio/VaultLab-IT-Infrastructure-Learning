# ADR-008 — Configuration management: Ansible

**Status:** Accepted

## Context

DC01's Windows Server 2025 evaluation expires around 1 March 2027 with one rearm
available, giving a hard ceiling of roughly late August 2027. At that point the
forest is gone and everything configured by hand is lost.

Phase 1 was built entirely by hand. That proves the plumbing works; it does not
prove it can be reproduced. A lab that cannot be rebuilt is a lab that ends when
its licence does.

## Decision

Ansible, running from a dedicated Linux control node (ANS01) on CORE, managing
Windows targets over WinRM and Linux targets over SSH.

## Alternatives considered

**PowerShell DSC** — native to Windows, no control node required, no extra VM.
Rejected because Microsoft has deprioritised it, DSC v3 changed direction, and
the skill transfers to nothing outside Windows.

**Terraform / OpenTofu** — the wrong category of tool. Terraform provisions
infrastructure; Ansible configures what runs on it. They are complementary, and
VMware Workstation has no meaningful Terraform provider, so there is nothing here
for it to provision. OpenTofu enters in a later phase if cloud resources appear.

**Chef / Puppet** — agent-based, heavier on both control node and targets, and
both have shrinking mindshare in job postings.

**WSL on the Windows host instead of a control node VM** — rejected, and the
reason is structural rather than a preference. Only one component can own the
CPU's virtualization extensions at a time. Once Hyper-V is enabled it takes that
ownership at boot, and VMware Workstation must then run on top of it through the
Windows Hypervisor Platform API rather than addressing the hardware directly.
Runbook 01 section 4.2 disabled Hyper-V and VBS specifically to give VMware
direct ownership. WSL2 requires the Virtual Machine Platform feature, which is
that same hypervisor under a different name — enabling it reverses runbook 01 for
every VM in the lab, permanently, to save memory on one.

## Consequences

- ANS01 is a VM: Ubuntu Server 24.04 LTS, 1 vCPU, 2 GB, 20 GB, `10.10.10.30`
  static on CORE. 2 GB rather than 1 GB because `ansible-galaxy` installs and
  multi-host playbook runs push a 1 GB node into swap, and swapping on a shared
  NVMe degrades every other VM on the host.
- Placement on CORE rather than a management segment: the control node can
  rewrite the domain controller's configuration, and anything that can do that
  *is* tier zero. It must not sit on a lower-trust segment where compromise of
  that segment would inherit control of the domain.
- Profile A becomes FW01 2 + DC01 3 + WS01 4 + ANS01 2 = 11 GB, or 14 GB once
  SRV01 exists. Wazuh at 6 GB does not fit alongside SRV01 regardless. See
  `docs/resource-budget.md`.
- WinRM must be enabled and secured on every Windows target. A default
  `winrm quickconfig` creates an unencrypted HTTP listener on 5985; that is a
  bootstrap state, retired once ADR-010's CA can issue server authentication
  certificates.
- Playbooks become the authoritative record of configuration. Anything changed by
  hand and not backported to code is lost at the next rebuild. This is a
  discipline cost, and it is the point.
- Credentials live in Ansible Vault, never in inventory. A leaked secret stays
  leaked — deleting it in a later commit does not remove it from history.
- The measured time of a full rebuild becomes a stated recovery time objective.
  Being able to state an RTO with evidence behind it is what this phase produces.
