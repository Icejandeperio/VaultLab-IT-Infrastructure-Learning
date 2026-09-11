# Runbook 04 — WS01 (Windows 11 domain-joined client)

**Result:** A domain-joined workstation on the CLIENT segment, proving the full
chain from DHCP through DNS, firewall traversal, Kerberos, and OU targeting.

**Status:** Complete on Enterprise Evaluation. Rebuild on LTSC 2024 pending —
see `docs/troubleshooting-log.md`, entries 11 and 12, and
`docs/licensing-clock.md`.

## 0. Media

Use **Windows 11 Enterprise LTSC 2024**, build `26100.1742.240906-0331`.

Do **not** use the standard Windows 11 Enterprise Evaluation. That image is
`26100.1.240331-1435` and ships with a fixed expiry already passed — evaluation
media carries the expiry baked into the build, not a timer starting at install.

The installed WS01 is nevertheless licensed until **3 December 2026**: the single
available rearm was applied and reset the clock to a full 90-day window. That
rearm is spent, there is no second one, and the correction to the original
"zero rearms, permanently expired" reading is entry 12. Verify the state on any
install with `slmgr /dlv` rather than inferring it — and re-check after a reboot,
because a rearm does not apply until the machine restarts.

Verify the SHA-256 against the download page before installing.

## 1. Prerequisite on FW01

WS01 gets its resolver from DHCP. If that resolver is the firewall, the domain
join fails: FW01 has no knowledge of `corp.vaultlab.net` and cannot answer for
the `_ldap._tcp.dc._msdcs` SRV records.

**Services → Dnsmasq DNS & DHCP → General**

- **Interfaces** must include **CLIENT**. Binding to LAN alone means no listening
  socket on `10.10.20.1`, and DHCP broadcasts on that segment are discarded.

**Services → Dnsmasq DNS & DHCP → DHCP ranges → CLIENT**

- Range `10.10.20.100`–`10.10.20.200`
- Domain `corp.vaultlab.net` — becomes the client's DNS search suffix

**Services → Dnsmasq DNS & DHCP → DHCP options**

- Option **6** (`dns-server`) = `10.10.10.10`, scoped to CLIENT

Set this in the scope, never on the client. Hardcoding DNS on WS01 papers over
the problem for exactly one machine.

> **From Phase 2 onward this section changes.** DHCP moves to Windows Server on
> DC01 with a relay on FW01, and the scope options are set there instead. See
> ADR-009 and runbook 05. If the rebuild happens after that migration, configure
> the scope on DC01 and skip this section entirely.

## 2. Create the VM

| Setting | Value |
|---|---|
| Guest OS | Microsoft Windows → Windows 11 x64 |
| Name / location | `WS01` / `C:\Lab\VMs\WS01` |
| Firmware | UEFI, Secure Boot on |
| Processors | 2 |
| Memory | 4096 MB |
| Disk | 64 GB, split, **not** preallocated |
| Network | **Custom → VMnet3 (CLIENT)** |

VMnet3, not VMnet2. WS01 must sit on the client segment so its traffic to DC01
crosses the firewall. Same segment as the DC and the segmentation is decorative.

Decline Easy Install — choose *"I will install the operating system later"* and
attach the ISO afterward.

## 3. TPM and encryption

Windows 11 requires TPM 2.0. VMware requires the VM to be encrypted before a
vTPM can be added, and the wizard offers this inline for a Windows 11 guest.

Choose **"Only the files needed to support a virtual TPM"** — `.nvram`, `.vmss`,
`.vmem`, `.vmx`, `.vmsn`. Not full disk encryption; that costs CPU on every I/O
and gains nothing here.

**Record the encryption password** as `WS01 VM encryption`. It is not the Windows
password and there is no recovery path. Credential Manager storage is convenient
but is tied to the Windows profile and lost on a host rebuild.

Encrypting the `.nvram` is the point of the exercise. That file holds the virtual
TPM's key storage, and a TPM whose sealed secrets sit in a plaintext file
provides no guarantee at all. This is what makes the vTPM meaningfully a TPM
rather than a checkbox — and it is what BitLocker and Credential Guard rely on in
Phase 5.

**Bypass alternative** (rejected here, documented for completeness): Shift+F10 at
the first setup screen, `regedit`, create `HKLM\SYSTEM\Setup\LabConfig` with
DWORD values `BypassTPMCheck`, `BypassSecureBootCheck`, `BypassRAMCheck` set to
`1`.

## 4. Install

At the network screen, take the offline path and create a **local account**
(`localadmin`). A Microsoft account brings telemetry, OneDrive sync, and a
recovery-key upload that has no place on a lab VM. Domain join replaces it
anyway.

If the offline option is hidden: **Shift+F10**, then `oobe\bypassnro`. The machine
reboots with the option available.

Decline every privacy toggle — location, Find my device, diagnostic data,
tailored experiences, advertising ID. These are CIS Benchmark controls you will
be scanning against in Phase 4.

Install VMware Tools.

**Then check the licence state before doing anything else:**

```powershell
slmgr /dlv
```

Read **License Status**, **Time remaining**, and **Remaining Windows rearm
count**, and record all three in `docs/licensing-clock.md`. Do not carry the
figures across from the previous install. Prefer **Time remaining** over
`slmgr /xpr` for the expiry — it is expressed in minutes and does not depend on
whether the machine's date format is month-first or day-first.

