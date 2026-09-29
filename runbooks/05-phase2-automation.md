# Runbook 05 — Phase 2: Automation

**Prerequisite:** Phase 1 complete and verified
**Result:** The forest rebuildable from code, DHCP served by Windows Server via a relay, and an internal certificate authority
**Cost:** Zero

**Status:** In progress. §2 complete — ANS01 built and verified, snapshot
`01-ans01-base`. §3, WinRM on DC01, next. ADRs 008, 009, 010 written and accepted.

---

## Why this phase, and why now

DC01's evaluation expires around **1 March 2027**, with **one** rearm available,
giving a hard ceiling of roughly **late August 2027**. See
`docs/licensing-clock.md`.

At that point the forest is gone. Everything configured by hand is lost.

Automating the build turns expiry from a lost weekend into a 20-minute
`ansible-playbook` run, and converts a licensing limitation into an annual
disaster-recovery drill. Rebuilding your own environment from code once a year is
closer to real operational practice than most labs ever reach.

This is also the phase that changes what the project demonstrates. "I built a
domain controller" is common. "I can rebuild the entire forest from
version-controlled code, with a measured recovery time" is not.

## What Ansible actually is

Three properties define it, and each explains a decision in this runbook.

**Agentless.** Most configuration management tools install a daemon on every
managed machine that wakes periodically and pulls its configuration. Ansible does
not. It opens a connection — SSH for Linux, WinRM for Windows — pushes a small
Python or PowerShell module across, runs it, collects the result, deletes it.
Nothing persists on the target.

The consequence: nothing to install on DC01 or WS01 beyond a management protocol
they already speak. But the control node needs credentials for every target,
which is why ANS01 sits on CORE and why the vault matters.

**Push, not pull.** Nothing happens until a playbook runs. There is no drift
correction between runs — a machine changed by hand at 3pm stays changed until
the next execution.

**Idempotent.** Modules describe *desired state*, not actions.
`microsoft.ad.ou: state=present` does not mean "create this OU," it means "this OU
should exist." Against a domain where it already exists, the module checks, finds
it, changes nothing, reports `ok`. Where it does not, it creates it and reports
`changed`.

That is why `changed=0` against live DC01 is the correctness test rather than "it
ran without errors." A playbook reporting zero changes against real infrastructure
provably describes reality — and if it describes reality, replaying it onto a
blank server reproduces that reality. That inference is the entire basis for
treating March 2027 as a drill rather than a loss. A shell script offers no such
guarantee; `New-ADOrganizationalUnit` run twice throws an error the second time.

---

## 1. Resource plan

Phase 2 adds two VMs. See `docs/resource-budget.md` for the full picture.

| VM | vCPU | RAM |
|---|---|---|
| FW01 | 2 | 2 GB |
| DC01 | 2 | 3 GB |
| WS01 | 2 | 4 GB |
| ANS01 | 2 | 2 GB |
| SRV01 | 2 | 3 GB |
| **Total (Profile A3)** | **10** | **14 GB** |

RAM is inside the ~16 GB ceiling. vCPU exceeds the ~8 meaningful cores, which is
accepted — these VMs are not busy simultaneously. If A3 feels sluggish during
playbook work, shut down WS01; it is not needed for playbooks targeting DC01 and
SRV01.

Wazuh at 6 GB does not fit alongside. SRV01 and ANS01 both shut down before
Profile B runs.

---

## 2. Build ANS01

**Complete.** This section records the build as it was actually done, including
the places where the original plan was wrong. Rationale for the OS version, core
count, and CORE placement is in ADR-008.

**Before starting:** FW01 and DC01 must be running. CORE has no DHCP server and
ANS01 resolves names through DC01. With either one off, the installer's mirror
test fails in a way that looks like an installer problem.

### 2.1 Create the VM

New Virtual Machine → **Custom**. Decline Easy Install — *"I will install the
operating system later"* — so partitioning and user creation stay under your
control.

