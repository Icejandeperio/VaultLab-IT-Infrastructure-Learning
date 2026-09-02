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
slmgr /rearm    # resets the 180-day timer; requires internet
```

Applications, settings, and configuration are unaffected. Run it near the end of
the period, not immediately.

## Windows 11 Enterprise — WS01 (planned)

90-day evaluation. The prebuilt developer VMs Microsoft previously published were
discontinued in October 2024; build from an Evaluation Center ISO instead.

## Why this drives Phase 2

There are roughly twelve months of forest life available, then a mandatory
rebuild. Anything configured by hand is lost at that point.

Automating the build in Ansible turns expiry from a lost weekend into a
20-minute `ansible-playbook` run — and converts a licensing limitation into an
annual disaster-recovery drill, which is closer to real operational practice than
most labs ever reach.

**Action:** begin Ansible work well before March 2027, not after.
