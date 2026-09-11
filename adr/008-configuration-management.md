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

## Control node specification

| Setting | Value |
|---|---|
| OS | Ubuntu Server **26.04.1 LTS** |
| vCPU | 2 (1 socket × 2 cores) |
| RAM | 2 GB |
| Disk | 20 GB, thin |
| Address | `10.10.10.30` static, CORE (VMnet2) |

**Amended from the original 24.04 LTS / 1 vCPU / 1 GB.** Both figures changed
before the build; recorded here rather than left in conversation.

**26.04.1 rather than 24.04.** 26.04 was initially reasoned against, because it
replaces core system utilities with Rust reimplementations and Canonical delayed
its own upgrade offer pending backports for rust-coreutils regressions. That
argument was weaker than it appeared: Ansible is Python, and the control node
ships Python or PowerShell to targets, so coreutils sit largely outside the
execution path. They matter for shell work at the ANS01 prompt, not for Ansible's
operation. Against a 2.8 GB re-download to avoid a hypothetical, the existing
media wins.

*Accepted cost:* 26.04 is months old and most Ansible documentation assumes 22.04
or 24.04. When something behaves unexpectedly — a package that is not where a
guide says, a changed Python packaging rule — the OS version is a legitimate early
suspect rather than a late one.

**2 vCPU rather than 1.** Ansible is I/O-bound and spends most of its life blocked
on WinRM round-trips, which was the original argument for one core. But CPU is
time-sliced rather than hard-allocated the way RAM is, so an idle vCPU costs
nearly nothing, and there was no constraint to optimise against. The installer,
`apt upgrade`, `ansible-galaxy` collection installs, and NTLM message encryption
across concurrent forks are all genuinely CPU-bound. Configured as one socket with
two cores — sockets imply NUMA boundaries on real hardware and count against
Windows Server licensing, so "one socket, N cores" is the habit that stays correct
when the guest is Windows.

## Placement

The control node can rewrite the domain controller's configuration. Anything that
can do that *is* tier zero, regardless of how small the VM looks. It must not sit
on a lower-trust segment where compromise of that segment would inherit control of
the domain. See `docs/topology.md`.

## Consequences

- Profile A3 becomes FW01 2 + DC01 3 + WS01 4 + ANS01 2 + SRV01 3 = 14 GB and
  10 vCPU. RAM is within the ~16 GB ceiling; vCPU exceeds the ~8 meaningful cores
  and is accepted, because those VMs are not all busy at once. See
  `docs/resource-budget.md`.
- WinRM must be enabled and secured on every Windows target. A default
  `winrm quickconfig` creates an unencrypted HTTP listener on 5985; that is a
  bootstrap state, retired once ADR-010's CA can issue server authentication
  certificates.
- ANS01 → WS01 crosses CORE to CLIENT and therefore needs an explicit firewall
  rule once segmentation is tightened, scoped to ANS01's address rather than to
  `CORE net`. See `docs/firewall-policy.md`.
- Playbooks become the authoritative record of configuration. Anything changed by
  hand and not backported to code is lost at the next rebuild. This is a
  discipline cost, and it is the point.
- Credentials live in Ansible Vault, never in inventory. A leaked secret stays
  leaked — deleting it in a later commit does not remove it from history.
- The measured time of a full rebuild becomes a stated recovery time objective.
  Being able to state an RTO with evidence behind it is what this phase produces.
