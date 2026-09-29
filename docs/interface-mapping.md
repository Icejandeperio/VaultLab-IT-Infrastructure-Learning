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

Adapter model: **E1000e** (Intel 82574L).

## DC01

| VMX device | MAC | VMnet | Windows | Address |
|---|---|---|---|---|
| ethernet0 | `00:0c:29:16:99:84` | VMnet2 | Ethernet0 | 10.10.10.10/24 |

## WS01

| VMX device | MAC | VMnet | Windows | Address |
|---|---|---|---|---|
| ethernet0 | *not recorded* | VMnet3 | Ethernet0 | 10.10.20.139/24 (DHCP) |

Single adapter, so there is no ordering to get wrong and the mapping was never
in doubt. Recorded for completeness — a segment assignment stated in a document
is checkable, and one held only in the VM settings dialog is not. Capture the MAC
next time WS01 is powered on, or during the LTSC rebuild.

## ANS01

| VMX device | MAC | VMnet | Linux | Address |
|---|---|---|---|---|
| ethernet0 | `00:0c:29:17:45:22` | VMnet2 | **ens32** | 10.10.10.30/24 static |

Adapter model: **E1000** (Intel 82545EM). Read from the Ubuntu installer's network
screen and confirmed on the running system with `ip -br link`.

**`ens32`, not `ens33`.** Earlier drafts of runbook 05 assumed `ens33`, the name
commonly seen on VMware. Linux's predictable interface names are derived from the
PCI slot the virtual NIC occupies, so the name depends on this VM's hardware
layout, not on a convention. A netplan file written for `ens33` would have
configured an interface that does not exist, and the machine would have come up
with no network and no error. This is the verify-don't-assert rule paying off.

### vsock — a path outside the VMnet model

After the first package upgrade, ANS01's console login began printing:

```
Try contacting this VM's SSH server via 'ssh vsock%<id>' from host.
```

**vsock** is a host-to-guest communication channel that uses no network at all —
no IP address, no VMnet, no firewall. Recent systemd releases automatically expose
SSH over it and print that hint.

It extends no trust: only the hypervisor host can reach a guest over vsock, and the
host already controls the guest completely — its disk, its memory, its power
state. It is the same trust relationship as ADR-007's management adapter, one
layer lower. The Windows SSH client addresses hosts by name or IP, so it is not a
practical path from this host in any case.

Recorded because it is a listener on a tier-zero machine that was never
deliberately configured, and a path the network segmentation does not govern. If
Phase 4 hardening wants it removed, systemd provides a boot option to disable the
automatic vsock SSH listener — confirm the exact switch at that point rather than
from memory.

## Adapter models differ by guest profile

VMware chooses the virtual NIC model from the **guest OS type** selected when the
VM is created, and the choice is not exposed in the Workstation GUI:

| VM | Guest OS type | Adapter | Emulated chip | Guest driver |
|---|---|---|---|---|
| FW01 | FreeBSD 14 64-bit | E1000e | Intel 82574L | `em` |
| ANS01 | Ubuntu 64-bit | E1000 | Intel 82545EM | `e1000` |

An earlier version of this file labelled FW01's adapter "E1000 (Intel 82574L)."
The 82574L is VMware's **E1000e**; plain E1000 is the 82545EM, as ANS01 shows.
Both are Intel gigabit models and FreeBSD's `em` driver handles both, which is why
the mislabel never caused a fault — but a label that is wrong and harmless today
is the kind that misleads a later diagnosis.

VMXNET 3, VMware's paravirtualized adapter, is not selectable from the Workstation
GUI — it requires editing the VMX file directly.

## How to verify

**Host side:**

```powershell
Select-String -Path C:\Lab\VMs\FW01\FW01.vmx `
  -Pattern "^ethernet\d\.(vnet|connectionType|generatedAddress|virtualDev) " |
  ForEach-Object { $_.Line.Trim() } | Sort-Object
```

The `virtualDev` line, where present, names the adapter model.

**Guest side** — OPNsense console option 8 (Shell):

```sh
ifconfig | grep -E "^em|ether"
```

On a Linux guest such as ANS01:

```sh
ip -br link
```

Match the final octet of each MAC. On a multi-NIC VM, VMware derives all MACs
from the VM's UUID with an offset of 0, 10, 20, 30, 40, 50 — so only the last byte
differs.

## Why verify rather than assume

VMware assigns PCI slots in ascending `ethernetN` order and FreeBSD's `em` driver
numbers by PCI enumeration, so the mapping is deterministic **on this hypervisor**.
It stops being reliable on mixed physical hardware, where an onboard NIC and an
add-in card produce `re0`, `igb0`, `igb1` in driver and bus order rather than the
order the ports are labelled on the chassis.

The MAC address is the identifier that exists on both sides of the boundary.
Names are for humans; the stable identifier underneath is what you check against.
