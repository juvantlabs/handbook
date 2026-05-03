# Naming conventions

Single source of truth for **how things are named** across the
`juvantlabs` ecosystem: repos, packages, branches, tags, commits,
issue / PR titles. Each repo-type spec ([`docs/repo-types/`](repo-types/))
references this doc rather than duplicating naming rules.

## Namespaces (org / user)

| Namespace | Type | Used for |
|---|---|---|
| `juvantlabs/*` | GitHub user account | OSS framework, MCP servers, libraries, this handbook, audit gists, public toolboxes |
| `juvantio/*` | GitHub organization | Commercial — Juvant Srls per-company instance, Hardys-family product repos, future commercial products |
| `<adopter-org>/*` | external adopter's own GitHub org | Per-company Juvant OS instances mirror-pushed by external adopters |

The `juvantlabs` / `juvantio` split is governed by
[ADR 0001 — GitHub account and organization structure](adr/0001-account-and-org-structure.md).

## Repo names per type

| Type | Pattern | Examples |
|---|---|---|
| **MCP server** | `juvantlabs/<vendor>-mcp-server` | `juvantlabs/finom-mcp-server`, `juvantlabs/aruba-fattura-mcp-server`, `juvantlabs/m365-graph-mcp-server` |
| **Library** | `juvantlabs/<library-name>` (no suffix) | `juvantlabs/engram` |
| **Framework template** | `juvantlabs/<framework-name>-os` | `juvantlabs/juvant-os` |
| **Toolbox (public)** | `juvantlabs/<name>-tools` | `juvantlabs/juvant-tools` (planned) |
| **Toolbox (private, product-coupled)** | `juvantio/<product>-dev-tools` | `juvantio/hardys-dev-tools` (renamed from `juvant-dev-tools` 2026-05-03) |

### Suffix rules

- `-mcp-server` is **mandatory** for MCP server repos. Used by CI rules
  (`docs/repo-types/mcp-server.md`) and by tooling that discovers MCP
  servers in the org (e.g. for `MCP_INVENTORY.md` cross-checks).
- `-os` is **mandatory** for framework templates (current canonical:
  `juvant-os`). When future framework patterns emerge that aren't
  "operating system" shape (e.g. a future `<framework>-platform` or
  `<framework>-runtime`), open a follow-up ADR before adopting a new
  suffix.
- `-tools` (public) / `-dev-tools` (private) are **mandatory** for
  toolboxes. Distinguishes from libraries.
- Libraries have **no suffix**. The repo name is the library's
  identity.

### Vendor / family prefixes for MCP servers

When an MCP server wraps a single vendor's API: `<vendor>` is the
vendor's name, lowercased and hyphenated.

- Single dominant vendor → `<vendor>-mcp-server` (`finom`, `aruba`,
  `mercury`).
- Vendor family wrapping multiple endpoints under one auth surface →
  prefix with the family (`m365-graph` covers OneDrive + SharePoint +
  Calendar through one Microsoft Graph endpoint, named accordingly).
- Italian-fiscal-specific Aruba naming (`aruba-fattura`) reflects the
  scope (fatturazione elettronica) — Aruba runs other unrelated
  services, so the scope qualifier matters.

## Package / registry names

| Registry | Pattern | Notes |
|---|---|---|
| npm (scoped) | `@juvantlabs/<repo-name>` | MCP servers, TypeScript libraries; preserves the `juvantlabs` namespace on npm |
| PyPI | `<repo-name>` if available, else `<repo-name>-<disambiguator>` | If the canonical name is taken (Engram hit this with `engram`, fell back to `engram-browser`), document the discrepancy in README + ARCHITECTURE.md |
| Crates.io / Hex / etc. | `<repo-name>` | Same fallback rule on conflict |

The repo name and the registry name **should match** by default.
Discrepancies are documented; never silent.

## Abstract role names (MCP `agent_tool_matrix` qualifiers)

When an MCP qualifier is **abstract** (multiple providers can fulfill
it — `bank` covers Finom, Mercury, Revolut, Wise, etc.), the abstract
role name follows these rules:

- **Single noun, lowercased, no underscores in role names** unless the
  semantic clearly requires it: `bank`, `buffer`, `github` (as
  `github:read` / `github:write`).
