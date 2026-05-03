# Repo type — Toolbox

Spec for repos that ship a collection of utility scripts, dev tools,
automation helpers, or CLI commands. Lower-formality than libraries and
MCP servers, but with enough structure to be discoverable and maintainable.

## Purpose

Use this type when:

- The artifact is a **collection of utilities** rather than a single
  cohesive API, AND
- Consumers run the tools directly (CLI, scripts, one-off automations)
  rather than importing them, AND
- The tools are **generic enough to be useful across products** (or to
  external Juvant OS adopters).

Do NOT use this type for:

- Single-cohesive-API code → use [library](library.md).
- Vendor MCP wrappers → use [mcp-server](mcp-server.md).
- Framework scaffolds → use [framework](framework.md).
- Product-specific scripts coupled to one product's workflow → use a
  private repo under `juvantio/<product>-dev-tools` instead. Toolbox in
  `juvantlabs/` means **OSS-shareable, generic across products**.

## Distribution model

Two distinct distribution patterns, depending on how a tool is invoked:

### `git clone` + run

Most toolboxes ship as `git clone` + run-the-script. Consumers clone
the repo and run scripts directly:

```bash
git clone https://github.com/juvantlabs/<name>-tools.git
cd <name>-tools
./scripts/<tool-name>.py --help
```

Adopters typically clone once and update with `git pull` periodically.

### `git clone` + language-registry install

When a tool is mature enough to warrant packaging (a CLI used across
many machines, with versioned semver), it can additionally publish to
PyPI / npm:

```bash
pip install juvant-tools
juvant-tool <subcommand>
```

The published package is a thin wrapper over the canonical script. The
repo retains the `git clone` flow for tinkering / extension.

A toolbox may start with `git clone` only and add registry distribution
when it earns it (≥ 3 distinct users / production-critical / weekly use).

## Stack

Toolboxes are **language-flexible** by design — different tools may use
different languages where each language fits the task. Common patterns:

| Tool category | Language | Notes |
|---|---|---|
| Network protocol introspection | Python | rich ecosystem (websockets, requests, etc.) |
| Build / CI helpers | Bash, Makefile, just | minimal dependencies, run anywhere |
| AWS / Azure / GCP automation | Python (boto3 / azure-sdk / google-cloud-*) | vendor SDKs are best in Python today |
| Cross-platform CLIs with rich UX | Python (click / typer) or TS (commander / yargs) | language depends on team familiarity |
| Quick-and-dirty scripts | Bash | pragmatic, no setup |
| Long-running monitors / daemons | Python or TS | depends on integration surface |

A single toolbox repo MAY contain tools in multiple languages (unlike
[libraries](library.md), where one primary language is the rule).
Document the language per directory if mixed.

When a toolbox has a clear primary language, its CI / lint / formatting
follows that language's canonical conventions (see
[library.md](library.md) for per-language details).

## Required directory structure

```
juvantlabs/<name>-tools/
├── README.md
├── LICENSE                  ← MIT (public OSS-shareable)
├── .gitignore
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       └── ci.yml           ← lint + smoke test (lighter than library/MCP)
├── <tool-category>/         ← e.g. observability/, build/, cloud/
│   ├── README.md            ← per-category README explaining the tools
│   ├── <tool-name>.py       ← (or .ts, .sh, etc.)
│   └── ...
└── docs/
    └── ...                  ← (optional) deeper docs per tool
```

Per-category subdirectories are encouraged when the toolbox grows
beyond ~5 tools. Each subdirectory has its own README explaining what
the tools in that category do, with copy-paste invocation examples.

For toolboxes with a single primary language and registry distribution
(e.g. `pip install juvant-tools` exposing a `juvant-tools` CLI),
add the language manifest at the root:

```
├── pyproject.toml          ← Python with PyPI distribution
└── <package-name>/         ← importable package providing the CLI entrypoints
    └── cli.py
```

(See [library.md](library.md) for the per-language file conventions.)

## Required files

