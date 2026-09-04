# ADR-010 — SRV01 as an internal certificate authority

**Status:** Accepted

## Context

Several planned components need certificates issued by an authority the lab's
machines already trust: WinRM HTTPS listeners (Phase 2), LDAPS, WireGuard
certificate authentication (Phase 3), Always On VPN (Phase 6), and any internal
service that should not throw browser warnings.

Self-signed certificates provide encryption without authentication of the
endpoint. They defeat passive interception but not an active man-in-the-middle,
because nothing vouches for the certificate's subject. That is acceptable for a
bootstrap step on an isolated segment and unacceptable as a resting state.

## Decision

SRV01 — `10.10.10.11` on CORE, Windows Server 2025 Standard Evaluation, Server
Core, 2 vCPU / 3 GB / 60 GB — domain-joined, then built as an Enterprise Root CA
using ADCS. Key length 4096, SHA-256, ten-year validity.

## Alternatives considered

**Standalone Root CA** — does not require domain membership. Rejected: no AD
integration, so the root certificate must be distributed to every machine by
hand, and it supports neither certificate templates nor autoenrolment. An
enterprise CA publishes its root to the domain automatically via Group Policy,
and that automatic distribution is the feature being demonstrated.

**ADCS on DC01** — saves a VM and 3 GB. Rejected on two grounds. A domain
controller and a CA are both tier-zero, and collapsing them removes any ability
to demonstrate separation of roles. More practically, a CA on a DC cannot be
rebuilt independently of the forest, and rebuilding the forest from code is the
entire point of Phase 2.

**Self-signed certificates throughout** — zero infrastructure. Rejected: no
autoenrolment, no revocation, manual trust distribution, and it teaches the wrong
model.

**A public CA such as Let's Encrypt** — requires public DNS and
internet-reachable validation. The lab has neither by design.

## Consequences

- A CA is a tier-zero asset. Compromise of the CA is equivalent to compromise of
  the domain, because it can issue a certificate for any identity. It belongs on
  CORE and warrants the same care as the domain controller.
- ADCS misconfiguration is one of the most reliable privilege escalation paths in
  modern Active Directory. ESC1 through ESC8 describe template and enrolment
  misconfigurations that take a low-privileged user to domain admin. This is
  deliberate: build it correctly in Phase 2, then enumerate and exploit it with
  Certipy against GOAD-Light in Phase 5. The attack surface is part of the
  curriculum, not an accident.
- The correctness test is precise. On a domain member, after a policy refresh,
  `Get-ChildItem Cert:\LocalMachine\Root | Where-Object Subject -like "*VAULTLAB*"`
  must return the root without anyone having installed it there. If manual
  installation is required, the AD integration is not working and the CA has been
  built as a standalone in all but name.
- SRV01 brings the working profile to roughly 14 GB of ~16 GB. SRV01 and ANS01
  both shut down before Profile B (Wazuh) runs.
- SRV01 starts a third licensing clock — 180 days and one rearm from its own
  install date, with the 10-day activation grace applying separately. Record it in
  `docs/licensing-clock.md` from `slmgr /dlv` at build time rather than copying
  DC01's dates, and include SRV01 in the playbook set so it rebuilds with
  everything else.
- The self-signed WinRM listener created during Phase 2 bootstrapping must be
  retired once SRV01 can issue a server authentication certificate. That
  replacement is an acceptance criterion for the phase, not optional cleanup.
