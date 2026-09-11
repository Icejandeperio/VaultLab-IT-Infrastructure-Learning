# Runbook 05 — Phase 2: Automation

**Prerequisite:** Phase 1 complete and verified
**Result:** The forest rebuildable from code, DHCP served by Windows Server via a relay, and an internal certificate authority
**Cost:** Zero

**Status:** Not started. ADRs 008, 009, 010 written and accepted.

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

| Setting | Value |
|---|---|
| Guest OS | Linux → Ubuntu 64-bit |
| Name / location | `ANS01` / `C:\Lab\VMs\ANS01` |
| Firmware | UEFI |
| Processors | 1 socket × 2 cores |
| Memory | 2048 MB |
| Disk | 20 GB, split, not preallocated |
| Network | **Custom → VMnet2 (CORE)** |
| Address | `10.10.10.30` static |

Ubuntu Server **26.04.1 LTS**, minimal install, OpenSSH server enabled. Decline
Easy Install — choose *"I will install the operating system later"* and attach the
ISO in Customize Hardware. Easy Install writes an autoinstall answer file and
takes the partitioning and user-creation decisions away from you.

Remove the Sound Card and USB Controller.

Rationale for the OS version, the core count, and the CORE placement is in
ADR-008. Set the hostname to `ANS01` during the installer so the VM name and the
machine's own name agree.

### Static addressing

Ubuntu Server uses netplan. Verify the actual filename rather than assuming:

```bash
ls /etc/netplan/
sudo nano /etc/netplan/<the file you found>
```

Confirm the interface name first — `ens33` is typical on VMware but depends on
PCI slot and firmware:

```bash
ip -br link
```

```yaml
network:
  version: 2
  ethernets:
    <the interface you found>:
      dhcp4: no
      addresses: [10.10.10.30/24]
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses: [10.10.10.10]
        search: [corp.vaultlab.net]
```

```bash
sudo netplan apply
ip addr show
ping -c3 10.10.10.10
nslookup dc01.corp.vaultlab.net
```

Record the MAC and interface name in `docs/interface-mapping.md`.

### Install Ansible

Check what the distribution actually provides before pasting an install command:

```bash
apt-cache policy ansible
apt-cache policy python3-winrm
```

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ansible git
ansible --version
```

`pywinrm` is what lets Ansible speak to Windows. Without it, Windows hosts fail
with a connection plugin error that does not obviously name the missing library.
Prefer the distribution package if `apt-cache policy python3-winrm` shows one
available. If not, pip is the fallback — and on a distribution with an
externally-managed Python environment, that needs either a virtual environment or
an explicit override:

```bash
pip3 install pywinrm --break-system-packages
```

`--break-system-packages` overrides PEP 668, which exists to stop pip from
overwriting files the system package manager owns. Acceptable on a single-purpose
control node; not a habit to carry to a shared machine.

Confirm it imports:

```bash
python3 -c "import winrm; print(winrm.__file__)"
```

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

```bash
ansible-galaxy collection install microsoft.ad
ansible-playbook -i inventory/hosts.yml playbooks/ou-structure.yml --ask-vault-pass
```

Expect `ok=6 changed=0`. **A `changed` count above zero on existing infrastructure
means the playbook does not describe reality** — fix the playbook, not the domain.

Then write playbooks for users and groups, DNS zones and forwarders, and time
configuration.

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

Build per runbook 03 sections 1–5. **Same hostname gate applies**, and activate
within 10 days.

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

- [ ] ANS01 built, addressed, resolving `corp.vaultlab.net`
- [ ] ANS01 MAC and interface name recorded in `docs/interface-mapping.md`
- [ ] `ansible-inventory --host dc01` shows credentials resolved from vault
- [ ] `ansible domain_controllers -m win_ping` returns `pong`
- [ ] `git check-ignore` confirms `vault.yml` is excluded
- [ ] Playbooks run against existing DC01 with `changed=0`
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