| Setting | Value | Why |
|---|---|---|
| Guest OS | Linux → **Ubuntu 64-bit** | Plain **Ubuntu** is the 32-bit profile. The ISO is `amd64`; match the profile to the ISO's architecture |
| Name / location | `ANS01` / `C:\Lab\VMs\ANS01` | |
| SCSI controller | **LSI Logic** (default) | The Linux kernel carries the driver. Contrast runbook 03, where Server 2025 needs LSI Logic SAS. The right controller is a property of the guest OS |
| Disk | Create a new virtual disk, 20 GB, split, **not** preallocated | Never a physical disk — it disables snapshots and exposes a real drive to the installer |
| Processors | **1 processor × 2 cores** | The wizard defaults to 1 core. It came back as 1 once during this build — verify the Settings summary reads **2** |
| Memory | 2048 MB | |
| Network | **Custom → VMnet2** | See below |

**Network.** The wizard may offer only Bridged, NAT, and Host-only. None of those
is VMnet2 — they map to VMnet0, VMnet8, and VMnet1. Pick any as a placeholder and
set **Custom: Specific virtual network → VMnet2** in Customize Hardware. The label
`VMnet2 (Host-only)` describes the switch *type* configured in runbook 01, not the
Host-only radio button.

**Remove** the Sound Card and USB Controller.

### 2.2 Firmware — set it explicitly

**VM → Settings → Options → Advanced:**

- **Firmware type: UEFI.** The wizard never asks, and for the Ubuntu 64-bit
  profile it defaulted to **BIOS**. This must be right before the storage step —
  an installed BIOS system cannot be switched to UEFI without a reinstall. See
  troubleshooting entry 14.
- **Enable secure boot** — tick it if available. It was greyed out on this build.
  Cause unconfirmed; `shim-signed` is installed, so it can generally be enabled
  later.
- **Disable side channel mitigations** — ticked, matching FW01 and DC01.

Reopen Settings after saving and confirm the firmware change stuck.

### 2.3 Installer

1. GRUB → **Try or Install Ubuntu Server**
2. Language English, keyboard default. Skip any installer update offer.
3. Install type → **Ubuntu Server**, not *minimized* — minimized strips man pages
   and interactive tools. Third-party drivers **off**: they are for physical
   hardware, and the kernel supports VMware's virtual devices natively.
4. **Network — read the screen before pressing Enter.** DHCP fails, which is
   expected: CORE has no DHCP server. The cursor defaults to **Continue without
   network**. Arrow up to the interface instead → **Edit IPv4 → Manual**:

   | Field | Value |
   |---|---|
   | Subnet | `10.10.10.0/24` |
   | Address | `10.10.10.30` |
   | Gateway | `10.10.10.1` |
   | Name servers | `10.10.10.10` |
   | Search domains | `corp.vaultlab.net` |

   **Record the interface name shown.** On this build it was **`ens32`**, not the
   `ens33` earlier drafts assumed — see `docs/interface-mapping.md`.

   **Check:** the interface shows `static 10.10.10.30/24` and the button reads
   **Done**, not *Continue without network*.
5. Proxy → blank. ANS01 reaches the internet through its gateway; a proxy is a
   different mechanism, and entering one would break every download.
6. Mirror → wait for **"This mirror location passed tests."** That single test
   exercises the static address, the gateway, DNS through DC01, and FW01's NAT.
7. Guided storage → **Use an entire disk**, **LVM on**, **LUKS off**.

   LUKS would require a passphrase at the console on every boot, breaking
   unattended startup and the rebuild drill. The accepted cost: anything written
   to ANS01's disk in plaintext is readable from the VMDK. So **never store the
   vault password in a file on ANS01** — use `--ask-vault-pass`.
8. **Storage summary — two checks before Done:**
   - Partition 1 must be **primary ESP, FAT32, mounted at `/boot/efi`**. If it
     reads *BIOS grub spacer*, the firmware is BIOS — power off and fix 2.2. Safe
     at this point: nothing has been written yet.
   - **`ubuntu-lv` is left at roughly half the volume group by default.** Select it
     → **Edit** → set Size to the maximum shown. Free space should drop to zero.

   Then **Done → Continue** on the destructive-action dialog. This is the point
   of no return.
9. Profile:
   - Server name **`ans01`** — lowercase is the Linux convention; DNS is
     case-insensitive
   - Username **`vlabadmin`** — a **local** account, deliberately not named like
     the AD accounts; ANS01 is not domain-joined
   - Password — a password manager, never the repo. Different from the AD
     passwords: ANS01 will hold the vault containing `jcruz-adm`'s credentials
