# ADR-009 — DHCP on Windows Server with a relay on FW01

**Status:** Accepted

## Context

Phase 1 served DHCP from Dnsmasq on FW01, which has an interface directly on the
CLIENT segment. That was correct at the time — it removed a dependency on DC01
existing before a client could get an address.

It is not what enterprises run, and it exercises nothing about how DHCP crosses a
routed boundary.

DHCP discovery is a broadcast. A client with no address cannot unicast to a
server it has not yet found, so DHCPDISCOVER goes to `255.255.255.255`. Routers
do not forward broadcasts — that is the definition of a broadcast domain, and it
is what makes a router the boundary between two of them. A DHCP server therefore
needs either an interface on every segment it serves, or a relay agent on each
segment to rewrite the broadcast as a unicast packet addressed to the server.

## Decision

DHCP moves to Windows Server on DC01 (`10.10.10.10`, CORE). FW01 is configured
as a DHCP relay on the CLIENT interface. The Dnsmasq DHCP range is disabled
before the Windows scope is activated.

The Windows scope covers `10.10.20.0/24` — a subnet the server has no interface
on. That is exactly what the relay makes possible.

## Alternatives considered

**Leave Dnsmasq in place** — zero work, functionally adequate. Rejected because
it exercises none of the above, and because the AD integration is genuinely
useful rather than decorative.

**A second DC01 interface on CLIENT** — would work without a relay. Rejected
because multi-homing a domain controller creates DNS registration problems, and
because it sidesteps the concept the phase exists to teach.

## Consequences

- The relay inserts its own receiving-interface address into the `giaddr` field
  of the forwarded packet. That field is how the server knows which scope to
  answer from: the unicast source address is the relay, not the client, so
  without `giaddr` a server holding scopes for four subnets has no way to tell
  where the request originated. This is the entire mechanism behind
  `ip helper-address` on Cisco equipment.
- `Add-DhcpServerInDC` registers the server in Active Directory, and an
  unauthorised Windows DHCP server refuses to issue leases in a domain. That is a
  real security control built to blunt rogue DHCP — a classic man-in-the-middle
  vector, since whoever answers DHCP sets the client's default gateway and DNS
  server.
- AD integration also brings dynamic DNS registration tied to machine accounts,
  scope options managed alongside the directory, and failover between two servers
  later.
- Two DHCP servers on one segment produce intermittent, order-dependent failures
  that are unpleasant to diagnose. Dnsmasq's range must be disabled **before**
  the Windows scope is activated, not after.
- CLIENT now depends on DC01 being up to obtain an address. Existing leases
  survive a DC01 outage until they expire; new clients do not get one. This is
  the normal enterprise dependency and is accepted.
- CORE, SEC, RED, and DMZ remain static. The relay is configured on CLIENT only.
- Verification is specific: `ipconfig /all` on WS01 must show `DHCP Server` as
  `10.10.10.10`, not `10.10.20.1`. A lease renewed from the old server is
  otherwise indistinguishable from the client's point of view.
