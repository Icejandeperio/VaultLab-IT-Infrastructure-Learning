# Git Workflow

Work is done from PowerShell in the repository root, `C:\Lab\Docs\vaultlab`.

## One-time setup

```powershell
cd C:\Lab\Docs\vaultlab
git init
git branch -M main
git config user.name  "Your Name"
git config user.email "you@example.com"
```

Create an **empty** repository on GitHub — no README, no licence, no .gitignore,
since this repo already has them. Then:

```powershell
git remote add origin https://github.com/<user>/vaultlab.git
git add .
git commit -m "Initial commit: repo scaffold, ADRs, address plan"
git push -u origin main
```

## Secret scanning

Enable in GitHub repository settings:

- **Settings → Code security → Secret scanning** — on
- **Push protection** — on. Blocks a push containing a recognised credential
  pattern rather than reporting it afterward.

Locally, install [gitleaks](https://github.com/gitleaks/gitleaks) and run before
pushing:

```powershell
gitleaks detect --source . --verbose
```

**A leaked secret stays leaked.** Deleting the file in a later commit does not
remove it from history — the object remains and GitHub's search indexes it.
Recovery means rewriting history *and* rotating the credential. Treat every
commit as permanent.

## The normal loop

```powershell
git status
git add <specific paths>
git diff --cached
git commit -m "Subject line

Body explaining why."
git push
```

`git diff --cached` shows what is staged, which is the step that catches a
password pasted into a command block. Stage specific paths rather than `git add .`
until reviewing every diff is automatic.

### Multi-line commit messages in PowerShell

PowerShell keeps reading a quoted string across newlines, so a message can be
written inline:

```powershell
git commit -m "Subject line here

First paragraph of the body.

Second paragraph."
```

The prompt changes to `>>` while the string is open and the commit runs when the
closing quote arrives. If a paste breaks mid-string, or the terminal submits at
the first newline, fall back to repeated `-m` flags — each becomes its own
paragraph, and that form works in `cmd` as well:

```powershell
git commit -m "Subject line" -m "First paragraph." -m "Second paragraph."
```

## Commit messages

Explain **why**, not what. The diff already shows what.

Bad:

```
updated firewall
```

Good:

```
Fix SEC and RED rules sourced from wrong segment

Both were sourced from CLIENT net rather than their own segments,
so neither rule could ever match — no host on SEC or RED holds a
10.10.20.0/24 address. Rules were present in the ruleset but
functionally inert, and both segments were silently isolated.
```

That reads like an engineer wrote it. The first does not, and the commit history
is a portfolio artifact in its own right.

## When a push is rejected

```
! [rejected]  main -> main (fetch first)
```

This is not an error in your commits. It means the remote branch holds commits
your local branch does not, usually because something was edited through the
GitHub web interface or from another machine.

**A branch is a pointer to one commit, and each commit names its parent.** A
*fast-forward* is when the remote's current commit is an ancestor of yours — git
can slide the pointer along an unbroken chain, and nothing becomes unreachable.
When the histories have diverged, moving the pointer to yours would orphan
whatever sits on the remote past the split point. Git refuses rather than do that
silently. `--force` does it anyway, and is how people destroy work that is not
theirs.

### Diagnose before integrating

All three of these are read-only:

```powershell
git fetch origin
git log --oneline main..origin/main
git log --oneline origin/main..main
```

`git fetch` downloads the remote's commits and updates `origin/main` **without
touching your `main`**. That separation is the useful part — you inspect before
integrating.

The two-dot ranges read as "commits reachable from the right side but not the
left." The first lists what exists only on the remote; the second, only locally.

### Then rebase

```powershell
git rebase origin/main
git log --oneline -3
git push
```

Rebase detaches your commits, moves the branch pointer to the remote's tip, and
replays your changes on top. The result is a straight line rather than a merge
commit recording a collaboration that did not happen.

**The trade-off:** rebase rewrites commits, so yours get new SHAs — a commit's
hash covers its parent, and the parent changed. Harmless when the commits have not
been pushed. Destructive if someone else has already pulled them.

### If rebase refuses

```
error: cannot rebase: You have unstaged changes.
```

Rebase rewrites files in the working directory, and it will not risk overwriting
changes git has no record of. **Read what they are before clearing them:**

```powershell
git status
git diff --stat
```

Then either commit them, or `git stash` / rebase / `git stash pop`.

`git restore <file>` discards a change permanently and is normally the one command
here with no undo. It is safe only when the content exists somewhere else — for
example when the "change" is a local deletion of a file that is still present in
an earlier commit. Establish that first.

## Cadence

Commit at the end of every working session, not every phase. A year of dated,
methodical commits is far more persuasive evidence of sustained work than any
claim on a CV.

Before committing, run the pre-commit checklist in `docs/change-control.md`. Most
documentation faults in this project were a decision made in one place and never
carried to the file that records it.

## Recommended VS Code extensions

| Extension | Why |
|---|---|
| GitLens | Blame and history inline |
| Markdown All in One | TOC generation, table formatting |
| Markdown Preview Mermaid Support | Renders the topology diagrams locally |
| PowerShell | Syntax and linting for `scripts/` |
| YAML | For `ansible/` in Phase 2 |
| markdownlint | Keeps documentation consistent |
