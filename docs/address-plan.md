# Address Plan

Space: `10.10.0.0/16`. One `/24` per segment. Third octet identifies the segment.

## Segments

| Segment | VMnet | Subnet | Gateway | Purpose | Internet |
|---|---|---|---|---|---|
| WAN | VMnet8 (NAT) | 192.168.213.0/24 | 192.168.213.2 | Uplink | Yes |
| CORE | VMnet2 | 10.10.10.0/24 | 10.10.10.1 | Domain controllers, file, CA | Yes |
| CLIENT | VMnet3 | 10.10.20.0/24 | 10.10.20.1 | Workstations | Yes |
| SEC | VMnet4 | 10.10.30.0/24 | 10.10.30.1 | SIEM and security tooling | Yes |
| RED | VMnet5 | 10.10.40.0/24 | 10.10.40.1 | Attack tooling | Yes |
| DMZ | VMnet6 | 10.10.99.0/24 | 10.10.99.1 | Vulnerable targets | **No — blocked** |

VMware's NAT service occupies `.2` on VMnet8 and reserves `.1` for the host
adapter. Do not assume the gateway is `.1` on that segment.

## Host assignments

| Host | Address | Segment | Method | Status |
|---|---|---|---|---|
| FW01 | `.1` on every segment | all | static | Built |
| DC01 | 10.10.10.10 | CORE | static | Built |
| SRV01 | 10.10.10.11 | CORE | static | Planned |
| WS01 | 10.10.20.100–200 | CLIENT | DHCP | Planned |
| SIEM01 | 10.10.30.20 | SEC | static | Planned |
| KALI01 | 10.10.40.20 | RED | static | Planned |
| Windows host adapter | 10.10.10.5 | CORE | static, **no gateway** | Built — see ADR-007 |

## Allocation convention

| Range | Use |
|---|---|
| `.1` | Segment gateway (FW01) |
| `.2`–`.9` | Reserved — infrastructure and management |
| `.10`–`.49` | Static servers and appliances |
| `.100`–`.200` | DHCP pool where enabled |
| `.201`–`.254` | Reserved — temporary and test systems |

## Static versus DHCP

Static addressing for anything other systems point at: domain controllers appear
in DHCP scope options, in member DNS configuration, and in firewall rules. A
lease expiry that moved the address would break all three.

DHCP for anything that only points outward. Workstations are found by name via
DNS, not by address.

## DHCP

Enabled on **CLIENT only**, served by FW01, pool `10.10.20.100`–`10.10.20.200`.

Phase 2 migrates this to Windows Server with a DHCP relay configured on FW01 —
the `ip helper-address` pattern. CORE, SEC, RED, and DMZ remain static.
