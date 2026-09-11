# Change Control

Decisions in this project get made faster than documents get written. That gap is
where every documentation fault so far has come from. This file closes it.

## The rule

**A change to a decision, an address, a resource allocation, or a procedure is not
finished until the file that records it has moved.** Not "noted for later." The
same working session.

If that feels heavy for a small change, the small changes are the ones that rot.
ADR-008 was committed and contradicted within three days — the OS version and the
vCPU count both changed in conversation and neither reached the file. Nothing
about that was a large decision.

## Where a change lands

| What changed | File to update |
|---|---|
| A design decision with lasting consequences | The relevant `adr/` record, plus `adr/README.md` if status changed |
| An IP address, a segment, an allocation range | `docs/address-plan.md` |
| A VM's vCPU, RAM, or disk; a runtime profile | `docs/resource-budget.md` |
| A host built, retired, or renamed | `docs/address-plan.md`, `docs/topology.md`, `README.md` status table |
| A NIC, VMnet, or interface mapping | `docs/interface-mapping.md` |
| A firewall rule or an intended segment flow | `docs/firewall-policy.md` |
| An evaluation clock, rearm, or expiry | `docs/licensing-clock.md` |
| A build procedure that turned out wrong | The relevant `runbooks/` file |
| A real fault, diagnosed | `docs/troubleshooting-log.md`, next number |

Most changes touch more than one. A new host touches four.

## Amend, or supersede?

An ADR records reasoning, not just an outcome. How you change one depends on
whether the reasoning still holds.

**Amend in place** when the decision stands and a detail moved. ANS01 changing
from 24.04 to 26.04, or from 1 vCPU to 2, does not disturb "use Ansible from a
dedicated Linux control node on CORE." Edit the consequences, state what changed
and why, leave the status Accepted.

**Supersede** when the reasoning itself failed. Write a new ADR that explains why
the old one was wrong, and mark the old one `Superseded by ADR-0NN`. Do not
delete or rewrite it. ADR-007 already carries a supersession plan for Phase 3 and
is the model.

The test: would someone reading only the amended file understand the decision the
same way? If yes, amend. If the *rationale* changed, they need both documents.

## Corrections to the troubleshooting log

Entries are append-only. When a later entry proves an earlier one wrong, annotate
the original — do not rewrite it — and write the correction as a new entry.

Entry 11 concluded that `slmgr /rearm` did nothing; entry 12 established that it
had been checked before a reboot applied it. Entry 11 keeps its original text plus
a pointer forward. The wrong conclusion is the teaching material; deleting it
would leave a log that has never been wrong about anything, which is not credible
and not useful.

## Work locally, not on github.com

**Edit files in the working copy, commit, push.** A file changed through the
GitHub web interface creates a commit the local clone has never seen, and the
next push is rejected as a non-fast-forward. Recovering costs a
fetch-inspect-rebase cycle every time — more than the edit saved. This happened
twice in one session; see troubleshooting entry 13.

Reserve the web interface for things with no local equivalent: repository
settings, secret scanning and push protection, branch protection rules.

The general form is worth holding on to. **A change made outside your working copy
is invisible until you fetch.** The web UI, a second machine, and a collaborator
all produce the same divergence, and `git fetch` followed by inspecting both
commit ranges is the same first move in all three cases.

## Pre-commit checklist

Before every commit, ask four questions:

1. **Does anything I decided this session contradict a file I did not touch?**
   Addresses, RAM, vCPU, OS versions, and status tables are the usual suspects.
2. **Does any status still say Planned or Not started for something that exists?**
   `docs/address-plan.md`, `docs/topology.md`, and the `README.md` table all carry
   status and all drift independently.
3. **Did anything fail today?** If yes, it earns a troubleshooting entry now,
   while the diagnosis is still in your head.
4. **After committing, does `git status` read clean?** Anything still listed as
   modified was meant to be in that commit and was not staged. A written and
   placed file that never reached the `git add` line is invisible otherwise — it
   happened to `docs/git-workflow.md`, which was rewritten, saved, and left out of
   the commit that was supposed to carry it.

Then read the diff for every staged file. `git diff --cached` before committing;
that is also the step that catches a password pasted into a command block.

## Verify before recording

Two failure modes have produced faults in this project, and both are about
recording something that was never checked.

**An asserted path.** Four faults came from file paths, filenames, and menu
locations stated from memory — `vmx0` versus `em0`, `setup64.exe` versus
`setup.exe`, the VMXNET 3 advice, the Advanced button. Confirm with
`Get-ChildItem`, `Test-Path`, `Get-Volume`, or `ls` and write down what came back,
not what you expected.

**A staged change checked before it applied.** `Rename-Computer` (entry 08),
interface assignment (entry 02), and `slmgr /rearm` (entries 11 and 12) all report
success while doing nothing until a reboot or a confirmation step. Re-check after
the reboot. A command's return value is not a state change.

Anything recorded as fact but not actually verified gets labelled as a hypothesis,
in the file, with the check that would settle it. Entry 12 does this.

## Project-folder sync

Five files are loaded as Claude project knowledge so conversations start with
context. They are copies and they go stale silently, because nothing warns you:

- `README.md`
- `docs/address-plan.md`
- `docs/resource-budget.md`
- `docs/change-control.md`
- `CLAUDE.md` — project knowledge only, deliberately not in the repo

**After any commit that touches the first four, re-upload them.** A stale project
copy is worse than none: it produces confident advice built on a state that no
longer exists. That is how a decision gets made against a nine-gigabyte Profile A
that has not been accurate for a week.

This file is on the list deliberately. A change-control protocol that is not
loaded into the session it governs protects nothing.

`CLAUDE.md` is the exception. It exists only as project knowledge and has no repo
copy to drift from. Its two portable conventions — the hostname pattern and the
verify-don't-assert rule — are restated in `README.md` so the repo stands alone.

## Phase boundaries

At the end of each phase, before starting the next:

- Every ADR for the phase written and its status current
- `README.md` status table and roadmap reflect completion
- Runbook exists for the phase, amended for anything that went differently
- Address plan, topology, and resource budget match what is actually running
- Every fault has an entry
- Project-folder copies re-uploaded

The commit history is a portfolio artifact in its own right. A phase that closes
with four documents contradicting each other is visible to anyone who reads it.
