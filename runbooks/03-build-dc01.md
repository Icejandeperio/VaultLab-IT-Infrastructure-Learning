# Runbook 03 — DC01 (Windows Server 2025 Core)

**Result:** An activated, correctly addressed domain controller hosting `corp.vaultlab.net`

**Status:** Sections 1–5 complete. Section 6 (promotion) pending.

Do not install Windows Server until ready to build. The 180-day clock starts at
installation, not first use. See `docs/licensing-clock.md`.

## 1. Create the VM

New Virtual Machine → **Custom**:

| Setting | Value |
|---|---|
| Guest OS | Microsoft Windows → Windows Server 2025 (take the exact match if offered) |
| Name / location | `DC01` / `C:\Lab\VMs\DC01` |
| Firmware | UEFI, Secure Boot on |
| SCSI controller | **LSI Logic SAS** — LSI Logic is unsupported on Server 2025, and Paravirtualized SCSI has no in-box Windows driver |
| Processors | 2 |
| Memory | 3072 MB |
| Disk | 60 GB, split, **not** preallocated |
| Network | **Custom → VMnet2** |

**Decline VMware Easy Install.** At the first wizard screen choose *"I will
install the operating system later"*, then attach the ISO afterward. Easy Install
automates the setup answers, defaults to Datacenter with Desktop Experience, and
hides the edition and partitioning steps.

**Customize Hardware:** attach ISO with Connect at power on, remove Sound Card and
USB Controller.

## 2. Two settings before first boot

**VM → Settings → Options** (a different tab from Customize Hardware):

- **Advanced** → tick **Disable side channel mitigations**
- **VMware Tools** → **untick "Synchronize guest time with host"**

The time sync setting is not optional. VMware pushing host time into the guest
fights the Active Directory time hierarchy, and Kerberos rejects authentication
past five minutes of clock skew. The failure presents as a wrong password on a
correct password.

## 3. Install

- **Windows Server 2025 Standard Evaluation** — the entry **without** "(Desktop
  Experience)". Wrong choice here means reinstalling; the edition cannot be
  changed afterward.
- No product key
- **Custom: Install Windows only (advanced)**
- Select the 60 GB unallocated space. Let Windows create the EFI, MSR, and
  primary partitions itself.
- Diagnostic data: **`1` (Required)**. Blank opts into the fuller telemetry.
- Set the Administrator password. **Record it.**

You land at `C:\Users\Administrator>` with no desktop. That is a successful Server
Core install.

SConfig launches automatically. Type **`15`** to exit to PowerShell — SConfig is
fine for emergencies, but PowerShell is what is used in practice and what can be
automated later.

## 4. VMware Tools

**VM → Install VMware Tools** (requires the VM to be running). If the Windows ISO
is still attached, detach it first — one virtual drive holds one disc.

```powershell
Get-Volume                 # confirm the CD-ROM is VMware Tools, ~135 MB
Get-ChildItem D:\          # confirm the installer filename
D:\setup.exe               # setup.exe, not setup64.exe on current ISOs
```

Click through Typical → Install → Finish. Reboot.

## 5. Networking and activation

```powershell
Get-NetAdapter    # confirm the alias, usually Ethernet0

New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.10.10 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.10.1
```

Type these on one line. A trailing space after a backtick breaks line
continuation invisibly.

DNS points at the firewall for now; it changes to `127.0.0.1` after promotion.

**Verify in layers** — each tests a different thing, and the first failure tells
you where to look:

```powershell
ping 10.10.10.1                 # segment and gateway
ping 1.1.1.1                    # routing and NAT
Resolve-DnsName microsoft.com   # name resolution
```

**Activate** — this is the 10-day deadline:

```powershell
slmgr /ato
slmgr /dlv
```

Expect `License Status: Licensed` and ~259,200 minutes (180 days). Activation
requires working DNS; run it after the checks above, not before.

If `/ato` returns `0xC004E028`, a request is already in flight — wait, then check
`/dlv` rather than firing again.

**Rename, then reboot:**

```powershell
Rename-Computer -NewName DC01 -Restart
```

Rename **before** promotion. A DC's name is embedded in the AD database, DNS SRV
records, and Kerberos SPNs; renaming afterward is a substantial operation.

Snapshot: `02-dc01-base`.

## 6. Promotion — pending

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest -DomainName "corp.vaultlab.net" -DomainNetbiosName "VAULTLAB" -InstallDns -Force
```

Record the DSRM password. Losing it costs the forest.

## 7. Post-promotion — pending

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1
Set-DnsServerForwarder -IPAddress 10.10.10.1 -PassThru
Add-DnsServerPrimaryZone -NetworkID "10.10.10.0/24" -ReplicationScope Domain

w32tm /config /manualpeerlist:"10.10.10.1,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time

dcdiag /v
```

`dcdiag` must pass every test before continuing to WS01.

## Verification

- [ ] `hostname` returns `DC01`
- [ ] `slmgr /dlv` shows Licensed, 180 days
- [ ] `Get-NetIPConfiguration` shows 10.10.10.10, gw 10.10.10.1
- [ ] All three connectivity layers succeed
- [ ] Snapshot `02-dc01-base` exists