- **Multi-word roles use underscores** when the concept is fundamentally
  compound: `fattura_elettronica` (Italian SDI), `m365_graph` if needed
  (currently called `m365-graph` in matrix entries — TBD by ADR 0002).
- Abstract roles are introduced via the process documented in
  [ADR 0002 — MCP abstract roles + binding policy](adr/0002-mcp-abstract-roles.md)
  (when written): CA proposes → CSO + CA review → CEO approves → COO
  updates `juvantlabs/juvant-os/docs/MCP_INVENTORY.md`.

The `:read` / `:write` qualifier is appended where scope distinction
matters (`github:read` vs. `github:write`; `bank:read` is the only
permitted scope for `bank`).

## Branch names

| Pattern | Use |
|---|---|
| `main` | Default branch on every repo. The template `juvant-os` and the framework's per-company instances both default to `main`. (Note: `juvantlabs/handbook` and `juvantlabs/engram` use `master` historically — fine to leave as-is per repo, but new repos default to `main`.) |
| `<short-slug>` | Working branch for any work. Short, hyphenated, descriptive. No category prefix (`feature/`, `fix/`, etc.) required |
| `dependabot/*` | Automation-managed; never push to manually |
| `revert-<sha>-<branch>` | GitHub-generated revert PR branches |

Working branches are deleted after merge. Long-lived branches other
than `main` (or `master` on legacy repos) are an audit finding —
forks rebase or merge regularly.

## Tag names

| Pattern | Use |
|---|---|
| `v<major>.<minor>.<patch>` | Semver releases on every `juvantlabs/*` repo. Annotated tags only (`git tag -a`); never lightweight |
| `v<major>.<minor>.<patch>-alpha`, `-beta`, `-rc.N` | Pre-release qualifiers when applicable (`v1.0.0-rc.1`) |

Tags drive npm / PyPI publish workflows. Push the tag → CI publishes
the release. Never publish without a tag.

The version-bump rules per repo type are documented in each spec's
"Lifecycle" section. Common rules:

- **patch** — bug fixes, no public API change
- **minor** — new feature, backward-compatible
- **major** — breaking change. Bump to `v1.0.0` indicates committed API
  stability (libraries) or production-ready (frameworks)

## Commit message style

`juvantlabs/*` follows a **Keep a Changelog-aligned** style — not
strict Conventional Commits, but disciplined enough that CHANGELOG
entries can be cross-referenced to commits.

### Subject line

```
<area>: <imperative summary, ≤70 chars>
```

`<area>` examples:

- `docs` — documentation-only changes
- `feat` — new functionality
- `fix` — bug fix
- `refactor` — code restructure with no behaviour change
- `chore` — dependency bumps, CI tweaks, formatting
- `build` — packaging / release infrastructure
- `test` — test suite changes
- `<feat-id>` — when a commit is the canonical landing of a tracked
  FEAT issue (e.g. `feat(v0.5.0): OSS template shipping defaults
  (FEAT-013)`)

### Body

Free-form. Wrap at 72 chars. Cite issue / FEAT / ADR numbers when
applicable. Use Markdown freely (especially bullet lists for
multi-change commits).

### Co-author tags

When AI-assisted, include:

```
Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

at the end of the commit message. This is set by the AI tool and
should remain in commits to maintain audit trail.

### Examples

Good:

```
fix(v0.4.2): document storage folder mapping + null-binding policy

Closes the 5 spec/template gaps surfaced during the 2026-05-03
dogfood OneDrive folder-mapping discussion (juvantio/juvant
instance) — wizard / template gaps only, no functional regression.

JUVANT_OS.md:
- New Step 1.5 "Document storage folder mapping" between Step 1
  (Identity) and Step 2 (Database). Discover-via-tool path...
- ...

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

Avoid: subject without `<area>:` prefix; subject in past tense
("fixed bug"); body that describes WHAT (visible in diff) instead of
WHY (necessary in commit).

## Issue / PR title style

Same `<area>:` prefix convention as commits. PR titles match the
intended squash-commit subject so the merged commit is well-formed.

Issue titles should be **descriptive and searchable**:

- ✅ `FEAT-014 M365 Graph MCP server — read+write across OneDrive,
  SharePoint, Calendar`
- ❌ `M365 server`
- ❌ `Need to add MCP for graph stuff`

