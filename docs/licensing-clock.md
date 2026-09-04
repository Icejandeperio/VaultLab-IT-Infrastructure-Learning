# Licensing Clock

Evaluation media is the only free route to Windows Server. The expiry is a
scheduled event to design around, not an obstacle.

## Windows Server 2025 — DC01

| | |
|---|---|
| Edition | Standard Evaluation (`ServerStandardEval`) |
| Channel | `TIMEBASED_EVAL` |
| Activated | 2 September 2026 |
| Evaluation period | 180 days |
| **Expires** | **~1 March 2027** |
| **Remaining rearm count** | **1** |
| Ceiling with rearm | **~late August 2027** |

Verify at any time with `slmgr /dlv`. Read **License Status** and **Remaining
Windows rearm count**.

Server 2025 permits **one** rearm, not the six that older guidance describes for
Server 2016/2019/2022. That older figure does not carry over — this was confirmed
directly from `slmgr /dlv` on this installation rather than taken from
documentation.

## Two separate clocks

| Clock | Duration | Consequence if missed |
|---|---|---|
| Activation grace | 10 days from install | Automatic shutdown until activated |
| Evaluation period | 180 days from activation | Hourly shutdowns |

The 10-day clock catches people because nothing warns you. `slmgr /ato` must
succeed within it. Activation requires working DNS — see `troubleshooting-log.md`,
entry 06.

## What expiry actually does

Nothing is deleted. The AD database, files, and configuration remain intact. The
server begins shutting down roughly every hour with a licence expiry warning. It
becomes unusable as a service, not destroyed.

## Extending

```powershell
slmgr /dlv      # confirm rearm count before relying on it
slmgr /rearm    # resets the timer; requires internet
Restart-Computer
```

**A rearm does not take effect until the machine restarts.** Checking the clock
between the command and the reboot shows the old value and looks like failure.
That assumption produced a wrong entry in this repository — see
`troubleshooting-log.md`, entries 11 and 12.

A rearm also does not *extend* a running clock. It resets it to the full
evaluation period and decrements the counter. A host showing a full evaluation
window next to a spent counter has already rearmed.

Applications, settings, and configuration are unaffected. Run it near the end of
the period, not immediately.

## Windows 11 Enterprise — WS01

| | |
|---|---|
| Edition | Enterprise Evaluation, build `26100.1.240331-1435` |
| Evaluation period | 90 days |
| **Expires** | **3 December 2026** |
| **Remaining rearm count** | **0** |
| Ceiling | **3 December 2026 — no extension path** |
| Windows Update | Working |

The installed image shipped with a fixed expiry already passed (entry 11). A
rearm was subsequently applied and reset the clock to a full 90-day window
running to 3 December 2026. The counter is now spent. **There is no second
rearm.** The exact sequence is recorded as a hypothesis in entry 12, not a
finding. What is established: the machine is licensed, it patches, and
3 December 2026 is final.

The prebuilt developer VMs Microsoft previously published were discontinued in
October 2024; build from an Evaluation Center ISO instead.

### Replacement: Enterprise LTSC 2024

Windows 11 **Enterprise LTSC 2024** (`26100.1742.240906-0331`) is a separate
product page from the standard Enterprise evaluation, with a genuinely newer
build and its own unused evaluation period.

LTSC is the Long-Term Servicing Channel — the same kernel under a different
servicing contract. Security updates only, no annual feature updates, build fixed
for the life of the release. It ships without the Store, Edge, and most inbox
apps, which is a net benefit here: less RAM, fewer background services, and
materially less log noise once Wazuh agents ship events in Phase 4.

Enterprise SKU is why LTSC rather than unactivated Windows 11 Pro, which would
escape the clock entirely. AppLocker, Credential Guard, and Windows Defender
Application Control are Enterprise and Education only, and Phase 4 tier-zero GPO
work and Phase 5 credential-theft simulation both depend on them.

**Known cost.** CIS and STIG content assumes the General Availability channel.
Rules referencing the Store, Edge, or inbox UWP apps will evaluate as
non-applicable on LTSC, leaving gaps in the Phase 4 remediation delta. Those gaps
must be explained in the evidence rather than quietly omitted.

**Verify on the new install:** the published build number, the License Status,
and the rearm count. Do not assume LTSC ships with more than one rearm.

**Deadline:** rebuild before Phase 4 begins, and in any case before 3 December
2026.

## Windows Server 2025 — SRV01 (planned)

Phase 2 adds SRV01 as a certificate authority. That is a third evaluation clock:
180 days and one rearm from its own install date, with the 10-day activation
grace applying separately. Record it here from `slmgr /dlv` at build time rather
than copying DC01's dates.

## Reading either clock

```
slmgr /xpr     # expiry date only, brief
slmgr /dlv     # full detail
```

`/dlv` opens a dialog rather than printing to the console. Four fields matter:

| Field | Tells you |
|---|---|
| Name / Description | The actual SKU and channel — Evaluation and LTSC are different strings |
| License Status | `Licensed`, `Notification` (expired), or `Initial grace period` |
| Time remaining | In minutes — resolves date-format ambiguity that `/xpr` does not |
| Remaining Windows rearm count | Whether any lever remains |

`/xpr` returns one of three phrasings and they are not interchangeable. **"Time
based activation will expire"** is an evaluation or timed edition: valid now, on
a countdown, no renewal. **"Volume activation will expire"** is KMS, which renews
itself against a KMS host. **"The machine is permanently activated"** has no
clock. Every host in this lab is in the first category.

Read `/dlv` after any activation work, not only when something appears broken.

## Why this drives Phase 2

There are roughly twelve months of forest life available, then a mandatory
rebuild. Anything configured by hand is lost at that point.

Automating the build in Ansible turns expiry from a lost weekend into a
20-minute `ansible-playbook` run — and converts a licensing limitation into an
annual disaster-recovery drill, which is closer to real operational practice than
most labs ever reach.

**Action:** begin Ansible work well before March 2027, not after.
