# Org-level Architecture Decision Records

Governance, principles, organizational structure, and process decisions
for **juvantlabs as a whole**. Distinct from framework-level ADRs, which
live at [`juvantlabs/juvant-os/docs/adr/`](https://github.com/juvantlabs/juvant-os/tree/main/docs/adr).

ADRs are immutable. An ADR is `Accepted` when it is in force; if a later
decision overrides it, both the new ADR and the original remain — the old
one is annotated `Superseded by NNNN`. Decisions never disappear from the
record.

## Distinction — org-level vs. framework-level

| Lives here (org-level) | Lives in `juvantlabs/juvant-os/docs/adr/` (framework-level) |
|---|---|
| GitHub account / org structure | Skill-first orchestrator (no CLI, no daemon) |
| Contribution policy | Turso as cloud state store |
| Security disclosure process | Bootstrap Protocol |
| MCP-server naming + licensing convention | Single-Writer Invariant |
| OSS-genericity rules | PreCompact hook for context management |
| Code-of-conduct / community governance | Manifesto fast-start (Tier 1 + Tier 2) |

If an ADR is about how the **Juvant OS framework** works architecturally,
it goes to the framework's own `docs/adr/`. If it's about how **juvantlabs
the org** operates and makes decisions across multiple framework projects,
it goes here.

## Index

| # | Title | Status | First decided |
|---|---|---|---|
| [0001](0001-account-and-org-structure.md) | GitHub account and organization structure (juvantlabs / juvantio) | Accepted | 2026-05-03 |
| [0002](0002-mcp-abstract-roles.md) | MCP abstract roles + binding policy | Accepted | 2026-05-03 |

## Modification governance

Amendments to an Accepted ADR follow a standard supersession flow:

1. Proposer drafts a successor ADR (any contributor; for org-level
   decisions affecting the OSS arm, juvantlabs as primary identity).
2. PR opened against this repo with the new ADR file + an annotation
   `Superseded by NNNN` on the original.
3. Review involves at minimum: juvantlabs primary identity (today:
   Antonio Gatti) + any active OSS contributor with write access.
4. Merge once the review thread converges.
5. The new ADR's `Status` is `Accepted` from merge time; the original's
   `Status` is updated to `Superseded by NNNN` in the same PR.

For decisions that touch both org-level (here) AND framework-level
(`juvantlabs/juvant-os/docs/adr/`), draft both ADRs in parallel and
cross-reference. Avoid making a single ADR span both repos — the
distinction is load-bearing.

Trivial fixes (typos, broken links, formatting) are out-of-band and do
not require a successor ADR.

## Conventions

- Numbering is monotonic, zero-padded to four digits.
- Filenames are kebab-case, prefixed with the ADR number.
- All ADR text is in English. No exceptions.
- ADRs may freely cite the framework-level ADRs at
  `juvantlabs/juvant-os/docs/adr/` (and vice versa) when context
  warrants.