10. Ubuntu Pro → **Skip**. See ADR-008 for why it is worth revisiting in Phase 4.
11. SSH → **Install OpenSSH server**, password authentication **on** — temporarily.
    There are no keys yet, so a password is the only way in for the first
    connection. Disabling it once key-based login works is an open item.
12. Featured snaps → **none**. Snaps update themselves automatically in the
    background; on a machine holding domain credentials, software should change
    only when you run the update. ANS01 stays on apt.
13. **Reboot Now.** At *"Please remove the installation medium"*: **VM → Settings
    → CD/DVD** → untick **Connected** and **Connect at power on** → OK → Enter.

On first boot, setup messages print *after* the `login:` prompt and look like the
prompt vanished. Press Enter for a fresh one.

### 2.4 First connection — verify the host key

From PowerShell on the Windows host:

```powershell
ssh vlabadmin@10.10.10.30
```

The first connection shows the server's `ED25519 key fingerprint is SHA256:...`.
**Compare it character-for-character against the `SHA256:` line printed on the
ANS01 console at first boot** before typing `yes`.

This is trust on first use: SSH authenticates the server before you authenticate
yourself, and the first connection is the only one with no stored reference. The
console is a channel no one can sit in the middle of. After `yes`, the key is
stored in `known_hosts` and every later connection is checked against it. A later
warning that the host key **has changed** means either ANS01 was rebuilt or
something is impersonating it — investigate before dismissing.

Work over SSH from here on. The VMware console has no copy-paste.

**Verify the build against spec:**

```bash
hostnamectl                                          # ans01
nproc                                                # 2
[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS   # UEFI
df -h /                                              # ~17 GB
ip -br addr                                          # ens32 UP 10.10.10.30/24
systemctl is-active ssh                              # active
```

The `/sys/firmware/efi` directory only exists when the kernel was booted by UEFI,
which makes it the definitive check regardless of what the VM settings claim.

### 2.5 Baseline

**Updates.**

```bash
sudo apt update && sudo apt full-upgrade -y
apt list --upgradable
```

`full-upgrade` rather than `upgrade`, because plain `upgrade` never removes a
package and can hold back updates that need dependencies swapped. The second
command should list nothing — proof the first run completed.

The output includes harmless noise worth recognising: `File descriptor ... leaked
on vgs invocation` is an LVM warning triggered by the bootloader updater, and
`os-prober will not be executed` means GRUB no longer scans for other operating
systems. `Service restarts being deferred: dbus.service` is not noise — dbus
cannot be safely restarted on a live system, so reboot before the snapshot.

**Timezone.**

```bash
sudo timedatectl set-timezone Asia/Manila
timedatectl
```

The output shows `Asia/Manila (PST, +0800)`. **`PST` here is Philippine Standard
Time**, not Pacific — timezone abbreviations are ambiguous, which is why logs
should record UTC or an explicit offset.

**Time sync — Ubuntu 26.04 runs chrony, not timesyncd.** Earlier drafts of this
runbook said to edit `/etc/systemd/timesyncd.conf`. Check which service is actually
running before configuring anything:

```bash
systemctl is-active chrony systemd-timesyncd
```

On this build: `active`, `inactive`. Confirm chrony reads a drop-in directory,
then add DC01:

```bash
grep sourcedir /etc/chrony/chrony.conf
echo "server 10.10.10.10 iburst prefer" | sudo tee /etc/chrony/sources.d/dc01.sources
sudo chronyc reload sources
sleep 20
chronyc sources -v
```

A drop-in file rather than editing `chrony.conf` directly: package updates may
replace the main config, while a separate file is yours and survives. Canonical's
NTS sources are kept alongside DC01 as an authenticated cross-check — see ADR-008
for the trade-off.

**Expected result as built:** DC01 appears marked **`?`** — rejected as unusable,
because its Windows time service advertises about 4 seconds of uncertainty and
chrony refuses anything above 3. ANS01 syncs to the NTS sources meanwhile. This is
correct behaviour, not a fault to work around; the fix is on DC01, in
`time-config.yml`. See troubleshooting entry 15. **Do not raise chrony's limit.**

The `dc01.sources` line is knowingly inert until then, and it makes
`chronyc sources` a live measurement of DC01's clock against authenticated time.

