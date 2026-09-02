# Interface Mapping — FW01

Verified by correlating MAC addresses between the VMX file and `ifconfig` in the
guest, not assumed from ordering.

## FW01

| VMX device | MAC | VMnet | FreeBSD | OPNsense role | Address |
|---|---|---|---|---|---|
| ethernet0 | `00:0c:29:43:4f:18` | NAT (VMnet8) | em0 | WAN | DHCP → 192.168.213.128 |
| ethernet1 | `00:0c:29:43:4f:22` | VMnet2 | em1 | LAN (CORE) | 10.10.10.1/24 |
| ethernet2 | `00:0c:29:43:4f:2c` | VMnet3 | em2 | OPT1 (CLIENT) | 10.10.20.1/24 |
| ethernet3 | `00:0c:29:43:4f:36` | VMnet4 | em3 | OPT2 (SEC) | 10.10.30.1/24 |
| ethernet4 | `00:0c:29:43:4f:40` | VMnet5 | em4 | OPT3 (RED) | 10.10.40.1/24 |
| ethernet5 | `00:0c:29:43:4f:4a` | VMnet6 | em5 | OPT4 (DMZ) | 10.10.99.1/24 |

## DC01

| VMX device | MAC | VMnet | Windows | Address |
|---|---|---|---|---|
| ethernet0 | `00:0c:29:16:99:84` | VMnet2 | Ethernet0 | 10.10.10.10/24 |

## How to verify

Adapter type is E1000 (`Intel 82574L`), which is why interfaces enumerate as
`em*` rather than `vmx*`. VMXNET 3 is not selectable from the Workstation GUI —
it requires editing the VMX file directly.

**Host side:**

```powershell
Select-String -Path C:\Lab\VMs\FW01\FW01.vmx `
  -Pattern "^ethernet\d\.(vnet|connectionType|generatedAddress) " |
  ForEach-Object { $_.Line.Trim() } | Sort-Object
```

**Guest side** — OPNsense console option 8 (Shell):

```sh
ifconfig | grep -E "^em|ether"
```

Match the final octet of each MAC. VMware derives all six from the VM's UUID with
an offset of 0, 10, 20, 30, 40, 50 — so only the last byte differs.

## Why verify rather than assume

VMware assigns PCI slots in ascending `ethernetN` order and FreeBSD's `em` driver
numbers by PCI enumeration, so the mapping is deterministic **on this hypervisor**.
It stops being reliable on mixed physical hardware, where an onboard NIC and an
add-in card produce `re0`, `igb0`, `igb1` in driver and bus order rather than the
order the ports are labelled on the chassis.

The MAC address is the identifier that exists on both sides of the boundary.
Names are for humans; the stable identifier underneath is what you check against.
