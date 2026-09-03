# Runbook 03 — DC01 (Windows Server 2025 Core)

**Result:** An activated, correctly addressed domain controller hosting `corp.vaultlab.net`

**Status:** Sections 1–8 complete. DC01 is a working domain controller.

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

> ### GATE — do not proceed until this passes
>
> ```powershell
> hostname
> ```
>
> **Must return `DC01`.** If it returns anything else, rename again and reboot.
>
> `Rename-Computer` reports no error when it fails to take. Promoting an
> unrenamed machine embeds the auto-generated hostname into the FSMO role
> assignments, the `_msdcs` SRV records, and every Kerberos SPN. Recovering
> costs a snapshot revert and a second promotion.
>
> See `docs/troubleshooting-log.md`, entry 08.

Snapshot: `02-dc01-base`.

## 6. Promotion

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Get-WindowsFeature AD-Domain-Services      # confirm Installed before continuing
```

That installs the binaries only. The server is still standalone.

```powershell
Install-ADDSForest -DomainName "corp.vaultlab.net" -DomainNetbiosName "VAULTLAB" -InstallDns -Force
```

| Parameter | Why |
|---|---|
| `-DomainName` | Forest root namespace. Permanent — see ADR-003. |
| `-DomainNetbiosName` | Would default to `CORP` from the first label. `VAULTLAB\jcruz` is more meaningful at a login prompt. |
| `-InstallDns` | AD is unusable without DNS. Clients locate DCs via SRV records such as `_ldap._tcp.dc._msdcs.corp.vaultlab.net`, not by address. |
| `-Force` | Suppresses confirmations. DSRM is still prompted. |

**DSRM password.** Directory Services Restore Mode is the escape hatch when AD
itself will not start — the local Administrator identity no longer exists
independently once promoted, so this is the only way into a broken DC. Record it,
and make it different from the Administrator password. Losing it costs the forest.

Two delegation warnings are expected: there is no parent `vaultlab.net` zone to
create a delegation record in, because the domain is not registered. The warning
text says so.

Reboots automatically. Log in as **`VAULTLAB\Administrator`**.

**Verify the promotion landed under the right name:**

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, PDCEmulator, RIDMaster, InfrastructureMaster
```

All three FSMO holders must read `DC01.corp.vaultlab.net`.

## 7. Post-promotion

### DNS self-reference

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1
```

DC01 is now authoritative for `corp.vaultlab.net`; FW01 knows nothing about that
zone. A DC resolving through someone else can fail to find its own SRV records.

`127.0.0.1` rather than `10.10.10.10` because loopback works even when the
adapter is down — which matters during boot, when services start before
networking is fully up.

### Forwarder

```powershell
Set-DnsServerForwarder -IPAddress 10.10.10.1 -PassThru
```

Authoritative for its own zone, forwards everything else. The chain is
client → DC01 → FW01 → public resolver.

### Reverse zone

```powershell
Add-DnsServerPrimaryZone -NetworkID "10.10.10.0/24" -ReplicationScope Domain
```

Forward DNS answers "what address is DC01." Reverse answers "what is
10.10.10.10." Most tutorials skip it; Kerberos, some SMB operations, and every
log tool in Phase 4 perform reverse lookups. Without it, Wazuh shows addresses
instead of hostnames in every alert.

### Time hierarchy

```powershell
w32tm /config /manualpeerlist:"10.10.10.1,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
Start-Sleep -Seconds 20
w32tm /resync
w32tm /query /status
```

The PDC Emulator is the authoritative clock for the domain; every member syncs
from it automatically. The PDC itself needs an external reference — hence FW01.
`0x8` means "client, standard NTP." `/reliable:yes` marks this machine as a
trusted source for the domain.

Kerberos rejects tickets past five minutes of skew. This is what stops WS01's
domain join failing with a misleading credential error.

**Expected result:** `Source: 10.10.10.1,0x8`, stratum 5–8, `ReferenceId`
showing `0x0A0A0A01`.

**If it reports `Source: Local CMOS Clock` and `LOCL`** — the configuration is
probably correct and the service simply has not accepted the peer. Diagnose in
layers before touching the firewall:

```powershell
w32tm /query /configuration | Select-String -Pattern "NtpServer|Type"
w32tm /stripchart /computer:10.10.10.1 /samples:3 /dataonly
```

If stripchart returns offsets, the network path is fine — restart the service and
resync. See `docs/troubleshooting-log.md`, entry 09.

### Verify

```powershell
dcdiag /v
dcdiag /test:DNS /v
Resolve-DnsName dc01.corp.vaultlab.net
Resolve-DnsName microsoft.com
```

`dcdiag /test:DNS` should show PASS across Auth, Basc, Forw, Del, Dyn, and RReg.
`Ext` shows `n/a` on an isolated forest.

Snapshot: `03-dc01-promoted`.

## 8. Directory structure

Do not place objects in the default `Users` container — it is not an OU and
cannot have Group Policy linked to it.

```powershell
$dn = "DC=corp,DC=vaultlab,DC=net"

New-ADOrganizationalUnit -Name "VAULTLAB" -Path $dn
$ou = "OU=VAULTLAB,$dn"

New-ADOrganizationalUnit -Name "Users"           -Path $ou
New-ADOrganizationalUnit -Name "Workstations"    -Path $ou
New-ADOrganizationalUnit -Name "Servers"         -Path $ou
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path $ou
New-ADOrganizationalUnit -Name "Groups"          -Path $ou
```

The `Workstations` OU must exist before WS01 joins, since the join specifies it
as the target.

**Accounts — one person, two identities:**

```powershell
$userOU = "OU=Users,OU=VAULTLAB,DC=corp,DC=vaultlab,DC=net"

New-ADUser -Name "Juan Cruz" -SamAccountName "jcruz" `
  -UserPrincipalName "jcruz@corp.vaultlab.net" -Path $userOU `
  -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true

New-ADUser -Name "Juan Cruz (Admin)" -SamAccountName "jcruz-adm" `
  -UserPrincipalName "jcruz-adm@corp.vaultlab.net" -Path $userOU `
  -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true

Add-ADGroupMember -Identity "Domain Admins" -Members "jcruz-adm"
```

Separating daily-use and privileged accounts is tier-zero administrative
practice. A single account used for both means one phished credential or one
malicious document running in a browser session grants domain-wide control. Build
the habit here, where it costs nothing.

## Verification

- [ ] `hostname` returns `DC01` **(checked before promotion)**
- [ ] `slmgr /dlv` shows Licensed, 180 days
- [ ] All three FSMO roles held by `DC01.corp.vaultlab.net`
- [ ] `dcdiag /v` passes every test
- [ ] `dcdiag /test:DNS /v` shows PASS across all applicable columns
- [ ] `w32tm /query /status` names `10.10.10.1` as source, not LOCL
- [ ] `Resolve-DnsName dc01.corp.vaultlab.net` returns 10.10.10.10
- [ ] `Resolve-DnsName microsoft.com` resolves through the forwarder
- [ ] OU structure exists, `Workstations` present
- [ ] Snapshots `02-dc01-base` and `03-dc01-promoted` exist
