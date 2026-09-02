# Resource Budget

## Ceiling

| Resource | Total | Reserved for host | Available |
|---|---|---|---|
| RAM | 23.7 GB usable | ~6 GB Windows + VMware + browser | **~16 GB** (2 GB headroom retained) |
| CPU | 16 threads (4P + 8E) | — | ~16 concurrent vCPU |
| Storage | 537 GB free | 80 GB Windows breathing room | **~450 GB** |

E-cores are substantially weaker than P-cores for virtualized workloads and the
Windows scheduler will park VM threads on them. Plan around roughly 8 meaningful
cores, not 12.

## Per-VM allocation

| VM | vCPU | RAM | Disk (thin) | Notes |
|---|---|---|---|---|
| FW01 | 2 | 2 GB | 20 GB | 6 NICs; headroom for Suricata in Phase 4 |
| DC01 | 2 | 3 GB | 60 GB | Server Core |
| WS01 | 2 | 4 GB | 64 GB | Windows 11, vTPM |
| SIEM01 | 2 | 6 GB | 100 GB | Wazuh all-in-one |
| KALI01 | 2 | 3 GB | 40 GB | Prebuilt VMware image |

## Runtime profiles

VMs run in sets, never all at once.

| Profile | Members | RAM |
|---|---|---|
| **A — Infrastructure** | FW01, DC01, WS01 | 9 GB |
| **B — Blue team** | FW01, DC01, WS01, SIEM01 | 15 GB |
| **C — Red team** | FW01, DC01, WS01, KALI01 | 12 GB |
| **D — Networking** | FW01, Containerlab host (6 GB) | 8 GB |

Profile B is the ceiling. Adding a fifth standing VM requires either the RAM
upgrade below or removing something else.

## Storage forecast

| Item | Estimate |
|---|---|
| ISOs | 40 GB |
| Five VMs, thin, post-install and patched | 180–220 GB |
| Snapshot headroom | 100 GB |
| **Total** | **~360 GB of 450 GB** |

Snapshot growth is the most common cause of a lab filling its disk. Keep the
100 GB reserve genuinely reserved.

## Highest-value upgrade

The host has two DDR4-3200 SO-DIMM slots in a 16 + 8 configuration, with a
platform ceiling of 64 GB. Replacing the 8 GB module with a 16 GB one yields
32 GB in matched dual channel.

At 32 GB: full GOAD and a Security Onion Eval node both come into range. This is
the cheapest capability jump available and beats any other purchase for this
project. Not required — the plan above works at 24 GB.
