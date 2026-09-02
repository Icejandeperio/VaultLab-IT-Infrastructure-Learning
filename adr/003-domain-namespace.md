# ADR-003 — Domain namespace: corp.vaultlab.net

**Status:** Accepted

## Context

Active Directory requires a DNS namespace chosen at forest creation. Changing it
afterward is a forest rebuild, not a configuration change.

## Decision

Forest root domain `corp.vaultlab.net`, NetBIOS name `VAULTLAB`. Resolved only
internally by DC01; not registered publicly.

## Alternatives considered

**`vaultlab.local`** — rejected. The `.local` TLD is reserved for multicast DNS
(mDNS/Bonjour). Using it causes resolution conflicts on networks with Apple
devices or Avahi, and Microsoft has advised against it for over a decade.

**Single-label domain (`VAULTLAB`)** — rejected. Breaks DNS delegation and is
formally unsupported by Microsoft.

**A subdomain of an owned public domain** — the current best practice, and what
would be used in production. Deferred only because no domain is currently owned.

## Consequences

- `corp.` as a subdomain leaves room for a separate forest or a resource domain
  later without renaming.
- The internal zone is unsigned and cannot be validated against the public DNS
  root, so DNSSEC validation on the upstream resolver is disabled. If DNSSEC is
  enabled later, a negative trust anchor for `corp.vaultlab.net` is required.
