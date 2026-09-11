# Runbook 01 — Host Preparation

**Applies to:** Gigabyte G5 KF, Windows 11
**Result:** A host ready to run VMware Workstation at full speed with the lab network fabric in place

Skipping section 2 is the single most common reason a home lab feels sluggish.

## 1. Firmware

1. Power on, press **Del** (or **F2**) repeatedly to enter BIOS
2. **Advanced** or **Chipset**
3. Enable **Intel Virtualization Technology (VT-x)**
4. Enable **Intel VT-d** — required for nested virtualization later
5. **F10** to save and exit

## 2. Remove competing hypervisor layers

Only one component can own the CPU's virtualization extensions at a time. If
Hyper-V or VBS is active, it takes that ownership at boot and VMware must run on
top of Microsoft's hypervisor through the Windows Hypervisor Platform API rather
than addressing the hardware directly.

**Windows features** — run `optionalfeatures.exe`, uncheck: Hyper-V (all),
Virtual Machine Platform, Windows Hypervisor Platform, Windows Sandbox, Windows
Subsystem for Linux. Reboot.

**Memory Integrity** — Windows Security → Device security → Core isolation
details → **Memory integrity** off. Reboot.

**Hypervisor launch** — PowerShell as Administrator:

```powershell
bcdedit /set hypervisorlaunchtype off
```

Reboot.

**Verify** — run `msinfo32`:

| Field | Required value |
|---|---|
| Virtualization-based security | **Not enabled** |
| "A hypervisor has been detected…" | **Absent** |
| Hyper-V - Virtualization Enabled In Firmware | **Yes** |

Recheck after every Windows feature update. They have been known to silently
re-enable Memory Integrity.

> **Trade-off:** this reduces host security. Acceptable on a dedicated lab
> machine. If the laptop holds client data or work credentials, reconsider.

> **This decision is load-bearing beyond this runbook.** WSL2 requires Virtual
> Machine Platform, which is the Hyper-V hypervisor under a different name.
> Turning WSL back on later reverses this section for **every VM in the lab**,
> permanently. ADR-008 rejected running Ansible from WSL on exactly these grounds
> and specifies a dedicated control node VM instead. Anything that asks for WSL,
> Windows Sandbox, Docker Desktop's WSL backend, or Android emulation is asking
> for the hypervisor back — treat it as an architectural change, not a feature
> toggle.

## 3. Power and thermal

- Always run plugged in
- Settings → System → Power → **Best performance**
- Thermal throttling under Profile B is expected on a thin chassis

## 4. Directory layout

```
C:\Lab\
├── VMs\        virtual machines
├── ISOs\       installation media
├── Exports\    OVA exports, config backups
└── Docs\       git repository
```

Exclude `C:\Lab\VMs` from Defender real-time scanning — Windows Security → Virus
& threat protection → Manage settings → Exclusions → Add folder. Defender
scanning virtual disk I/O causes severe, hard-to-diagnose performance loss.

## 5. VMware Workstation Pro

1. Create a free Broadcom account — there is no anonymous download
2. Download **Workstation Pro 26H1** for Windows
3. Verify before installing:
   ```powershell
   Get-FileHash -Path .\VMware-workstation-*.exe -Algorithm SHA256
   ```
4. Install. Choose the free-use option; no key required.

**Edit → Preferences:**

- **Memory** → Reserved memory `17000 MB`, allow some VM memory to be swapped
- **Workspace** → default VM location `C:\Lab\VMs`

Reserved memory is the hard ceiling on what VMware may claim from the host across
all running VMs. It is not the same number as the planning figure in
`docs/resource-budget.md`, and the two are not meant to match: the budget plans
around **~16 GB** with headroom retained, and the 17000 MB cap sits deliberately
above it so the planning figure binds first. If VMs are ever refused memory at
power-on, the budget has been exceeded and this cap is the thing that caught it.

## 6. Virtual network fabric

**Edit → Virtual Network Editor → Change Settings** (elevates).

| VMnet | Type | Subnet | Host adapter | VMware DHCP |
|---|---|---|---|---|
| VMnet2 | Host-only | 10.10.10.0/24 | **Connected** | **Disabled** |
| VMnet3 | Host-only | 10.10.20.0/24 | Disconnected | **Disabled** |
| VMnet4 | Host-only | 10.10.30.0/24 | Disconnected | **Disabled** |
| VMnet5 | Host-only | 10.10.40.0/24 | Disconnected | **Disabled** |
| VMnet6 | Host-only | 10.10.99.0/24 | Disconnected | **Disabled** |

Two things that are easy to get wrong:

1. **Uncheck "Use local DHCP service"** on all five. OPNsense serves DHCP. Two
   DHCP servers on one segment produces intermittent failures.
2. **Uncheck "Connect a host virtual adapter"** on VMnet3–VMnet6. Only CORE keeps
   one, per ADR-007. A host on every segment makes the firewall decorative.

## 7. Resolve the host address collision

VMware assigns its VMnet2 adapter `10.10.10.1`, which FW01 will also claim.

```powershell
Remove-NetIPAddress -InterfaceAlias "VMware Network Adapter VMnet2" -IPAddress 10.10.10.1 -Confirm:$false
New-NetIPAddress  -InterfaceAlias "VMware Network Adapter VMnet2" -IPAddress 10.10.10.5 -PrefixLength 24
```

No default gateway. Adding one places a second default route on the laptop and
breaks normal internet access.

If it reverts after reboot, change it at source in the Virtual Network Editor.

## Verification

- [ ] `msinfo32` shows VBS not enabled, no hypervisor detected
- [ ] Five VMnets present with correct subnets, DHCP disabled
- [ ] Host holds `10.10.10.5` on VMnet2 with no gateway
- [ ] `C:\Lab\VMs` excluded from Defender
