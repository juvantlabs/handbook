# ADR 0001 — GitHub account and organization structure (juvantlabs / juvantio)

## Status

Accepted (2026-05-03). Codifies a structural decision implicit since the
two-namespace model was introduced in
[`juvantlabs/juvant-os-pm/docs/session-commit-p1.md`](https://github.com/juvantlabs/juvant-os-pm/blob/main/docs/session-commit-p1.md)
on 2026-04-23 and operationalized through every subsequent feature, ADR,
and dogfood action since.

This is the first ADR on the org-level handbook. Earlier framework-level
ADRs (0001–0008 over at `juvantlabs/juvant-os/docs/adr/`) cover Juvant OS
architecture; this ADR covers how juvantlabs **the org** structures
itself.

## Context

The Juvant project has two distinct concerns that map naturally onto two
GitHub namespaces:

- **OSS framework** — the `juvant-os` template, its PM repo, the
  `*-mcp-server` family, framework ADRs, security audits,
  contribution-friendly artifacts. Designed for adopters worldwide; MIT
  / CC0 licensed; no commercial entanglement.
- **Commercial product / per-company instance** — Juvant Srls's own
  Juvant OS instance, plus product repos (Hardys family, future
  product lines). Private, business-owned, governed by Juvant Srls's
  commercial interests.

The question: should these two concerns share a single GitHub account,
live in completely separate accounts, or use a hierarchical model with
semantic separation?

## Decision

**`juvantlabs`** (a GitHub user account) hosts the OSS framework family.
**`juvantio`** (a GitHub organization, accessible via the same primary
identity) hosts the commercial product family. The two share
authentication and tooling but are semantically separated by namespace.

| Namespace | Type | Scope | Example repos |
|---|---|---|---|
| `juvantlabs/*` | User account | OSS framework — template, PM tracker, MCP servers, this handbook, audits | `juvantlabs/juvant-os`, `juvantlabs/juvant-os-pm`, `juvantlabs/handbook`, `juvantlabs/finom-mcp-server` (planned), `juvantlabs/m365-graph-mcp-server` (planned) |
| `juvantio/*` | Organization | Commercial — per-company Juvant OS instance + product repos | `juvantio/juvant`, `juvantio/hardys-pm`, `juvantio/hardys-core`, `juvantio/hardys-app`, … |
| `<adopter-org>/*` | External adopter | Per-company Juvant OS instance for any other company adopting the framework | `acme/acme-os`, etc. |

### Convention — what goes where

Anything **OSS-shareable** lives under `juvantlabs/*`:

- The Juvant OS template (`juvant-os`).
- Project-management for the OSS framework (`juvant-os-pm`).
- This handbook (`handbook`).
- MCP servers built MIT-clean as canonical implementations
  (`*-mcp-server`).
- Framework-level Architecture Decision Records (within
  `juvant-os/docs/adr/`).
- Security audits / disclosures conducted as juvantlabs work (gists,
  public reports).

Anything **commercial / instance-specific** lives under `juvantio/*`:

- Juvant Srls's per-company Juvant OS instance (`juvant`).
- Product repos for any Juvant-Srls-owned product (Hardys family
  today; others in the future).
- Private operational documentation not relevant to OSS adopters.

**External adopters** of the framework do NOT touch either namespace.
They mirror-push the OSS template into their own GitHub org and operate
from there. The two-namespace model documented in this ADR is
juvantlabs-specific; adopters are encouraged to follow a similar pattern
(an OSS-leaning identity vs. a commercial-product identity) but the
exact choice is theirs.

## Consequences

### Positive

- **Single primary identity for solo-founder operation.** Auth, 2FA, gh
  CLI tokens are shared across both namespaces — operationally
  lightweight for the early stage.
- **Clear semantic separation.** Adopters reading the framework know to
  look at `juvantlabs/*` for OSS artifacts and ignore `juvantio/*`
  (which is the canonical product instance, not their concern).
  Minimizes confusion about where to file bugs, where to PR, where to
  look for releases.
- **Aligned with industry pattern.** Microsoft has `microsoft`
  (commercial) and `dotnet` / `Azure-Samples` (OSS-friendly arms);
  Anthropic uses `anthropics` for everything; many startups split
  similarly. The juvantlabs / juvantio split is a recognizable shape.
- **Cross-cutting work stays auditable.** A feature that touches both
  template (`juvantlabs/juvant-os`) and instance (`juvantio/juvant`)
  does so under the same primary identity; the audit trail is unified.
- **Reusable OSS toolchain.** npm publish credentials, GitHub Actions
  secrets, CI configurations for the `juvantlabs/*-mcp-server` family
  centralize naturally.

### Negative

- **No legal isolation between OSS and commercial.** If the commercial
  entity faces an issue (bankruptcy, acquisition, dispute), the OSS
  work under `juvantlabs` is not formally insulated. Mitigated today
  by the shared primary identity; will need to be revisited if the
  commercial entity is sold or restructured.
- **Single 2FA failure mode.** A compromise of the primary GitHub login
  affects both namespaces. Mitigated by hardware key + recovery codes;
  acceptable risk profile for solo-founder operation.
- **Convention fragility.** If contributors don't read this ADR, they
  may file commercial issues at `juvantlabs/*` or OSS PRs at
  `juvantio/*`. Mitigated by README pointers (this repo's README and
  `juvantlabs/juvant-os/README.md`), but real-world drift will happen.

## Re-evaluation triggers

This ADR is correct **for solo-founder, early-stage operation**. It
should be revisited (and likely superseded) when any of the following
happens:

1. **First OSS contributor outside the founding identity** joins
   `juvantlabs/juvant-os` (or any other juvantlabs/* repo) with write
   access. The single-account model means the contributor implicitly
   has potential access pathways to commercial repos via account
   compromise or social engineering. At this point, `juvantlabs`
   should be migrated from a personal user account to a true GitHub
   organization with explicit team-membership controls.

2. **The commercial entity hires employees** who work only on
   commercial products. They should never have access to
   `juvantlabs/*` OSS work. The current `juvantio` organization
   already handles this scoping correctly, so this trigger reinforces
   the value of the existing split rather than forcing a structural
   change. But it tests the convention.

3. **Acquisition / spin-off / IP transfer** of any commercial product
   from the commercial entity. The OSS contributions made by the
   founding identity personally may need to be formally separated
   from commercial business interests. At this point, consider
   migrating `juvantlabs` to a foundation or independent legal entity
   (cf. Apache Software Foundation, Linux Foundation governance
   models).

4. **Material OSS adoption** (≥ 10 external companies running their
   own `<adopter>/<juvant-os-instance>` mirror-pushed instances)
   creating community governance pressure. At that point, the OSS tier
   benefits from a foundation-style identity that is not visibly
   owned by a single commercial vendor.

None of these triggers are imminent in 2026. Re-check at v1.0 release
of `juvantlabs/juvant-os` and annually thereafter.

## Implementation

The convention is codified in:

- This ADR (`docs/adr/0001-account-and-org-structure.md` in this
  repository).
- The handbook README at the repo root, which explains the distinction
  to external readers.
- `juvantlabs/juvant-os/README.md` and
  `juvantlabs/juvant-os/JUVANT_OS.md` Appendix B — reference
  `juvantlabs/juvant-os` as the OSS template source and
  `<your-org>/<company-slug>` as the per-company destination,
  illustrating the model from the adopter's perspective.
- `juvantlabs/juvant-os/docs/MCP_INVENTORY.md` and
  `juvantlabs/juvant-os/docs/branch-protection-spec.md` — written as
  if shipped from `juvantlabs/*` to per-company instances.

No code change is required by this ADR; it documents existing structure.

## References

- [`juvantlabs/juvant-os-pm/docs/session-commit-p1.md`](https://github.com/juvantlabs/juvant-os-pm/blob/main/docs/session-commit-p1.md)
  — first appearance of the two-namespace model (2026-04-23).
- [`juvantlabs/juvant-os/docs/adr/`](https://github.com/juvantlabs/juvant-os/tree/main/docs/adr)
  — framework-level ADRs that cite this org-level structure
  (notably ADR 0001 Skill-first architecture and ADR 0005 Portal
  agent variants reference the per-company-instance pattern).
- [Project memory `feedback_oss_genericity.md`](https://github.com/juvantlabs/juvant-os/blob/main/.claude/projects/.../memory/feedback_oss_genericity.md)
  (private to the contributor's session memory) — the OSS-template
  artifacts under `juvantlabs/*` use generic placeholders, never
  Juvant-Srls-specific names; the convention is enforced by audit
  (CSO Layer 5 placeholder substitution rule).
