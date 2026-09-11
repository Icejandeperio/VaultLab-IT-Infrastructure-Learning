# Resource Budget

## Ceiling

| Resource | Total | Reserved for host | Available |
|---|---|---|---|
| RAM | 23.7 GB usable | ~6 GB Windows + VMware + browser | **~16 GB** (2 GB headroom retained) |
| CPU | 16 threads (4P + 8E) | — | ~16 concurrent vCPU, **~8 meaningful** |
| Storage | 537 GB free | 80 GB Windows breathing room | **~450 GB** |

E-cores are substantially weaker than P-cores for virtualized workloads and the
Windows scheduler will park VM threads on them. Plan around roughly 8 meaningful
cores, not 12.

**RAM and CPU are not the same kind of constraint.** RAM is a hard allocation —
memory given to a VM is unavailable to everything else while that VM runs, which
is why the 16 GB figure governs every proposal here. A vCPU is a scheduling
entitlement, time-sliced: an idle VM's cores are handed to whoever wants them.
Exceeding the meaningful-core count is therefore a performance question, not a
hard stop, and it matters most when several VMs are busy simultaneously.

## Per-VM allocation

| VM | vCPU | RAM | Disk (thin) | Notes |
|---|---|---|---|---|
| FW01 | 2 | 2 GB | 20 GB | 6 NICs; headroom for Suricata in Phase 4 |
| DC01 | 2 | 3 GB | 60 GB | Server Core |
| WS01 | 2 | 4 GB | 64 GB | Windows 11, vTPM |
| ANS01 | 2 | 2 GB | 20 GB | Ubuntu Server, Ansible control node — see ADR-008 |
| SRV01 | 2 | 3 GB | 60 GB | Server Core, enterprise CA — see ADR-010 |
| SIEM01 | 2 | 6 GB | 100 GB | Wazuh all-in-one |
| KALI01 | 2 | 3 GB | 40 GB | Prebuilt VMware image |

ANS01 is 2 vCPU / 2 GB rather than the 1 / 1 originally planned. Ansible is
I/O-bound and spends most of its time blocked on WinRM round-trips, but the
installer, `apt upgrade`, `ansible-galaxy` collection installs, and NTLM message
encryption across concurrent forks are all genuinely CPU-bound. The extra
gigabyte keeps `ansible-galaxy` and multi-host runs out of swap — and swapping
degrades every VM on the host, because they share one NVMe.

## Runtime profiles

VMs run in sets, never all at once.

| Profile | Members | RAM | vCPU |
|---|---|---|---|
| **A — Infrastructure** | FW01, DC01, WS01 | 9 GB | 6 |
| **A2 — Automation** | FW01, DC01, WS01, ANS01 | 11 GB | 8 |
| **A3 — Phase 2 full** | FW01, DC01, WS01, ANS01, SRV01 | 14 GB | 10 |
| **B — Blue team** | FW01, DC01, WS01, SIEM01 | 15 GB | 8 |
| **C — Red team** | FW01, DC01, WS01, KALI01 | 12 GB | 8 |
| **D — Networking** | FW01, Containerlab host (6 GB) | 8 GB | 4 |

Profile B remains the RAM ceiling at 15 GB. **A3 is the vCPU ceiling at 10, which
exceeds the ~8 meaningful cores.** That is accepted rather than resolved: A3 runs
during playbook development and rebuild drills, where FW01 and WS01 are idle and
the work is concentrated on ANS01 and one target at a time. If A3 feels sluggish,
drop WS01 — it is not needed for playbook work against DC01 and SRV01.

SRV01 and ANS01 both shut down before Profile B runs. Wazuh at 6 GB does not fit
alongside either.

## Storage forecast

| Item | Estimate |
|---|---|
| ISOs — OPNsense, Server 2025, Windows 11 LTSC, Ubuntu Server | 40 GB |
| Five VMs through Phase 2, thin, post-install and patched | 180–220 GB |
| SIEM01 and KALI01, added Phase 4–5 | 60–100 GB |
| Snapshot headroom | 100 GB |
| **Total at Phase 2** | **~360 GB of 450 GB** |
| **Total at Phase 5** | **~420–460 GB of 450 GB** |

**Phase 5 is at or past the ceiling.** That is a known decision point, not a
surprise to discover mid-build. Options when it arrives, cheapest first: delete
snapshots that have outlived their purpose, retire KALI01 between exercises rather
than keeping it standing, or move ISOs off the lab volume once the installs are
done. The 100 GB snapshot reserve is the thing not to raid — snapshot growth is
the most common cause of a lab filling its disk, and a full disk with a running
snapshot chain is how virtual machines get corrupted rather than merely stopped.

## Highest-value upgrade

The host has two DDR4-3200 SO-DIMM slots in a 16 + 8 configuration, with a
platform ceiling of 64 GB. Replacing the 8 GB module with a 16 GB one yields
32 GB in matched dual channel.

At 32 GB: Profile A3 plus Wazuh comes into range, and full GOAD and a Security
Onion Eval node both become possible. This is the cheapest capability jump
available and beats any other purchase for this project. Not required — the plan
above works at 24 GB.

It does nothing for the CPU or storage positions above.