| File | Purpose | Notes |
|---|---|---|
| `README.md` | What the toolbox is for, list of tools with one-line description, install / clone instructions, contribution pointer | Must list every tool; orphan files in repo root are anti-pattern (see below) |
| `LICENSE` | MIT for `juvantlabs/*` toolboxes. Copyright holder: **Juvant Srls** (legal entity); see [`docs/naming.md` § Copyright + LICENSE](../naming.md#copyright--license) | Required; never ship a toolbox without LICENSE — historical mistake on `juvantio/juvant-dev-tools` (LICENSE: null) |
| `.gitignore` | Standard ignore patterns + secrets / `.env` | Same ignore baseline as juvant-os |

## Conventions

### Tool documentation

Every tool has at minimum:

1. **A help / usage message** when invoked with `--help` or no args.
2. **A README entry** in the parent README listing the tool's purpose
   in one line.
3. **A header comment** in the source explaining what the tool does and
   any non-obvious assumptions (e.g. "requires Edge running with
   `--remote-debugging-port=9222`").

For tools with > 50 LOC of logic, add a per-tool README in the
subdirectory.

### Coupling boundaries

A toolbox in `juvantlabs/` is **OSS-shareable**. That means:

- No business-confidential strings (counterparty names, internal URLs,
  product-specific feature names).
- No hardcoded credentials or tokens.
- No coupling to a single private product (e.g. tools that only make
  sense when running against Hardys-internal endpoints belong at
  `juvantio/hardys-*`).

When a tool starts as generic and grows product-specific dependencies,
**split the repo**: extract the generic core to `juvantlabs/`, leave
the product-specific extension at `juvantio/`.

### Test surface

Test rigor scales with tool maturity:

- **Quick-and-dirty scripts** — manual smoke test on add. No unit tests
  required.
- **Stable scripts used regularly (≥ weekly)** — basic unit tests on
  pure-logic functions; manual smoke for I/O paths.
- **Production-critical CLIs** — full unit + integration tests, same
  rigor as a [library](library.md).

CI runs whatever tests exist and fails on regression. Coverage
thresholds are not enforced (toolboxes have heterogeneous scripts).

### Versioning

- **No registry distribution** → no semver discipline; `main` is the
  source of truth.
- **Registry distribution** → semver applies. Same lifecycle as
  [library.md](library.md).

### CI requirements

Every PR and push to `main`:

1. Lint (per language; light-touch by default — toolboxes have
   heterogeneous quality bars, and demanding library-level lint can
   block legitimate quick scripts).
2. Smoke test where smoke tests exist (run any tool with `--help` and
   assert exit 0).
3. Tracked-secret detector (same as juvant-os).

Optional but encouraged:

- Type check (Python: `mypy`; TS: `tsc`) on the more mature tools.
- `pip-audit` / `npm audit` when registry-published.

## Anti-patterns

1. **Product-specific coupling in a `juvantlabs/` toolbox.** The
   archetypal failure mode: `juvantio/juvant-dev-tools` (renamed to
   `juvantio/hardys-dev-tools` on 2026-05-03) named itself "Juvant"
   but every subdirectory was Hardys-specific (Blackboard Collaborate
   debugging, Hardys connector workflows). The result: a repo nobody
   else can use. If a tool is product-specific, it belongs at
   `juvantio/<product>-dev-tools`, not `juvantlabs/`.
2. **Missing LICENSE.** A toolbox without LICENSE is not OSS — by
   default copyright restricts use entirely. Always ship `LICENSE`
   (MIT for `juvantlabs/`).
3. **Orphan files at repo root.** Every script must be findable from
   the README. Files that exist but aren't documented are dead code
   masquerading as production assets.
4. **Hardcoded tenant / org / user identifiers.** A script that hits
   `https://my-company.example.com/api` is not OSS-shareable. Take URLs
   / IDs from `--flag` or `--env-var`; document the substitution.
5. **Silent assumptions about environment.** Tools that fail
   inscrutably when missing a tool (`ffmpeg`, `gh`, `jq`, etc.) waste
   adopter time. Check prerequisites at startup; fail with a clear
   error message and remediation hint.
6. **One mega-README.md** that buries individual tool documentation in
   a 2 000-line wall of text. Prefer a top-level README that lists
   tools with one-line descriptions + per-category subdirectory
   READMEs for the deeper details.
7. **Duplicating logic across tools.** When two scripts share parsing /
   formatting / I/O patterns, extract a shared module. This is the
   migration signal from "toolbox" to "library + CLI" — at that point
   consider whether the repo wants to be reframed as a [library](library.md)
   with CLI entry points instead.

## Canonical reference

**[`juvantlabs/juvant-tools`](https://github.com/juvantlabs/juvant-tools)**
— created 2026-05-03 as the canonical home for OSS-shareable Juvant
utility scripts. First tool shipped 2026-05-03:
[`scaffold mcp-server`](https://github.com/juvantlabs/juvant-tools/blob/main/juvant_tools/scaffolders/mcp_server/README.md)
generates a new `juvantlabs/<vendor>-mcp-server` repo skeleton from the
[`mcp-server.md`](mcp-server.md) spec.

Stack: Python ≥ 3.10, click for CLI, jinja2 for templating, pytest for
tests, hatchling build, pyproject.toml. Distributed via `git clone` +
editable install (`pip install -e .`); PyPI name `juvant-tools` is
reserved but unpublished until the toolbox earns it (per
[Promote to registry](#promote-to-registry)).

Demonstrates the toolbox spec working end-to-end:

- Required files (README, LICENSE, .gitignore, CODEOWNERS, CI workflow,
  SECURITY.md, CONTRIBUTING.md) all present.
- Pattern conventions followed: language-flexible (Python here),
  CLI-first, light formality but complete documentation, test smoke +
  unit coverage on the scaffolder logic itself.
- Anti-patterns avoided: no business-confidential strings, no hardcoded
  credentials, no product-coupling.

### Counter-example — what NOT to do

[`juvantio/hardys-dev-tools`](https://github.com/juvantio/hardys-dev-tools)
(renamed from `juvantio/juvant-dev-tools` on 2026-05-03) is the
canonical study of what NOT to do: a toolbox repo that started
OSS-named ("juvant-dev-tools") but accumulated entirely product-specific
content (Blackboard Collaborate connector debugging, Hardys-prefixed
prompts, product-specific Docker/ACA scripts). The slow drift from
generic to product-specific without a structural reckoning produced a
repo that nobody outside Hardys could use, despite the OSS-leaning
name. The split-when-generic-becomes-specific rule documented above is
the codified lesson; the rename to `hardys-dev-tools` aligns the name
with the actual scope.

## Naming convention

`juvantlabs/<name>-tools` (public) or `juvantio/<product>-dev-tools`
(private, product-coupled). Full rules and the public-vs-private split
rationale live in [`docs/naming.md`](../naming.md).

## Lifecycle

1. **Init** — open an issue under `juvantlabs/handbook` describing
   the toolbox's purpose and primary language(s). Light-touch review:
   the goal is to confirm the toolbox category is appropriate (vs.
   library / MCP / framework) and that the proposed name fits.
2. **Scaffold** — create the repo with README, LICENSE, .gitignore,
   .github/workflows/ci.yml. Initial commit can include the first tool
   directly (no need for empty stubs).
3. **Add tools** — each new tool ships with: README entry, header
   comment, `--help` message, smoke-testable invocation. Quick PRs are
   acceptable; the bar is "discoverable and runnable", not "library-
   grade documentation".
4. **Promote to registry** — when a tool / family of tools earns it
   (≥ 3 distinct users / production-critical / weekly use), add a
   language manifest (`pyproject.toml` / `package.json`) and publish to
   PyPI / npm. From this point the toolbox follows the [library.md](library.md)
   release discipline for those packaged tools.
5. **Maintain** — quarterly review for product coupling drift. If a
   tool has accumulated product-specific dependencies, fork the
   product-specific extension to `juvantio/<product>-dev-tools` and
   restore the toolbox to its OSS-shareable scope.
6. **Deprecate** — individual tools can be removed when they're no
   longer used; CHANGELOG entry + README pruning suffice. The whole
   toolbox repo is archived only when the entire category becomes
   irrelevant — rare for utility repos.