**VMware tools.**

```bash
systemctl is-active open-vm-tools
```

Should be `active` — the installer detects VMware and installs it.

### 2.6 Ansible

Check what the distribution provides before installing:

```bash
apt-cache policy ansible python3-winrm
```

```bash
sudo apt install -y ansible python3-winrm git
ansible --version
python3 -c "import winrm; print(winrm.__file__)"
ansible-galaxy collection list microsoft.ad
```

**As built:**

| Item | Result |
|---|---|
| `ansible` package | 13.1.0 — the bundle of ansible-core plus curated collections |
| ansible-core | 2.20.1 — the engine; `ansible --version` reports this, not 13.x |
| Python | 3.14.4 |
| `pywinrm` | From apt, under `/usr/lib/python3/dist-packages/winrm/` |
| `microsoft.ad` | 1.10.0, bundled — no separate install |

`pywinrm` is what lets Ansible speak to Windows. It came from apt, so no pip
install and no `--break-system-packages` override were needed — apt tracks and
updates it with everything else. If a future rebuild finds no apt candidate, pip
is the fallback, and on a system with an externally-managed Python environment it
needs either a virtual environment or that override. Acceptable on a
single-purpose control node; not a habit for a shared machine.

Both `ansible` and `python3-winrm` come from Ubuntu's **universe** component —
see ADR-008.

### 2.7 Close out

```bash
sudo reboot
```

Reconnect and confirm the state survived a reboot before capturing it — this is
the first boot since the upgrade touched `netplan.io` and deferred dbus:

```bash
ip -br addr
systemctl is-active ssh chrony
chronyc sources
```

Then `sudo poweroff`, and **VM → Snapshot → Take Snapshot → `01-ans01-base`**.
Confirm it appears in Snapshot Manager rather than assuming the dialog succeeded.

### 2.8 Observed, not configured

After the upgrade, the console login began printing `Try contacting this VM's SSH
server via 'ssh vsock%<id>' from host.` — systemd exposing SSH over vsock, a
host-to-guest channel with no network path. Recorded in
`docs/interface-mapping.md`.

### 2.9 Open items from this build

- Disable SSH password authentication once key-based login is set up
- Secure Boot, if VMware ever offers it for this VM
- DC01 accepted by chrony — depends on `time-config.yml`

---

## 3. Enable WinRM on Windows targets

On **DC01**, from PowerShell:

```powershell
Enable-PSRemoting -Force
winrm quickconfig
Get-Service WinRM | Select-Object Status, StartType
```

**Then verify what listener was actually created:**

```powershell
winrm enumerate winrm/config/listener
```

Read the output rather than assuming. A default `quickconfig` creates an HTTP
listener on 5985. That is unencrypted, and while Ansible's WinRM transport can
negotiate message-level encryption over NTLM or Kerberos, an HTTP listener on a
domain controller is not something to leave in place.

### HTTPS listener

Once SRV01 is a CA (section 7), issue a server authentication certificate to DC01
and bind a listener on 5986. Until then, for lab bootstrapping only, a self-signed
certificate works:

```powershell
$cert = New-SelfSignedCertificate -DnsName "dc01.corp.vaultlab.net" -CertStoreLocation Cert:\LocalMachine\My
New-Item -Path WSMan:\localhost\Listener -Transport HTTPS -Address * -CertificateThumbPrint $cert.Thumbprint -Force
```

> **Trade-off, stated explicitly.** A self-signed WinRM certificate provides
> encryption without authentication of the endpoint — it protects against passive
> interception but not against an active man-in-the-middle. Acceptable for a
> bootstrap step on an isolated segment. Replace it with a CA-issued certificate
> as soon as SRV01 exists, and record the replacement in ADR-010.

ANS01 → DC01 is CORE → CORE and never reaches the firewall. ANS01 → WS01 crosses
CORE to CLIENT and will need an explicit rule once segmentation is tightened,
scoped to ANS01's address rather than to `CORE net`. See `docs/firewall-policy.md`.

---

## 4. Inventory and vault

On ANS01:

```bash
mkdir -p ~/vaultlab-ansible/{inventory,playbooks,roles}
mkdir -p ~/vaultlab-ansible/group_vars/all
cd ~/vaultlab-ansible
```

