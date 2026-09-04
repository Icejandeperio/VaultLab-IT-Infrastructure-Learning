# Troubleshooting Log

Real faults encountered during the build, with symptom, diagnosis, and fix.

This is the most valuable document in the repository. Tutorials show the path
where nothing goes wrong. This shows the reasoning applied when it did.

---

## 01 — Duplicate IP: host and firewall both claiming 10.10.10.1

**Symptom.** Web UI unreachable from the host after assigning `10.10.10.1` to
FW01's LAN interface. Ping behaviour inconsistent.

**Diagnosis.**

```powershell
Get-NetIPAddress -InterfaceAlias "*VMnet2*" -AddressFamily IPv4
# IPAddress : 10.10.10.1
```

VMware assigns its host virtual adapter the first usable address on a host-only
network by convention. OPNsense claims `.1` by the same convention. Neither
coordinates with the other.

**Fix.** Moved the host adapter to `10.10.10.5`, no gateway.

```powershell
Remove-NetIPAddress -InterfaceAlias "VMware Network Adapter VMnet2" -IPAddress 10.10.10.1 -Confirm:$false
New-NetIPAddress  -InterfaceAlias "VMware Network Adapter VMnet2" -IPAddress 10.10.10.5 -PrefixLength 24
```

**Lesson.** Duplicate addressing presents as *intermittence*, not failure. ARP
resolves to whichever device answers first, and that can vary per packet. A ping
that works then stops, with nothing in any log, is the signature.

---

## 02 — Interface assignment completed with only WAN assigned

**Symptom.** Console option 1 reached the confirmation summary showing a single
line: `WAN -> em0`. The other five interfaces were absent.

**Diagnosis.** An empty Enter at the LAN prompt means "finished assigning," and
is visually indistinguishable from a skipped field. Proceeding would have deleted
five interface assignments.

**Fix.** Answered `n` at the confirmation, re-ran option 1, entered `em0` through
`em5` in sequence, and only pressed Enter alone after `em5` was accepted.

**Lesson.** Text consoles give almost no feedback. The confirmation summary is
the check — read it before committing, every time.

---

## 03 — `installer` account rejected after installation

**Symptom.** Repeated `Login incorrect` with a password known to be correct.

**Diagnosis.** The username was wrong, not the password. The `installer` account
exists only on the live installation media. Once OPNsense is installed to disk
and booted from it, that account is gone.

**Fix.** Logged in as `root`.

**Lesson.** Live media and installed systems are two different operating systems
that happen to look identical. Same trap in Kali (`kali`/`kali` on live media
only) and Ubuntu (passwordless live user).

Compounding factor: Unix password prompts echo nothing. A screen that appears
frozen is usually just a password field.

---

## 04 — Firewall rules present but functionally inert

**Symptom.** Allow-all rules created on SEC and RED. Both segments would have
been silently isolated.

**Diagnosis.** Both rules were sourced from `CLIENT net` rather than their own
segments. A rule is evaluated on traffic arriving at its interface, but only
matches packets whose source falls in the specified network. Nothing on SEC holds
a `10.10.20.0/24` address, so the rule could never match. Traffic fell through to
the implicit deny.

**Fix.** Corrected the source on each rule to its own segment.

**Lesson.** The worst firewall bugs are visibly present and functionally inert.
Read every rule as a sentence — *"on interface X, from source Y, to destination
Z"* — and question any case where interface and source disagree.

---

## 05 — `setup64.exe` not found on VMware Tools ISO

**Symptom.** `CommandNotFoundException` for `D:\setup64.exe`, twice, under
different causes.

**Diagnosis, first occurrence.** `Get-Volume` showed `D:` holding
`SSS_X64FREE_EN-US_DV9` at 7.59 GB — the Windows Server ISO was still mounted.
One virtual drive holds one disc.