The convention `FEAT-NNN`, `OP-NNN`, `ARCH-NNN` (legacy, now ADRs)
embeds the issue identity in the title for cross-reference.

## Filenames

### ADR files

`<NNNN>-<kebab-case-title>.md` where `<NNNN>` is monotonic 4-digit zero-
padded. See `juvantlabs/handbook/docs/adr/README.md` and
`juvantlabs/juvant-os/docs/adr/README.md` for the per-repo conventions
(both follow this pattern).

### Spec files

`<topic>.md` in kebab-case under `docs/<area>/` (e.g.
`docs/repo-types/mcp-server.md`, `docs/security/disclosure-process.md`).

### Audit report gists

`<owner>-<repo>-security-review-<YYYY-MM-DD>.md` (per
[`docs/security/audit-report-template.md`](security/audit-report-template.md)).

Example: `mcp-onedrive-sharepoint-security-review-2026-05-03.md` for
the ftaricano audit.

## Copyright + LICENSE

The **copyright holder** for every `juvantlabs/*` repo is **`Juvant Srls`**
— the Italian S.r.l. (limited liability company) that owns the OSS work
product. Not `juvantlabs`, which is a GitHub user account / namespace
identifier and has no legal personhood.

### MIT LICENSE convention

For repos shipped under MIT (the default for `juvantlabs/*-mcp-server`,
libraries, framework templates, and OSS-shareable toolboxes):

```
MIT License

Copyright (c) <year> Juvant Srls

Permission is hereby granted, free of charge, to any person obtaining a copy
...
```

`<year>` is the year of first commit on the repo (or first publication).
Don't update the year on every commit — semantic copyright dating uses the
year of original work; the line is conventionally a single year, not a
range, unless the repo has been actively rewritten across multiple years.

### CC0 dedication convention

For repos that dedicate to public domain (the default for `juvantlabs/handbook`
documentation):

```
CC0 1.0 Universal (CC0 1.0)
Public Domain Dedication

To the extent possible under law, Juvant Srls has waived all copyright
and related or neighboring rights to the documentation in this repository
under the CC0 1.0 Universal Public Domain Dedication.

[then the standard CC0 legal text]
```

### Why this matters

- Copyright is held by **legal persons** (humans or companies), not by
  GitHub usernames. A LICENSE line that names "juvantlabs" as copyright
  holder is technically defective — `juvantlabs` cannot hold copyright
  because it has no legal personhood.
- Future legal events (acquisition, IP transfer, dissolution, bankruptcy)
  affect the **legal entity**, not the GitHub identity. Naming Juvant Srls
  as copyright holder makes the chain of custody legally clear.
- Per [ADR 0001](adr/0001-account-and-org-structure.md), `juvantlabs` is
  the OSS user account; `juvantio` is the commercial organization; both
  ultimately roll up to Juvant Srls (the founding legal entity). Future
  re-evaluation triggers (acquisition, foundation transition) may change
  this; ADR 0001's modification flow governs.

### When the copyright holder differs

For an external community contributor's substantial PR, the contributor
retains copyright on their changes (per the standard MIT inbound = MIT
outbound convention). The repo's LICENSE line still lists Juvant Srls as
the original-work copyright holder; the contributor's authorship is
preserved through git history + co-author tags, not through LICENSE
edits.

If a future repo forks this convention (e.g. a `juvantlabs/*` foundation-
governed project), document the deviation in the repo's LICENSE
+ a corresponding ADR.

## What this doc does NOT cover

- **Code-level naming conventions** (variables, functions, classes).
  Those are language-specific and live in each repo's own style guide
  (or the language's canonical conventions: PEP 8 for Python, ESLint
  defaults for TypeScript, etc.).
- **Per-product naming inside `juvantio/*`** (e.g. how Hardys names
  its modules). Defined privately during each product's build.
- **Database schema / column naming**. Per-repo concern; framework
  template `juvant-os` documents its own conventions in
  `scripts/schema.sql` comments.

## Modification

Naming conventions evolve when the ecosystem grows. PRs against this
doc include a brief rationale ("we added a 5th repo type, here's the
naming pattern", "PyPI naming conflict with Engram surfaced; documented
the fallback rule"). Material changes (e.g. introducing a new repo-name
suffix, changing the abstract role syntax) warrant a corresponding ADR.

Trivial fixes (typos, link rot) are out-of-band, no rationale needed.