## 5. Verify DHCP before joining

```powershell
ipconfig /all
```

| Field | Expected |
|---|---|
| IPv4 Address | `10.10.20.100`–`.200` |
| Default Gateway | `10.10.20.1` |
| DHCP Server | `10.10.20.1`, or `10.10.10.10` after the Phase 2 migration |
| **DNS Servers** | **`10.10.10.10`** |
| DNS Suffix | `corp.vaultlab.net` |

The DNS line is the one that decides whether the join succeeds.

**If the address is `169.254.x.x`** — that is APIPA, meaning nothing answered the
DHCP request. See troubleshooting entry 10. Work outward: VM adapter setting,
then VMnet, then service running, then service binding.

Then confirm the locator chain:

```powershell
Resolve-DnsName corp.vaultlab.net
nltest /dsgetdc:corp.vaultlab.net
```

`nltest` is the real test — it performs DC location via SRV records and names the
DC it found. Returning `DC01` proves DHCP, DNS, the CLIENT→CORE firewall path,
and DC01's SRV registration all work simultaneously.

## 6. Rename

```powershell
Rename-Computer -NewName WS01 -Restart
```

> ### GATE
>
> ```powershell
> hostname
> ```
>
> **Must return `WS01` before joining.** `Rename-Computer` reports no error when
> it fails to take. See troubleshooting entry 08 — this cost a full DC rebuild.

## 7. Join

**Before joining a rebuilt machine**, delete the existing computer object on
DC01:

```powershell
Get-ADComputer WS01 -Properties whenCreated, DistinguishedName
Remove-ADComputer WS01
```

A fresh install under the same name is a different security principal with a
different SID. The old object holds a machine account password the new machine
does not know, and reusing the name without clearing it produces either a join
failure or an account the new machine cannot authenticate with.

```powershell
Add-Computer -DomainName "corp.vaultlab.net" `
  -OUPath "OU=Workstations,OU=VAULTLAB,DC=corp,DC=vaultlab,DC=net" `
  -Credential (Get-Credential) -Restart
```

Credentials: **`VAULTLAB\jcruz-adm`**. The domain prefix is required — without it
Windows authenticates against the local SAM, where that account does not exist,
and returns a misleading bad-password error.

`-OUPath` places the computer object directly in Workstations rather than the
default Computers container. Doing it at join time matters: a machine moved
afterward misses any GPO linked there until its next policy refresh.

**If it fails**, the error names the layer:

| Error | Layer |
|---|---|
| Cannot contact the domain | DNS or firewall path |
| Clock skew / time difference | Kerberos — see runbook 03 §7 |
| Access denied | Credentials or rights |
| Cannot find the OU | Typo in the distinguished name |
| Account already exists | Stale computer object not removed |

## 8. Verify

**On WS01**, logged in as `VAULTLAB\jcruz`:

```powershell
whoami                                  # vaultlab\jcruz, not ws01\jcruz
(Get-WmiObject Win32_ComputerSystem).Domain
klist
Resolve-DnsName ws01.corp.vaultlab.net
```

`klist` should show a TGT for `krbtgt/CORP.VAULTLAB.NET` plus a service ticket,
both `AES-256-CTS-HMAC-SHA1-96` with the `pre_authent` flag. That confirms
Kerberos rather than NTLM fallback — relevant in Phase 5, where NTLM fallback is
what gets relayed, RC4 tickets are what make Kerberoasting practical, and
accounts without pre-authentication are what get AS-REP roasted. Recognising the
healthy state now makes the unhealthy state visible later.

The DNS query confirms WS01 registered its own A record dynamically, using its
machine account to authenticate the update. This is why workstations can use DHCP
at all — nothing needs their address, only their name.

**On DC01:**

```powershell
Get-ADComputer WS01 -Properties * | Select-Object Name, DistinguishedName, OperatingSystem, LastLogonDate
```

`DistinguishedName` must read
`CN=WS01,OU=Workstations,OU=VAULTLAB,DC=corp,DC=vaultlab,DC=net`.
A populated `LastLogonDate` means it authenticated, not merely that an object was
created.

**Record the new lease address** in `docs/address-plan.md` if it differs from
`10.10.20.139`, and capture the adapter's MAC for
`docs/interface-mapping.md`.

Snapshot: `01-ws01-joined`.

## Verification

- [ ] Media is LTSC 2024, build verified against the download page
- [ ] `slmgr /dlv` read and the three figures recorded in `docs/licensing-clock.md`
- [ ] Dnsmasq bound to CLIENT with option 6 set to 10.10.10.10, **or** the Phase 2
      Windows scope active
- [ ] WS01 holds an address in the pool with DNS `10.10.10.10`
- [ ] `nltest /dsgetdc` returns DC01
- [ ] `hostname` returned `WS01` **before** the join
- [ ] Stale computer object removed before rejoining
- [ ] Computer object in the Workstations OU with a populated LastLogonDate
- [ ] `klist` shows an AES-256 TGT from DC01
- [ ] `ws01.corp.vaultlab.net` resolves to the DHCP-assigned address
- [ ] Address plan and interface mapping updated
- [ ] Snapshot `01-ws01-joined` exists
