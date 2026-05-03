# Contributing to `juvantlabs/*`

Thanks for considering a contribution. This document is the **meta
contributor guide**: it covers how contributions work across all
`juvantlabs/*` repositories. Each individual repo may have its own
`CONTRIBUTING.md` with repo-specific build / test instructions; that
file follows the conventions documented here and links back to this
handbook.

## Where to file what

Different concerns go to different places:

| Concern | Where |
|---|---|
| Bug in a specific repo (e.g. `juvant-os`, `engram`, `finom-mcp-server`) | Issue on **that repo's** GitHub tracker |
| Feature proposal for `juvant-os` | Issue on `juvantlabs/juvant-os-pm` (the PM tracker for the framework) |
| Architectural question / proposal touching `juvant-os` framework | ADR drafted via PR against `juvantlabs/juvant-os/docs/adr/` |
| Org-level governance question (naming, conventions, policy across repos) | Issue or PR on `juvantlabs/handbook` (this repo) |
| Org-level architectural decision | ADR drafted via PR against `juvantlabs/handbook/docs/adr/` |
| Security vulnerability | Per the [Security Disclosure Process](docs/security/disclosure-process.md) — **never as a public issue** |
| Question about how something works | Discussion on the relevant repo (or this repo, if cross-cutting) |

When in doubt, open an issue on the most likely repo and a maintainer
will redirect it.

## Before you start