### The group_vars layout

Ansible resolves variables for a group by looking for `group_vars/<groupname>` and
accepting **either a file or a directory**, not both. An earlier draft of this
runbook instructed creating `group_vars/all.yml` *and* `group_vars/all/vault.yml`
simultaneously. Those forms collide, and which wins is not something to guess at.

The failure mode is nasty: a variable that silently fails to load surfaces as an
authentication rejection, sending you to the credentials instead of the layout.

**Directory form only:**

```
group_vars/
└── all/
    ├── main.yml      ← non-secret
    └── vault.yml     ← encrypted, gitignored
```

The repo's `.gitignore` glob is `group_vars/*/vault.yml`, which matches the
directory form and would never have matched a flat `all.yml`.

### Files

`inventory/hosts.yml`:

```yaml
all:
  children:
    domain_controllers:
      hosts:
        dc01:
          ansible_host: 10.10.10.10
    member_servers:
      hosts:
        srv01:
          ansible_host: 10.10.10.11
    workstations:
      hosts:
        ws01:
          ansible_host: 10.10.20.139
```

WS01's address is a DHCP lease, not a reservation. If a playbook cannot reach it
later, confirm the lease before diagnosing WinRM.

`group_vars/all/main.yml` — **non-secret values only**:

```yaml
ansible_connection: winrm
ansible_winrm_transport: ntlm
ansible_port: 5985
ansible_winrm_server_cert_validation: ignore
domain_name: corp.vaultlab.net
domain_netbios: VAULTLAB

ansible_user: "{{ vault_ansible_user }}"
ansible_password: "{{ vault_ansible_password }}"
```

### Credentials in Ansible Vault

```bash
ansible-vault create group_vars/all/vault.yml
```

```yaml
vault_ansible_user: jcruz-adm@corp.vaultlab.net
vault_ansible_password: <the password>
```

Verify the ignore rule **before** the first commit:

```bash
git check-ignore -v group_vars/all/vault.yml
```

If that returns nothing, the file is *not* ignored. Fix it before committing. A
leaked secret stays leaked.

### Verify the variables loaded

Before blaming credentials for anything:

```bash
ansible-inventory -i inventory/hosts.yml --host dc01 --ask-vault-pass
```

That prints the resolved variable set for one host. If `ansible_password` is
absent, the vault file was not loaded and the problem is layout, not
authentication. Ten seconds here saves an hour in the wrong layer.

### Test

```bash
ansible domain_controllers -i inventory/hosts.yml -m win_ping --ask-vault-pass
```

`pong` means WinRM, credentials, DNS, and the firewall path all work.

**If it fails**, the error names the layer:

| Error | Layer |
|---|---|
| `ntlm: the specified credentials were rejected` | Credentials, UPN format, **or variables not loaded** |
| `Connection refused` / timeout | WinRM listener or firewall |
| `plugin requires pywinrm` | Missing Python library on ANS01 |
| `certificate verify failed` | Cert validation setting versus listener type |

---

## 5. First playbooks — reproduce Phase 1

Write these to match what already exists. The test of correctness is that running
them against DC01 reports **zero changes**.

`playbooks/ou-structure.yml`:

```yaml
---
- name: Build VAULTLAB OU structure
  hosts: domain_controllers
  gather_facts: no
  tasks:
    - name: Create top-level OU
      microsoft.ad.ou:
        name: VAULTLAB
        path: "DC=corp,DC=vaultlab,DC=net"
        state: present

    - name: Create child OUs
      microsoft.ad.ou:
        name: "{{ item }}"
        path: "OU=VAULTLAB,DC=corp,DC=vaultlab,DC=net"
        state: present
      loop:
        - Users
        - Workstations
        - Servers
        - ServiceAccounts
        - Groups
```