**Diagnosis, second occurrence.** After mounting Tools correctly,
`Get-ChildItem D:\` showed `setup.exe`, not `setup64.exe`. Recent Tools ISOs ship
a single 64-bit installer without the suffix.

**Fix.** Detached the Windows ISO via VM → Settings → CD/DVD → physical drive,
mounted Tools, then ran the correct filename.

**Lesson.** `CommandNotFoundException` with a full path means the path does not
resolve — not that the syntax is wrong. `Get-ChildItem`, `Test-Path`, and
`Get-Volume` answer the question directly. The filesystem is the authority, not
the instruction.

---

## 06 — Activation failure chain

**Symptom.** `slmgr /ato` failed three times with three different codes.

**Diagnosis and fix, in sequence.**

| Code | Meaning | Cause | Fix |
|---|---|---|---|
| `0x80072EE7` | `WININET_E_NAME_NOT_RESOLVED` | No IP, gateway, or DNS configured yet | Configured static addressing and DNS |
| `0xC004E028` | Activation request already in progress | Second `/ato` fired while the first was pending | Waited, then re-checked with `/dlv` |
| — | `License Status: Licensed` | | |

**Lesson.** An error that stays identical means the fix did nothing. An error
that *changes* means one layer was resolved and the next was uncovered. The
progression across three codes was the signal that the work was landing.

Also: `0x8007`-prefixed codes are standard Windows error codes in HRESULT form.
When activation, updates, or licensing fail in that family, check connectivity
before suspecting licensing.

---

## 07 — PowerShell line continuation swallowed parameters

**Symptom.** `New-NetIPAddress` prompted interactively for `IPAddress` despite it
being supplied on the continuation line.

**Diagnosis.** A trailing space after the backtick. The backtick must be the very
last character on the line; a space after it — invisible — breaks the
continuation, and the second line is parsed as a separate command.

**Fix.** Retyped as a single line without the backtick.

**Lesson.** Avoid backticks when typing interactively. Use them in scripts where
the file can be inspected, not at a live prompt.

---

## 08 — Domain controller promoted under an auto-generated hostname

**Symptom.** `Get-ADDomain` after a successful forest promotion returned:

```
PDCEmulator          : WIN-3KGG4AV0KPP.corp.vaultlab.net
InfrastructureMaster : WIN-3KGG4AV0KPP.corp.vaultlab.net
RIDMaster            : WIN-3KGG4AV0KPP.corp.vaultlab.net
```

The forest was built correctly, but under the Windows-generated hostname rather
than `DC01`.

**Diagnosis.** `Rename-Computer -NewName DC01 -Restart` had been issued but the
result was never verified. The rename did not take, and promotion proceeded on
the unrenamed machine. Once promoted, that hostname is embedded in the FSMO role
assignments, the `_msdcs` SRV records, and the Kerberos service principal names.

**Fix.** Reverted to snapshot `02-dc01-base` (pre-promotion), renamed, **verified
with `hostname`**, then promoted again. Fifteen minutes.

**Alternative rejected.** Renaming a live DC is supported:

```powershell
netdom computername <old> /add:DC01.corp.vaultlab.net
netdom computername <old> /makeprimary:DC01.corp.vaultlab.net
Restart-Computer
netdom computername DC01 /remove:<old>.corp.vaultlab.net
```

Worth knowing — in production with replication partners and joined members there
is often no choice. Rejected here because it leaves stale SRV records and SPNs
requiring cleanup, and the forest was ten minutes old.

**Lesson.** Verify a state change before building anything on top of it,
especially when the next step is irreversible. `Rename-Computer` reported no
error; the failure was silent. This is the same class of fault as entry 02 — a
command that appears to succeed while doing nothing.

`hostname` returning `DC01` is now a hard gate in runbook 03 before promotion.

**Secondary lesson.** This is exactly what the snapshot was for. Discovering the
same fault in Phase 4, with a client joined and Wazuh agents deployed, would have
forced the `netdom` path plus cleanup.

---

## 09 — w32time stuck on Local CMOS Clock despite correct configuration

**Symptom.** After configuring the PDC Emulator to sync from FW01:

```
Stratum: 1 (primary reference - syncd by radio clock)
ReferenceId: 0x4C4F434C (source name: "LOCL")
Source: Local CMOS Clock
The computer did not resync because no time data was available.
```

Stratum 1 normally means a GPS or atomic reference. This VM was claiming that
rank based on its own CMOS chip.

**Diagnosis, in layers.**

Configuration was correct:

```powershell
w32tm /query /configuration | Select-String -Pattern "NtpServer|Type"
# Type: NTP (Local)
# NtpServer: 10.10.10.1,0x8 (Local)
```

The peer was reachable and answering:

```powershell
w32tm /stripchart /computer:10.10.10.1 /samples:3 /dataonly
# 05:24:13, -00.0028244s
# 05:24:15, -00.0028619s
# 05:24:17, -00.0027962s
```

2.8 ms offset. So NTP worked at the network layer, the firewall was not blocking
UDP/123, and OPNsense was serving on LAN. The service simply had not accepted the
peer as its source.

**Fix.**

```powershell
Restart-Service w32time
Start-Sleep -Seconds 20
w32tm /resync
```

Plain `/resync`, not `/force` — the force flag can make the service discard a
peer it is still evaluating.

Result:

```
Stratum: 7 (secondary reference - syncd by (S)NTP)
ReferenceId: 0x0A0A0A01 (source IP: 10.10.10.1)
Source: 10.10.10.1,0x8
```

`0x0A0A0A01` is `10.10.10.1` in hex, byte by byte.

**If it recurs.** A freshly promoted PDC Emulator is supposed to be the domain's
authoritative clock, so `w32time` can mark itself reliable and treat external
peers as advisory. `AnnounceFlags` defaults to `10` — always announce as
reliable. Setting it to `5` announces reliability only once actually synchronised:

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\W32Time\Config" `
  -Name AnnounceFlags -Value 5