1. Read the **handbook README** ([this repo's `README.md`](README.md))
   to understand the OSS arm's structure.
2. If contributing to a specific repo, read **its own `README.md`** for
   build / test / run instructions.
3. If contributing a substantial change (anything beyond a typo /
   doc fix), **open an issue first** to discuss the direction. This
   avoids wasted work on PRs that won't merge.
4. For new repos under `juvantlabs/*`, read the matching
   [`docs/repo-types/<type>.md`](docs/repo-types/) spec and follow it.

## Pull request process

### Branching

- Branch from `main` of the target repo.
- Branch name: short, hyphenated, descriptive — no category prefix
  (`feature/`, `fix/`) required. Examples: `fix-token-refresh`,
  `add-finom-list-accounts`, `docs-naming-cross-ref`.
- Push to your fork (or directly to the upstream branch if you have
  write access).

### Commit style

Follow the [naming conventions](docs/naming.md#commit-message-style):

- Subject line: `<area>: <imperative summary, ≤70 chars>`
- `<area>` examples: `docs`, `feat`, `fix`, `refactor`, `chore`,
  `build`, `test`, or a tracked-issue reference like `fix(v0.4.2)` or
  `feat(FEAT-011)`.
- Body: free-form Markdown. Wrap at 72 chars. Cite issues / FEATs /
  ADRs when applicable. Explain the **why**, not the what.
- AI-assisted commits include the standard co-author tag at the end:

  ```
  Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
  ```

### PR title + body

- **PR title** matches the intended squash-commit subject (`<area>: ...`).
- **PR body** explains rationale, links the related issue / FEAT / ADR,
  surfaces non-obvious tradeoffs, includes a "Test plan" section when
  the change has runtime implications.

### Testing

Per the matching [`docs/repo-types/`](docs/repo-types/) spec for the
target repo:

- **MCP servers** — unit + integration tests, ≥ 80% coverage on real
  code paths, no `console.log` introduced, lint clean.
- **Libraries** — unit tests on every public surface, integration tests
  for I/O paths, lint + type-check clean.
- **Framework templates** (`juvant-os`) — markdown lint, YAML
  frontmatter parses, schema syntax check, `.claude/settings.json`
  parses, no tracked secrets. Plus dogfood validation for material
  changes (run a fresh `Initialize Juvant OS` against the change).
- **Toolboxes** — smoke test (each tool invokes with `--help` and
  returns exit 0); deeper tests for production-critical scripts.
- **Handbook** (this repo) — markdown link check, no broken
  cross-references between docs.

CI runs the relevant checks automatically. PRs that fail CI don't
merge.

### Review

- **CODEOWNERS** governs review requirements per file path. The
  `.github/CODEOWNERS` file in each repo is generated from
  `github_user_map` (per `juvant-os` Step 1.6) or specified directly
  by the repo's maintainers.
- A PR needs **at least one approving review** from a CODEOWNER for
  the touched paths.
- For sensitive changes (security posture, Universal Boundaries,
  invariants doc edits), additional reviewers (CA + CSO joint, or
  CHRO + CA + CEthO depending on the area) are required by the
  framework's invariants doc. Trust the CODEOWNERS file; if a path's
  ownership is unclear, ask in the PR.
- **Maintainer's prerogative**: the primary maintainer (today: Antonio
  Gatti) may merge their own PRs after review, but should leave a
  short comment confirming review on substantive changes for the
  audit trail.

### Squash vs. merge

- Default: **squash and merge** for feature branches (one commit per
  PR landed on `main`).
- Exception: long-running PRs with logical multi-commit history may
  use **rebase and merge** to preserve the structure. Specify in the
  PR body if you want this.
- **No merge commits** in `main`. Linear history is required by the
  branch-protection convention (per
  [`juvantlabs/juvant-os/docs/branch-protection-spec.md`](https://github.com/juvantlabs/juvant-os/blob/main/docs/branch-protection-spec.md)).

## Code of conduct

All `juvantlabs/*` spaces (issues, PRs, discussions, gists, audit
threads) operate under the
[Code of Conduct](docs/contributing/code-of-conduct.md).
TL;DR: be respectful, focus on technical substance, no harassment.
Maintainers enforce; report incidents to `conduct@juvant.io`.

## AI-assisted contributions

Substantial portions of `juvantlabs/*` codebases and docs are produced
with AI assistance (today: Claude Opus 4.7 with 1M context). This is
normal practice and welcomed. Conventions:

- **Co-author attribution**: AI-assisted commits include the
  `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>`
  tag at the end of the commit message. Don't strip it; it's part of
  the audit trail.
- **Human accountability**: every AI-assisted PR has a human author
  (the GitHub user who pushed the commits). The human is responsible
  for the change — for verifying it works, for reviewing the diff
  before merge, for responding to review feedback. AI assistance is a
  productivity tool, not an author substitute.
- **No magic**: PR bodies explain the change in human terms, regardless
  of how it was produced. "Generated by AI" is not an explanation.
- **Tests are non-negotiable**: AI-produced code goes through the same
  test bar as human-produced code.

## What constitutes a "substantial change"

A short typo / link fix / formatting change can land via a single PR
with minimal explanation. A change that:

- Modifies API surface (any breaking-or-near-breaking change to public
  interfaces),
- Touches security posture (auth, scopes, validators, sandbox boundaries),
- Edits an invariants doc (`SYSTEM_INVARIANTS.md`, this handbook's
  ADRs),
- Adds / removes a tool from an MCP server,
- Changes a `repo-types/` spec or `naming.md` convention,
- Renames or moves files in non-trivial ways,

is **substantial** and warrants:

1. An issue to discuss the direction first.
2. A PR with rationale + tests + test plan in the body.
3. Joint review (per the relevant CODEOWNERS / invariants).

When in doubt, lean toward "open an issue first". The cost of a
short discussion thread is much lower than the cost of a closed PR.

## Maintainer notes (today)

The current maintainer profile:

- **Primary identity**: Antonio Gatti (founder, juvantlabs primary
  account, sole reviewer for most PRs today).
- **AI assistant**: Claude Opus 4.7 (1M context) — co-author on a
  meaningful share of work in 2026.
- **External contributors**: none yet (as of 2026-05-03). When the
  first arrives, ADR 0001's re-evaluation triggers fire — the
  juvantlabs user account migrates to a true GitHub org with
  team-based access controls.

If you're considering a substantial contribution as an external
party, please open an issue first. The handbook's modification
governance assumes a small, slow-moving review cohort; we'll adjust
processes as the contributor base grows.

## Trademark + naming

`Juvant`, `juvantlabs`, `juvantio`, `Juvant OS` are identities of
Juvant Srls. Don't use them in derivative projects without explicit
permission. The MIT and CC0 licenses on individual repos cover code
and documentation re-use; they do not cover identity / brand.

When forking or mirror-pushing a `juvantlabs/*` repo for your own
use (the standard adoption pattern for `juvant-os`), feel free to
publish at `<your-org>/<your-name>` — that's the design. But don't
re-publish under a `juvantlabs`-derivative name (`juvantlabsv2`,
`juvant-extended`, etc.) without coordination.

## Release process

Per the matching `docs/repo-types/<type>.md` spec for each repo:

- **MCP servers + libraries** — semver releases on tag push (`v*`);
  CI runs full audit + tests + publish workflow.
- **Frameworks** (`juvant-os`) — semver tagged releases;
  formal GitHub Releases with structured release notes; CHANGELOG
  entry per release.
- **Toolboxes** — release discipline scales with maturity (no
  semver for ad-hoc; semver + tag-push when promoted to registry).
- **Handbook** (this repo) — no formal releases; `main` is the
  source of truth. Material decision changes are codified as ADRs;
  the ADR's `Accepted` date is its release.

## Questions

- General handbook / governance: open an issue here.
- Framework architecture: open an issue at
  [`juvantlabs/juvant-os-pm`](https://github.com/juvantlabs/juvant-os-pm).
- Per-repo questions: open an issue on that repo.
- Security: see [docs/security/](docs/security/).

Welcome aboard. The OSS arm of Juvant runs on contribution discipline +
clear conventions + good faith. Bring all three.
