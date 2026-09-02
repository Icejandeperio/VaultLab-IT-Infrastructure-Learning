# Runbook 02 — FW01 (OPNsense)

**Result:** A six-interface router and firewall with routing, NAT, DNS, DHCP, NTP, and DMZ isolation

Build the firewall first. A domain controller built before its gateway exists has
no route, no DNS forwarder, and no time source, and gets configured twice.

## 1. Create the VM

New Virtual Machine → **Custom**:

| Setting | Value |
|---|---|
| Guest OS | FreeBSD 14 x64 |
| Name / location | `FW01` / `C:\Lab\VMs\FW01` |
| Firmware | UEFI |
| Processors | 2 |
| Memory | 2048 MB |
| Disk | 20 GB, SCSI, split, **not** preallocated |

At **Ready to Create** → **Customize Hardware**:

- Attach the OPNsense ISO, **Connect at power on**
- Remove Sound Card and USB Controller
- Add five more network adapters (see below)

## 2. Network adapters — order matters

The wizard offers only Bridged / NAT / Host-only. Those map to VMnet0, VMnet1,
and VMnet8. Custom VMnets are reachable only through the **Custom** dropdown in
Customize Hardware.

| Adapter | Setting | Becomes |
|---|---|---|
| 1 | NAT | WAN (em0) |
| 2 | Custom → VMnet2 | LAN / CORE (em1) |
| 3 | Custom → VMnet3 | OPT1 / CLIENT (em2) |
| 4 | Custom → VMnet4 | OPT2 / SEC (em3) |
| 5 | Custom → VMnet5 | OPT3 / RED (em4) |
| 6 | Custom → VMnet6 | OPT4 / DMZ (em5) |

Adapter type is chosen automatically from the guest OS and is **not** settable
from the GUI Advanced button — that only exposes MAC and bandwidth. E1000
(`Intel 82574L`) is what you get, hence `em*` interface names.

Then **VM → Settings → Options → Advanced** → tick **Disable side channel
mitigations**.

## 3. Install

1. Boot. Log in as **`installer`** / **`opnsense`** — the live-media account
2. Accept the default keymap
3. **Install (UFS)** — ZFS wants more RAM than allocated and 20 GB is small for it
4. Select the disk, confirm the destructive write
5. Set the root password. **Record it.**
6. Reboot — then **VM → Settings → CD/DVD → untick Connect at power on**

After reboot, log in as **`root`**. The `installer` account does not exist on the
installed system.

## 4. Assign interfaces — console option 1

```
Configure LAGGs?   n
Configure VLANs?   n
WAN     em0
LAN     em1
OPT1    em2
OPT2    em3
OPT3    em4
OPT4    em5
(blank) [Enter — only after em5 is accepted]
```

**Read the confirmation summary before answering `y`.** It must list six lines.
An empty Enter means "finished," and pressing it early silently drops the
remaining interfaces.

To verify the mapping rather than trust it, see `docs/interface-mapping.md`.

## 5. Address interfaces — console option 2, five times

| Menu | Interface | Address | DHCP server |
|---|---|---|---|
| 1 | LAN | 10.10.10.1 /24 | no |
| 2 | OPT1 | 10.10.20.1 /24 | **yes** — .100 to .200 |
| 3 | OPT2 | 10.10.30.1 /24 | no |
| 4 | OPT3 | 10.10.40.1 /24 | no |
| 5 | OPT4 | 10.10.99.1 /24 | no |

Leave WAN on DHCP.

Two prompts worth understanding rather than pattern-matching:

- **"Configure IPv4 via DHCP?"** asks whether *this interface* should obtain an
  address from elsewhere. Internal interfaces: no. WAN: yes.
- **"Enable the DHCP server?"** asks whether it should *hand out* addresses.
  CLIENT only.
- **"Upstream gateway"** — only WAN has one. For internal segments this box *is*
  the gateway. Press Enter.

The IPv6 address prompt wants an address, not yes/no. Typing `n` makes it repeat.

## 6. Web configuration

Browse to `https://10.10.10.1` from the host. Certificate warning is expected.
Log in as `root`.

**Wizard:**

- Hostname `FW01`, domain `corp.vaultlab.net`
- DNS `1.1.1.1`, `9.9.9.9`, **untick Override DNS**
- Leave DNSSEC off — the internal AD zone is unsigned and cannot be validated
  against the public root (see ADR-003)
- Timezone `Asia/Manila`
- WAN: DHCP, **untick Block RFC1918 Private Networks** and **Block bogon
  networks**. The WAN sits on 192.168.213.0/24, which *is* RFC1918 — leaving
  these ticked blocks the firewall's own uplink.
- LAN: `10.10.10.1/24`, **untick Configure DHCP server**
- Deployment type: **untick both** Optimize for Multiwan and Automatic DHCP/DNS
  registration. The latter would have Unbound create records for DHCP leases,
  competing with AD-integrated DNS once DC01 is authoritative.

**Interfaces** → rename OPT1–OPT4 to `CLIENT`, `SEC`, `RED`, `DMZ`. Enable each.
Apply after every change.

## 7. Firewall rules

Temporary allow-all on CLIENT, SEC, RED — each sourced from **its own** segment.
Description: `TEMP - allow all, replaced in Section 10`.

DMZ isolation, permanent, in this order:

1. **Block** — source `DMZ net`, destination `10.10.0.0/16`
2. **Pass** — source `DMZ net`, destination `any`

First match wins, so the block must sit above the pass.

See `docs/firewall-policy.md` for the rationale and the Phase 4 target ruleset.

## 8. Updates and backup

**System → Firmware → Status** → check and apply.

**System → Configuration → Backups → Download configuration** → save to
`C:\Lab\Exports\`. **This file must never enter git** — it contains password
hashes and private keys. It is covered by `.gitignore`.

Snapshot: `02-fw01-configured`.

## Verification

```
Interfaces → Diagnostics → Ping      target 1.1.1.1, source blank
Interfaces → Diagnostics → DNS Lookup   microsoft.com
```

- [ ] Six interfaces up with correct addresses
- [ ] `Interfaces → Overview` shows a connected route per segment, one default route on WAN
- [ ] Ping and DNS both succeed
- [ ] `ping 10.10.10.1` replies from the host
- [ ] DMZ rules present, block above pass
