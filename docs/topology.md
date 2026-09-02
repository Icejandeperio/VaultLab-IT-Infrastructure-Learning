# Topology

## Logical

```mermaid
graph TD
    NET[Internet] --> NAT[VMware NAT · VMnet8<br/>gw 192.168.213.2]
    NAT -->|em0 WAN| FW[FW01 · OPNsense 26.7<br/>2 vCPU · 2 GB]

    FW -->|em1| CORE[CORE · 10.10.10.0/24]
    FW -->|em2| CLIENT[CLIENT · 10.10.20.0/24]
    FW -->|em3| SEC[SEC · 10.10.30.0/24]
    FW -->|em4| RED[RED · 10.10.40.0/24]
    FW -->|em5| DMZ[DMZ · 10.10.99.0/24]

    CORE --> DC01[DC01 · 10.10.10.10<br/>WS2025 Core · AD DS · DNS]
    CORE -.-> HOST[Windows host · 10.10.10.5<br/>mgmt only · ADR-007]
    CORE -.planned.-> SRV01[SRV01 · 10.10.10.11]
    CLIENT -.planned.-> WS01[WS01 · DHCP .100-.200]
    SEC -.planned.-> SIEM01[SIEM01 · 10.10.30.20 · Wazuh]
    RED -.planned.-> KALI01[KALI01 · 10.10.40.20]
    DMZ -.planned.-> TGT[Vulnerable targets]

    style DMZ fill:#5a1f1f,color:#fff
    style TGT fill:#5a1f1f,color:#fff
```

## The model

A physical firewall appliance is a box with several ethernet ports: one to the
ISP, the others to switches serving different parts of an office. Traffic between
those areas must pass through the firewall because no other wire connects them.

This is the same design in software:

- **A VMware "Network Adapter" = one physical port on the appliance.**
- **A VMnet = one physical switch.**

VMnet2 through VMnet6 are five switches with no wire between them. The only
device touching all of them is FW01. Every packet crossing from CLIENT to CORE
must traverse the firewall, where it can be inspected, permitted, or dropped.

Give two VMs the same VMnet and they talk directly — the firewall becomes
decorative. This is why each VM gets exactly one adapter on exactly one segment.

## Trust ordering

| Segment | Trust | Reasoning |
|---|---|---|
| CORE | Highest | Directory services. Compromise here is total. |
| SEC | High | Holds logs and evidence. Must not be reachable from what it monitors. |
| CLIENT | Medium | User workstations. Assumed to be the initial foothold. |
| RED | Low | Attack tooling. Deliberately hostile. |
| DMZ | None | Deliberately vulnerable. No egress, no lateral movement. |