**Do not run `ansible-galaxy collection install microsoft.ad`.** The collection
is already bundled with the `ansible` package (1.10.0 as built). Ansible searches
`~/.ansible/collections` **before** the bundled location, so a separate install
would place a second copy there and silently shadow the bundled one. Two copies at
possibly different versions, with the one that runs decided by search order, is
behaviour that will not reproduce on a rebuild. Install a separate copy only when a
newer version is deliberately needed — and record it when you do.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/ou-structure.yml --ask-vault-pass
```

Expect `ok=6 changed=0`. **A `changed` count above zero on existing infrastructure
means the playbook does not describe reality** — fix the playbook, not the domain.

Then write playbooks for users and groups, DNS zones and forwarders, and time
configuration.

### `time-config.yml` — scope expanded by the ANS01 build

This playbook does more than reproduce Phase 1. Troubleshooting entry 15 found two
problems on DC01 that were deliberately left for it:

- **Time accuracy.** Configure the Windows time service for high accuracy on
  DC01, following Microsoft's *Configuring systems for high accuracy*
  documentation. Take the registry values from that page at build time, not from
  memory.
- **Timezone.** Set UTC+08:00 with no daylight saving on **every** Windows host.
  DC01 is on US Pacific; WS01 is unverified. Use the same zone `Id` everywhere and
  record which one.

Unlike the other Phase 2 playbooks, this one is **expected to report `changed`**
on its first run — it corrects state rather than describing it. The second run
must report `changed=0`. That pair of runs is the idempotence lesson in its
cleanest form.

**Success check, from ANS01:** `chronyc sources` shows DC01 marked `*` or `+`
instead of `?`.

---

## 6. The rebuild drill

This is the point of the phase. Everything before it is preparation.

1. Take a fresh snapshot of DC01, labelled clearly as pre-drill
2. Build a **new** VM from the Server 2025 ISO following runbook 03 sections 1–5,
   including the hostname gate
3. Promote it with `Install-ADDSForest`
4. Run the full playbook set against it
5. Verify against runbook 03's checklist — `dcdiag`, DNS, time, OU structure
6. Domain-join WS01 to the rebuilt forest

**Record how long steps 4 to 6 take.** That number is your recovery time
objective, and being able to state it with evidence is a genuine differentiator.

Anything fixed by hand during step 5 is a gap in the playbooks. Backport it and
run the drill again.

---

## 7. SRV01 — certificate authority

| Setting | Value |
|---|---|
| Name | `SRV01` |
| Address | `10.10.10.11` static, CORE |
| OS | Windows Server 2025 Standard Eval, Server Core |
| Processors | 1 socket × 2 cores |
| Memory | 3072 MB |
| Disk | 60 GB |

Build per runbook 03 sections 1–5. **Same hostname gate applies**, the timezone
step applies, and activate within 10 days.

**Record SRV01's licensing clock in `docs/licensing-clock.md` at build time.**
Run `slmgr /dlv` after activation and write down License Status, Time remaining,
and the rearm count. This is the lab's third clock — do not carry DC01's figures
across by assumption.

Join the domain before installing ADCS:

```powershell
Add-Computer -DomainName "corp.vaultlab.net" `
  -OUPath "OU=Servers,OU=VAULTLAB,DC=corp,DC=vaultlab,DC=net" `
  -Credential (Get-Credential) -Restart
```

Then:

```powershell
Install-WindowsFeature ADCS-Cert-Authority -IncludeManagementTools

Install-AdcsCertificationAuthority `
  -CAType EnterpriseRootCA `
  -CACommonName "VAULTLAB-Root-CA" `
  -KeyLength 4096 `
  -HashAlgorithmName SHA256 `
  -ValidityPeriod Years `
  -ValidityPeriodUnits 10 `
  -Force
```

Verify:

```powershell
certutil -ping
certutil -CAInfo
```

On a domain member, confirm the root arrived automatically:

```powershell
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Subject -like "*VAULTLAB*"
```

If manual installation is required, the AD integration is not working — see
ADR-010.

---

## 8. DHCP migration and relay

### Install and authorise on DC01

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName dc01.corp.vaultlab.net -IPAddress 10.10.10.10
```

`Add-DhcpServerInDC` is the authorisation step and a genuine AD security control —
an unauthorised Windows DHCP server refuses to issue leases in a domain. It exists
to prevent rogue DHCP, a classic man-in-the-middle vector, since whoever answers
DHCP sets the client's gateway and DNS server.

### Create the scope

```powershell
Add-DhcpServerv4Scope -Name "CLIENT" `
  -StartRange 10.10.20.100 -EndRange 10.10.20.200 `
  -SubnetMask 255.255.255.0 -State Active

