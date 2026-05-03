# Repo type — Framework template

Spec for repos that ship a complete operating-system scaffold which
adopters mirror-push to their own GitHub org and customize for their
company. Distinct from libraries (importable code) and MCP servers
(runtime components): a framework template is the **whole house**, ready
to be moved into.

## Purpose

Use this type when:

- The artifact is a **scaffolded reference implementation** of an
  operating-pattern intended to be cloned, customized per-adopter, and
  operated independently, AND
- It contains **multiple components** (agent templates, hooks, schema,
  scripts, plugins) bound together by a contracted runtime, AND
- Adopters are **expected to customize** the contents (not just consume
  it).

Do NOT use this type for:

- Reusable code that consumers `import` → use [library](library.md).
- Single-vendor API wrappers consumed via MCP → use [mcp-server](mcp-server.md).
- Utility scripts that humans run directly → use [toolbox](toolbox.md).

## Distribution model

`git push --mirror`. Adopters clone the framework as a bare repository
and mirror-push to a private repository in their own GitHub
organization, where they customize. The framework template repository
itself is **never directly cloned for production use**.

```bash
# 1. Bare clone of the OSS framework
git clone --bare git@github.com:juvantlabs/<framework-name>.git

# 2. Mirror-push to per-company instance
cd <framework-name>.git
git push --mirror git@github.com:<adopter-org>/<adopter-instance>.git

# 3. Cleanup
cd .. && rm -rf <framework-name>.git

# 4. Working clone
git clone git@github.com:<adopter-org>/<adopter-instance>.git
```

The mirror-push (rather than GitHub Fork) is intentional: it decouples
the per-company repo's visibility from the framework's. The framework
may eventually go fully public; per-company instances stay private
regardless.

After mirror-push, the per-company repo carries an `upstream` remote
pointing back at the framework — for future structural updates pulled
through the framework's own upstream-sync flow (when defined; cf.
juvant-os Section "Upstream sync").

## Stack

A framework template is **multi-component** by definition. There is no
single primary language; the framework is hybrid by design:

- **Markdown** for skill orchestration, agent templates, ADRs, governance.
- **Bash** for lifecycle hooks (or cross-platform shell where adopters
  run on non-Unix platforms).
- **SQL** for schema definitions.
- **TypeScript** for plugins (when Channel-plugin pattern is used).
- **JSON** for runtime configuration (`.claude/settings.json`,
  `.juvant/config.json`).

The framework chooses each component's language pragmatically — there
is no rule "everything must be TypeScript". The goal is that each
component is in the language best suited to its role, with consistent
conventions within each language.

## Required directory structure

```
juvantlabs/<framework-name>/
├── README.md                ← what this is, who should mirror-push it
├── <ORCHESTRATOR>.md        ← Skill / orchestrator (e.g. JUVANT_OS.md)
├── <INVARIANTS>.md          ← cross-cutting invariants (e.g. SYSTEM_INVARIANTS.md)
├── CHANGELOG.md
├── LICENSE                   ← MIT
├── .gitignore
├── .claude/
│   └── settings.json         ← Claude Code config (hooks, channels, MCP)
├── agents/                   ← subagent templates (.md with placeholders)
│   ├── company/              ← company-scope agents
│   └── projects/             ← project-scope agents
├── hooks/                    ← lifecycle bash scripts
├── scripts/
│   ├── schema.sql            ← runtime schema
│   ├── migrate.sh            ← schema applier
│   └── ...
├── plugins/                  ← Channel plugins (Markdown README + TS source)
├── docs/
│   ├── adr/                  ← framework-level architecture decisions
│   ├── MCP_INVENTORY.md      ← (when applicable) canonical MCP list
│   └── ...
└── .github/
    ├── CODEOWNERS
    └── workflows/
        └── lint.yml          ← framework-level CI
```

