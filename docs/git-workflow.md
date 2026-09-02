# Git Workflow

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

## Committing in VS Code

The Source Control panel (`Ctrl+Shift+G`) covers the full loop.

1. Review the diff for every changed file before staging. This is the step that
   catches an accidental password in a pasted command.
2. Stage with the `+` next to each file. Avoid "Stage All Changes" until the
   review habit is automatic.
3. Write the message in the box, `Ctrl+Enter` to commit.
4. **Sync Changes** to push.

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

## Cadence

Commit at the end of every working session, not every phase. A year of dated,
methodical commits is far more persuasive evidence of sustained work than any
claim on a CV.

## Recommended VS Code extensions

| Extension | Why |
|---|---|
| GitLens | Blame and history inline |
| Markdown All in One | TOC generation, table formatting |
| Markdown Preview Mermaid Support | Renders the topology diagrams locally |
| PowerShell | Syntax and linting for `scripts/` |
| YAML | For `ansible/` in Phase 2 |
| markdownlint | Keeps documentation consistent |
