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