Set-DhcpServerv4OptionValue -ScopeId 10.10.20.0 `
  -Router 10.10.20.1 `
  -DnsServer 10.10.10.10 `
  -DnsDomain corp.vaultlab.net
```

The scope is for a subnet the server has no interface on. That is exactly what the
relay makes possible.

### Disable Dnsmasq DHCP on FW01

**Services → Dnsmasq DNS & DHCP → General** — remove CLIENT from the
**Interfaces** field, or disable the DHCP range.

Two DHCP servers on one segment produce intermittent failures that are unpleasant
to diagnose. Turn the old one off before turning the new one on.

### Configure the relay

**Services → DHCPRelay** in the OPNsense UI:

- Enable
- Interface: **CLIENT**
- Destination server: `10.10.10.10`

### What the relay actually does

WS01 broadcasts a DHCPDISCOVER to `255.255.255.255`. FW01 receives it on `em2`,
rewrites it as a unicast packet to `10.10.10.10`, and inserts its own CLIENT
interface address into the `giaddr` field.

That `giaddr` is how the server knows which scope to serve from — the unicast
source is the relay, not the client, so a server holding four scopes would
otherwise have no way to tell where the request originated. This is the entire
mechanism behind `ip helper-address`.

### Verify

On WS01:

```powershell
ipconfig /release
ipconfig /renew
ipconfig /all
```

`DHCP Server` should now read `10.10.10.10`, not `10.10.20.1`.

On DC01:

```powershell
Get-DhcpServerv4Lease -ScopeId 10.10.20.0
```

Update `docs/address-plan.md` if WS01's address changed.

---

## 9. Acceptance criteria

- [x] ANS01 built on UEFI, addressed, resolving `corp.vaultlab.net`
- [x] ANS01 host key fingerprint verified against the console on first connection
- [x] ANS01 MAC and interface name recorded in `docs/interface-mapping.md`
- [x] Snapshot `01-ans01-base` exists
- [ ] `ansible-inventory --host dc01` shows credentials resolved from vault
- [ ] `ansible domain_controllers -m win_ping` returns `pong`
- [ ] `git check-ignore` confirms `vault.yml` is excluded
- [ ] Playbooks run against existing DC01 with `changed=0`
- [ ] `time-config.yml` run twice: `changed` then `changed=0`; DC01 no longer `?`
      in ANS01's `chronyc sources`
- [ ] A DC rebuilt from ISO plus playbooks passes runbook 03's full checklist
- [ ] Rebuild time recorded as a stated RTO
- [ ] SRV01 domain-joined, ADCS installed, `certutil -ping` succeeds
- [ ] SRV01's licensing clock recorded in `docs/licensing-clock.md`
- [ ] Root certificate present in the Trusted Root store on a domain member
      without manual installation
- [ ] DHCP authorised in AD, scope active, Dnsmasq DHCP disabled
- [ ] Relay configured; WS01's lease shows `10.10.10.10` as DHCP server
- [ ] WinRM HTTPS listener using a CA-issued certificate, self-signed retired
- [ ] Every fault encountered has a troubleshooting-log entry
- [ ] Address plan, topology, resource budget, and README status table all match
      what is actually running

---

## 10. Repository additions

```
ansible/
├── inventory/hosts.yml
├── group_vars/
│   └── all/
│       ├── main.yml
│       └── vault.yml         ← gitignored
├── playbooks/
│   ├── ou-structure.yml
│   ├── users-groups.yml
│   ├── dns-config.yml
│   ├── time-config.yml
│   └── site.yml
└── README.md                 ← how to run, what each playbook does
```

`ansible/README.md` should state the vault password location (not the password),
the command to run a full rebuild, and the measured RTO.

---

## 11. What Phase 2 changes about the project

Phase 1 demonstrated that you can build infrastructure. Phase 2 demonstrates that
you can rebuild it — deterministically, from version control, with a stated
recovery time.

That is the difference between a lab and an operational capability, and it is the
distinction that matters when the roles targeted are infrastructure operations
rather than desktop support.

The commit history through this phase is itself evidence. A playbook that starts
imperfect and converges toward `changed=0` over several commits shows the
iteration honestly, which is more persuasive than a repository that appears to
have been correct on the first attempt.