Restart-Service w32time
```

**Lesson.** Diagnose in layers and let each test isolate one thing.
`/query /configuration` proved the settings were right, `/stripchart` proved the
network path was right — which left only the service state. Without the
stripchart the obvious next move would have been firewall rules, and that would
have been an hour spent in the wrong place.

**Note on stratum.** Stratum 7 rather than the expected 3–4 means FW01 is
several hops from a reference clock. Stratum measures distance from the source,
not accuracy. Measured offset was 2.8 ms.

---

## 10 — WS01 received an APIPA address; Dnsmasq not bound to CLIENT

**Symptom.** WS01 booted with `169.254.117.127`, mask `255.255.0.0`, no default
gateway. `ipconfig /release` and `/renew` changed nothing.

**Diagnosis.** `169.254.x.x` is APIPA — Windows self-assigns it when a DHCP
request goes unanswered. The `/16` mask and empty gateway confirm it. So the
client was asking and nothing was replying.

Worked outward from the machine:

1. **VM adapter** — VM → Settings showed `Custom: VMnet3`, Connected, Connect at
   power on. Correct. Hypervisor ruled out.
2. **Service binding** — `Services → Dnsmasq DNS & DHCP → General`, the
   **Interfaces** field contained **LAN only**. CLIENT was absent.

**Fix.** Added CLIENT to the Interfaces field, saved, applied. `ipconfig /renew`
on WS01 immediately returned `10.10.20.139` with gateway `10.10.20.1` and DNS
`10.10.10.10`.

**Why it failed.** DHCP relies on broadcast. A client with no address cannot
unicast to a server it has not yet found, so it broadcasts to
`255.255.255.255`. Broadcasts do not cross a router — that is the definition of
a broadcast domain — so the DHCP server needs a listening socket on each segment
it serves. Dnsmasq bound to LAN only had a socket on `10.10.10.1`. The DISCOVER
arriving on `em2` was discarded because no process was listening there.

The console's option 2 wrote the DHCP *range*, but binding the service to an
interface is a separate setting in the web UI. Two halves of one configuration
living in two places.

**Lesson.** Diagnose outward from the client: adapter setting, then virtual
switch, then service running, then service configuration. The APIPA address
already located the fault at layer 2 or 3, which made examining DHCP *options*
pointless — there was no lease to carry them.

**Forward reference.** This is precisely the constraint DHCP relay exists to
solve. Phase 2 moves DHCP to Windows Server on CORE; the relay agent on FW01
catches the broadcast on `em2` and forwards it as unicast to `10.10.10.10`. That
is what `ip helper-address` does on Cisco equipment.

---

## 11 — Windows 11 Enterprise Evaluation ISO shipped already expired

**Symptom.** Immediately after a clean install, the desktop showed *"Windows
License is expired."*

```
License Status: Notification
Notification Reason: 0xC004F009 (grace time expired)
Remaining Windows rearm count: 0
Remaining SKU rearm count: 0
```

**Diagnosis.** Build `26100.1.240331-1435` — dated 31 March 2024, installed
September 2026. Evaluation media carries a **fixed expiry baked into the image**,
not a timer that starts at installation. Rearm count was already zero, so
`slmgr /rearm` reported success but changed nothing.

**Wrong first hypothesis.** The initial advice was to download a fresher ISO.
That was incorrect — `26100.1.240331-1435` *is* the current Windows 11 Enterprise
Evaluation image on Microsoft's Evaluation Center, unrefreshed since March 2024.
Redownloading returns the identical file.

**Fix.** Windows 11 **Enterprise LTSC 2024** is a separate product page with a
genuinely newer build, `26100.1742.240906-0331`, carrying its own unused
evaluation period. Full Enterprise edition, so domain join, Group Policy,
BitLocker, and Credential Guard all work. LTSC removes the Microsoft Store,
Edge, Copilot, and most inbox apps — a net benefit for a lab workstation, since
it means less log noise and fewer benchmark findings.

**Alternative considered.** Windows 11 Pro installed without a product key runs
indefinitely unactivated: watermark and locked personalisation, but no expiry.
Domain join works. Rejected because Credential Guard is Enterprise and Education
only, and it is needed for Phase 5.

**What the expiry does and does not block.** Licensing and directory services are
unrelated subsystems. The domain join, Kerberos, DNS registration, and Group
Policy all functioned normally on the expired install — Phase 1 was completed on
it deliberately, to prove the plumbing before rebuilding. What it *does* break is
Windows Update, which matters for Phase 4: OpenSCAP against an unpatchable host
produces findings that reflect missing updates rather than misconfiguration, and
no clean remediation delta can be demonstrated.

**Lesson.** Check the build date on evaluation media before spending an hour
installing it. Also: when correcting an assumption, verify the correction. The
first fix proposed here was wrong in a way that would have cost another hour.

**Partly superseded — see entry 12.** The claim above that `slmgr /rearm`
"reported success but changed nothing" was wrong. It was checked without a
restart, and a rearm does not take effect until the machine reboots. The clock
was later found running with a full 90-day window to 3 December 2026. The build
date finding and the LTSC recommendation still stand; the rearm conclusion does
not. Original text left in place — the error is the teaching material.

---

## 12 — Evaluation clock found running on a machine documented as expired

**Symptom.** Entry 11 recorded WS01's Windows 11 Enterprise Evaluation as
expired, with zero rearms and Windows Update blocked. Checking before deciding
whether to rebuild:

```
slmgr /xpr
# Time based activation will expire on 12/3/2026 2:46 AM
```

A live expiry roughly 90 days out — a full evaluation window on a machine
documented as dead. Windows Update, documented as blocked, was working.

**Diagnosis.** Two facts constrain the explanation.

A rearm does not extend a running clock. It resets the clock to the full
evaluation period and decrements the counter. A full 90-day window alongside a
spent counter is the signature of a rearm that has already taken effect, not of
a clock that never started.

Entry 11 records `slmgr /rearm` being run and concluding it "changed nothing."
That conclusion was drawn without a restart. **A rearm is staged, not immediate —
it applies at the next boot.** Checking the licence state between the command and
the reboot shows the old value and looks exactly like failure.

Best available explanation: the rearm ran, consumed the single allowance — which
is why the counter read zero — and applied at a later reboot, resetting the clock
to a full 90 days ending 3 December 2026.

**Not fully established.** The counter was read as zero *before* the reset was
observed, and the ordering of that read against the `/rearm` call was not
recorded at the time. `slmgr /dlv` on WS01, read for **License Status** and
**Remaining Windows rearm count**, would confirm or kill this. That check has not
been run. Recorded as a hypothesis rather than a finding — writing it up as
settled would put a guess into the repository wearing the clothes of a fact.

A smaller trap alongside it: `12/3/2026` is 3 December under month-first
formatting and 12 March under day-first, and `/xpr` gives no clue which. The
**Time remaining** field in `/dlv` is expressed in minutes and is
locale-independent. Prefer it whenever the date matters.

**Fix.** Nothing at the system level; the machine is licensed and patching. The
fault was documentary. `README.md` and `docs/licensing-clock.md` corrected to
record licensed to 3 December 2026, zero rearms remaining, updates functional.
Entry 11 annotated rather than rewritten.

The consequence is a changed deadline, not a changed plan. The LTSC rebuild
driver moves from "cannot patch" to "clock expires before Phase 4 scanning."
Both lead to the same rebuild; they imply very different urgency, and the wrong
one was on record.

**Lesson.** A staged change checked before it applies looks identical to a change
that failed. `slmgr /rearm`, `Rename-Computer` (entry 08), and interface
assignment (entry 02) now account for three entries with the same shape — a
command whose result cannot be trusted at the moment it returns. Re-check after
the reboot, not before it.

The second lesson points inward. Entries 02, 05, and 08 are about trusting a
remembered path over the filesystem. This one is about trusting a written record
over the running system. A document is an assertion about state, and an assertion
nobody re-tests drifts away from what is true. This was caught only because two
sources disagreed and a ten-second command settled it. Had the documentation been
the sole source, the lab would have carried a wrong deadline into Phase 4.
