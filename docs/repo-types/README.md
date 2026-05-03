# Repo types

Specifications for the canonical kinds of repository that live under the
`juvantlabs` namespace. Each spec defines the **purpose, distribution
model, stack, required structure, conventions, anti-patterns, and
canonical reference** for that repo type.

These specs are **the source of truth** when creating a new repo. AI
agents and human contributors alike read the spec before scaffolding;
the goal is that anyone (or any LLM) given the instruction "create a
new <type> for <vendor/purpose>" can produce a conforming repo by
following the spec literally.

> **Commercial product repos are out of scope here.** Per-product
> conventions for `juvantio/*` (Hardys family, future Juvant Srls
> products) are defined during each product's own build, in the
> product's own private specs. This handbook covers OSS-shareable
> repo types under `juvantlabs/*` only.

## The four canonical types

| # | Type | Distribution | Examples (today) |
|---|---|---|---|
| 1 | [MCP server](mcp-server.md) | `npm publish @juvantlabs/<vendor>-mcp-server`, runnable via `npx` | (planned) finom-mcp-server, m365-graph-mcp-server, aruba-fattura-mcp-server |
| 2 | [Library](library.md) | per-language registry (PyPI for Python, npm for TS, etc.) | [Engram](https://github.com/juvantlabs/engram) |
| 3 | [Framework template](framework.md) | `git push --mirror` to adopter org | [juvant-os](https://github.com/juvantlabs/juvant-os) |
| 4 | [Toolbox](toolbox.md) | `git clone` (+ optional language-registry install) | [juvant-tools](https://github.com/juvantlabs/juvant-tools) (scaffold-only, first tool incoming) |

## Decision tree — choose your repo type

When creating a new repo under `juvantlabs/`, walk this tree:

```
1. Does it wrap a vendor's REST / Webhook API for agent consumption
   via MCP protocol?
   → mcp-server.md

2. Is it importable code with a stable API, no MCP wrapper, no
   runtime side-effects (pure library / SDK / utility module)?
   → library.md

3. Is it a complete operating-system scaffold that adopters
   mirror-push to their own org and customize for their company?
   → framework.md

4. Is it a collection of utility scripts, dev tools, automation
   helpers, or CLI commands?
   → toolbox.md

5. None of the above?
   → Open an issue at juvantlabs/handbook to discuss adding a 5th
     category. Do not stretch one of the existing four to fit; the
     specs become diluted and audits get harder. New categories are
     cheap to add when the pattern repeats (n ≥ 2).
```

## Per-type spec structure

Every spec under `docs/repo-types/` follows the same outline:

1. **Purpose** — when to use this type (criteria-shaped, not vibes-shaped)
2. **Distribution model** — how the repo ships, how consumers consume it
3. **Stack** — language, frameworks, build tools, key dependencies, version pins
4. **Required directory structure** — file tree expected at `main` HEAD
5. **Required files** — must-haves with role explanation
6. **Conventions** — lint rules, type strictness, naming, version pinning, CI
7. **Anti-patterns** — what NOT to do, with concrete file/line citations to past failures where applicable
8. **Canonical reference** — exemplar repo to follow; "(none yet)" if forward-looking
9. **Naming convention** — repo name format under `juvantlabs/`
10. **Lifecycle** — init → ship → maintain → deprecate / supersede

The uniformity helps both human readers (predictable navigation) and AI
agents (predictable structure to read mechanically).

## Modification

Specs are versioned in this repo. When a spec evolves materially:

1. Open a PR with the change.
2. Reference the trigger in the PR body — what new pattern, audit
   finding, or repeated mistake prompted the update.
3. Merge once review converges.

Trivial fixes (typos, broken links, formatting) are out-of-band and do
not require a trigger note.

## What lives in `juvantlabs/` vs. `juvantio/`

This handbook governs `juvantlabs/*` repos only. The split between
`juvantlabs/` (OSS) and `juvantio/` (commercial) is codified in
[ADR 0001](../adr/0001-account-and-org-structure.md). When in doubt:

- Will external adopters of Juvant OS find this useful? → `juvantlabs/`
- Is this specific to a commercial product or a per-company instance? → `juvantio/`
- Genuinely both? Consider splitting the repo or extracting the generic
  parts into a `juvantlabs/` repo with a private `juvantio/` consumer
  on top.