The framework's own ADRs (under `docs/adr/`) cover **framework
architecture** — how the framework itself works. Org-level ADRs (about
how `juvantlabs` operates as an entity) live separately at
[`juvantlabs/handbook/docs/adr/`](https://github.com/juvantlabs/handbook/tree/main/docs/adr)
per [ADR 0001](../adr/0001-account-and-org-structure.md).

## Required files

| File | Purpose |
|---|---|
| `README.md` | First-read for any adopter; what this framework is, the mirror-push procedure, where the orchestrator lives |
| `<ORCHESTRATOR>.md` | The Skill / Skill-orchestrator that the runtime loads (e.g. `JUVANT_OS.md` for Juvant OS) |
| `<INVARIANTS>.md` | Cross-cutting invariants that all components defer to (e.g. `SYSTEM_INVARIANTS.md`) |
| `CHANGELOG.md` | Versioned framework releases |
| `LICENSE` | MIT |
| `docs/adr/` | At least an index README + the framework's own architectural decisions |
| `.github/workflows/lint.yml` | CI for the framework template itself (markdown lint, YAML frontmatter check, schema syntax check, secret detector) |
| `SECURITY.md` | Disclosure channels + SLOs + supported versions; follows the [`SECURITY-template.md`](../security/SECURITY-template.md) baseline. Pointer back to [`docs/security/disclosure-process.md`](../security/disclosure-process.md) is mandatory |

## Conventions

### Placeholder syntax

Templates that get customized per-adopter use the `{{PLACEHOLDER}}`
convention. The orchestrator's setup procedure substitutes them at
adopter-init.

- **Whole-token substitution only.** No partial matches. `{{COMPANY_NAME}}`
  is replaced; `{{COMPANY_NAME_OWNER}}` is not.
- **Substitution failure is a hard error.** Any surviving non-allowlisted
  `{{...}}` token in a committed adopter file is a setup failure;
  the wizard refuses to write.
- **Allowlist** — runtime-bound placeholders that should survive
  compilation are explicitly enumerated in the framework's invariants
  doc (cf. SYSTEM_INVARIANTS.md §2 in juvant-os).

### Wizard-driven adoption

The framework is initialized through an **interactive wizard** invoked
from the orchestrator. The wizard:

1. Pre-flight checks the environment (e.g. correct repo, no prior init
   state).
2. Collects identity + configuration interactively (one question at a
   time, with discovery-via-tool preferred over type-it where possible).
3. Compiles templates with substitution.
4. Bootstraps any required external state (databases, OAuth, etc.).
5. Records the final state in a configuration file (gitignored secrets,
   committed structure).
6. Pushes the initial commit.

The wizard procedure is **the contract** between the framework and
adopters. Changes to the wizard semantics are framework-architecture
decisions and warrant ADRs.

### Multi-component coherence

The components reference each other by abstract role / contract:

- Agent templates reference invariants (e.g. "applies SYSTEM_INVARIANTS.md
  §3 disclosure cascade").
- Hooks consume schema (Turso tables defined in `scripts/schema.sql`).
- Plugins integrate via runtime config (`.claude/settings.json`).
- Skill orchestrator references everything.

Cross-references use **stable doc paths** — never relative line
numbers, never URLs that depend on commit hashes.

### Generic content only

Per [`feedback_oss_genericity.md`](../adr/0001-account-and-org-structure.md)
and the OSS-genericity rule:

- The framework template contains **no Juvant-Srls-specific identifiers**.
  No "Antonio", no "Juvant Srls", no "juvant.io", no `juvantio/juvant`.
- Examples in wizards use generic placeholders (`Acme Corp`, `Jane Doe`,
  `<your-org>/<company-slug>`).
- Adopter customization happens at adopter-init; the template is
  pristine.

### Per-component test strategy

Each component type has its own test approach:

- **Agent templates** — YAML frontmatter validation; invariants references
  resolve.
- **Schema** — `sqlite3 :memory: < schema.sql` parses cleanly.
- **Hooks** — manual test (run each hook with synthetic stdin, verify
  expected state changes).
- **Plugins** — TS unit + integration tests per plugin.
- **Wizard** — adopter-init test (full end-to-end on a fresh fork) — this
  is "dogfooding" and is the most important test.

### CI requirements

The framework's own CI workflow (`.github/workflows/lint.yml`) runs:

1. Markdown lint (relaxed; adopters tighten via fork).
2. YAML frontmatter parse on every agent template.
3. Schema syntax check (`sqlite3 :memory:` dry-run).
4. JSON validation of `.claude/settings.json`.
5. Tracked-secret detector.

(Adopters add their own CI on top of this in their per-company instance,
e.g. surviving-placeholder detector — that check is meaningful only
post-init when placeholders are expected to be substituted.)

## Anti-patterns

1. **Adopter-specific content in the template.** Any string that names a
   specific real company, person, domain, or instance breaks
   re-distributability. Caught by the OSS-genericity rule (see audit:
   the v0.4.x dogfood discovered "Antonio" + "Juvant Srls" + "juvant.io"
   leaks that had to be stripped).
2. **Wizard that asks the user to type information available via tool.**
   When a connector / API can discover the answer, prefer
   discover-via-tool over type-it. Surfaced during the v0.4.0 dogfood
   when the wizard tried to ask for OneDrive folder paths the M365
   connector could enumerate.
3. **Surviving placeholders in committed adopter files.** A `{{NAME}}`
   that escapes substitution becomes a runtime question mark. Always
   refuse to write on substitution failure (with an explicit allowlist
   for runtime-bound placeholders).
4. **Cross-component references via line numbers / commit hashes.**
   Stable section anchors and file paths only; the framework should
   tolerate internal refactors without breaking adopter references.
5. **Components that drift from the orchestrator.** When the orchestrator
   spec evolves, every component must catch up. Drift is an audit finding
   in the framework's own CSO Layer 5 audit (cf. juvant-os v0.4.0 dogfood).
6. **Adopter customizations baked into the template.** The template ships
   pristine; adopter-specific values appear only after wizard completion
   in `.juvant/config.json` (gitignored where secret).
7. **Mixing org-level governance with framework-level architecture.** ADRs
   under the framework's `docs/adr/` cover framework architecture only;
   org governance lives at `juvantlabs/handbook/docs/adr/`. Confusing the
   two led to the ADR-0009 revert on 2026-05-03.
8. **Skipping the wizard for "advanced users".** The wizard is the
   contract. If an advanced-user path is needed, it goes **through** the
   wizard with a fast-path flag — not around it. Bypassing the wizard
   short-circuits invariants enforcement.

## Canonical reference

**[juvant-os](https://github.com/juvantlabs/juvant-os)** — multi-agent
operating system that runs a company with Claude Code. Currently `n=1`
canonical framework template under `juvantlabs/`. Future verticals
(if any) — e.g. `juvant-edu-os` or `juvant-health-os` — would follow
this spec.

juvant-os exemplifies the spec by:

- Skill orchestrator at `JUVANT_OS.md` (the wizard-driver)
- Cross-cutting invariants at `SYSTEM_INVARIANTS.md` (§1 Bootstrap, §2
  Naming, §3 Disclosure Cascade, §4 Single-Writer, §5 Universal
  CONFIDENTIAL, §6 Spec Authorization, §7 Architectural Principles)
- Agent templates under `agents/company/` (10) + `agents/projects/` (9)
- Hooks under `hooks/` (7 scripts: SessionStart, SessionEnd, PreCompact,
  PostCompact, Notification, SubagentStart, SubagentStop)
- Schema at `scripts/schema.sql` (20 Turso tables)
- Architecture decisions at `docs/adr/` (ADRs 0001–0008)
- Generic-by-construction (no Juvant-Srls-specific content)
- Dogfood-validated (first real `Initialize Juvant OS` on 2026-05-02
  in 17:50 against `juvantio/juvant`)

## Naming convention

`juvantlabs/<framework-name>-os` — when the framework is an "operating
system" pattern (the canonical case). Other framework patterns may use
different suffixes (e.g. `<framework-name>-platform`, `<framework-name>-runtime`)
when they emerge. For now `juvant-os` is the only example, and the
`-os` suffix is appropriate.

The `<framework-name>` prefix should match the meaningful identity of
the framework — `juvant` for the canonical Juvant OS; future verticals
(`edu`, `health`, `legal`) would prefix accordingly.

## Lifecycle

1. **Init** — open an issue under `juvantlabs/<framework-name>-pm` (the
   framework's own PM tracker) capturing the framework's vision, scope,
   and invariants candidates. CA + CSO + CEthO + CHRO joint review of
   the initial spec before the repo is created (cross-cutting
   invariants are load-bearing).
2. **Scaffold** — create the repo per this spec. Initial commit must
   include the orchestrator stub, invariants stub, agent template
   skeleton, schema scaffold, hooks scaffold, README, LICENSE,
   CHANGELOG (empty `[Unreleased]`).
3. **Build out phases** — the framework grows in numbered phases, each a
   shipped tag. juvant-os example: Phase 1 (skeleton) → Phase 2 (schema)
   → Phase 3 (hooks) → Phase 4 (Skill orchestrator) → Phase 5 (agent
   templates) → Phase 6+ (plugins, scheduled tasks, etc.).
4. **Dogfood** — first real adopter-init run validates the wizard end-to-
   end. Bugs surfaced during dogfood become bug fixes in the next patch
   release.
5. **Publish** — tag-driven releases. `v0.1.0` first scaffold; `v1.0.0`
   indicates the framework is production-ready (wizard works, all
   advertised components ship, dogfood-validated).
6. **Maintain** — adopters pull structural updates via the framework's
   upstream-sync flow (cf. juvant-os Section "Upstream sync"). New ADRs
   document material decisions; the framework's own
   `docs/adr/<NNNN>-*.md` is the audit trail.
7. **Deprecate / supersede** — if a framework is replaced (e.g. by a
   v2 with different invariants), document via a new ADR + clear
   migration guidance. Adopters mirror-push the new framework into a
   parallel repo and migrate state per the published procedure.
