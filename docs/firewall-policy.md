# Firewall Policy — FW01

## Current state: permissive, intentionally

CLIENT, SEC, and RED carry temporary allow-all rules. This is deliberate. Building
segmentation before the domain works means every failure has two possible causes,
and debugging two layers at once is how people lose weekends.

DMZ isolation is **not** temporary. It was written before it was needed, because
the cost of a deliberately vulnerable host reaching the rest of the lab is not
worth the convenience of deferring it.

## Rules in place

| Interface | Action | Source | Destination | Status |
|---|---|---|---|---|
| LAN | Pass | LAN net | any | Default, retained |
| CLIENT | Pass | CLIENT net | any | **Temporary** |
| SEC | Pass | SEC net | any | **Temporary** |
| RED | Pass | RED net | any | **Temporary** |
| DMZ | **Block** | DMZ net | 10.10.0.0/16 | Permanent |
| DMZ | Pass | DMZ net | any | Permanent — egress only |

Rule order matters. pf evaluates top-down, first match wins. The DMZ block sits
above the DMZ pass, so anything aimed at lab space is dropped before it can reach
the permissive rule; only traffic destined elsewhere survives.

## Reading a rule

*"On interface X, traffic from source Y to destination Z is permitted."*

If the interface and the source do not refer to the same segment, question it. A
rule attached to SEC but sourced from `CLIENT net` will never match anything,
because no host on SEC holds a CLIENT address. It appears in the ruleset, looks
correct at a glance, and does nothing. See `troubleshooting-log.md`, entry 04.

## Phase 4 target: default deny on CLIENT

Replace the CLIENT allow-all with explicit rules, then observe what breaks using
**Firewall → Log Files → Live View**.

Required CLIENT → CORE flows for Active Directory:

| Service | Port | Protocol |
|---|---|---|
| DNS | 53 | TCP + UDP |
| Kerberos | 88 | TCP + UDP |
| RPC endpoint mapper | 135 | TCP |
| NetBIOS | 137–139 | TCP + UDP |
| LDAP | 389 | TCP + UDP |
| SMB | 445 | TCP |
| Kerberos password change | 464 | TCP + UDP |
| LDAPS | 636 | TCP |
| Global Catalog | 3268, 3269 | TCP |
| Dynamic RPC | 49152–65535 | TCP |
| NTP | 123 | UDP |
| DHCP relay to DC01 | 67, 68 | UDP |
| ICMP echo | — | ICMP |

The dynamic RPC range is why Active Directory is genuinely difficult to firewall
properly. Everything else CLIENT → CORE: block and log.

DHCP appears here from Phase 2 onward. The relay on FW01 forwards the client's
broadcast to `10.10.10.10` as unicast, so once CLIENT is tightened this becomes an
explicit flow rather than something the allow-all was quietly carrying. See
ADR-009.

## CORE → CLIENT: the management flow

Phase 2 introduces traffic in the direction this policy previously had no position
on. ANS01 on CORE must reach WS01 on CLIENT to configure it.

| Service | Port | Protocol | Source |
|---|---|---|---|
| WinRM HTTP | 5985 | TCP | ANS01 only — bootstrap, retired |
| WinRM HTTPS | 5986 | TCP | ANS01 only |

**Scoped to the control node's address, not to `CORE net`.** A rule permitting all
of CORE to reach CLIENT over WinRM would let a compromised DC01 or SRV01 reach
workstations by the same path. The flow exists because one specific machine needs
it; the rule should say so.

5985 is unencrypted and exists only until SRV01 can issue a server authentication
certificate. Remove it then — an HTTP management listener that outlives its
justification is exactly the kind of thing a Phase 4 compliance scan should find,
and finding it in your own lab is cheaper than finding it in production.

Note that ANS01 → DC01 and ANS01 → SRV01 are CORE → CORE. Both endpoints sit on
the same segment, so that traffic never reaches the firewall and no rule governs
it. Segmentation only constrains what crosses a boundary — a fact worth being
explicit about, because it is easy to assume a firewall rule is protecting a flow
that never passes through the firewall at all.

## Segment intent

| From → To | Policy |
|---|---|
| CLIENT → CORE | Explicit AD ports only |
| CLIENT → SEC | Deny |
| CLIENT → RED | Deny |
| CORE → CLIENT | Deny by default; WinRM from the control node only |
| CORE → SEC | Agent traffic to the SIEM, initiated from CORE |
| RED → CLIENT, DMZ | Permit — this is the attack path |
| RED → CORE | Deny by default; opened deliberately per exercise |
| SEC → anywhere | Deny outbound initiation; receives only |
| DMZ → anywhere internal | Deny, always |

CORE → SEC is listed because Wazuh agents in Phase 4 will run on CORE hosts and
report to SIEM01. The SEC row says SEC initiates nothing outbound — agents push to
the manager, the manager does not pull from agents, and that asymmetry is what
keeps the evidence store unreachable from the systems it monitors.
