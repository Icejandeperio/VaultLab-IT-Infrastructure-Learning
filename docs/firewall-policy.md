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
| ICMP echo | — | ICMP |

The dynamic RPC range is why Active Directory is genuinely difficult to firewall
properly. Everything else CLIENT → CORE: block and log.

## Segment intent

| From → To | Policy |
|---|---|
| CLIENT → CORE | Explicit AD ports only |
| CLIENT → SEC | Deny |
| CLIENT → RED | Deny |
| RED → CLIENT, DMZ | Permit — this is the attack path |
| RED → CORE | Deny by default; opened deliberately per exercise |
| SEC → anywhere | Deny outbound initiation; receives only |
| DMZ → anywhere internal | Deny, always |
