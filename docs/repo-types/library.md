# Repo type — Library

Spec for repos that ship importable code with a stable API. No MCP
wrapper, no runtime side-effects required, no agent-specific framing.
Pure library / SDK / utility module.

## Purpose

Use this type when:

- The artifact is **importable code with a stable, documented API**, AND
- Consumers integrate it into their own application (not via MCP, not via
  CLI), AND
- The code is reusable across projects (not coupled to one product).

Do NOT use this type for:

- Vendor API wrappers exposed to agents through MCP → use [mcp-server](mcp-server.md).
- Reference implementations of an entire framework that adopters
  mirror-push and customize → use [framework](framework.md).
- Collections of utility scripts and dev tools → use [toolbox](toolbox.md).

## Distribution model

- **Per-language registry** — PyPI for Python, npm for TypeScript, crates.io
  for Rust, etc. Each library targets one primary language; multi-language
  bindings are out-of-scope at this maturity (open a follow-up handbook
  discussion if needed).
- **Importable, no daemon, no separate process.** Consumers integrate as
  any other library: `pip install`, `npm install`, etc.
- **Semver discipline.** Every release is tagged; breaking changes bump
  major versions; consumers can pin.

## Stack

| Concern | Choice | Notes |
|---|---|---|
| Language | One primary language per library | Don't mix Python + TypeScript in the same repo. If both are needed, ship two separate libraries with bindings between them. |
| Build | Language-canonical | Python: `pyproject.toml` (PEP 517/518) with hatch / poetry / setuptools; TS: `package.json` + `tsc` |
| Test framework | Language-canonical | Python: `pytest`. TS: `vitest`. |
| Lint / formatter | Language-canonical | Python: `ruff` (fast, modern; replaces black + isort + flake8). TS: ESLint with strict rules. |
| Type checker | Language-canonical | Python: `mypy --strict`. TS: `tsc --strict` (no `any` in production). |
| Dependency hygiene | Lockfile committed | Python: `uv.lock` or `poetry.lock`. TS: `package-lock.json` or `pnpm-lock.yaml`. |

## Required directory structure

Python (Engram-style canonical):

```
juvantlabs/<library-name>/
├── README.md
├── ARCHITECTURE.md
├── CHANGELOG.md
├── CONTRIBUTING.md
├── SECURITY.md
├── ROADMAP.md            ← (optional but recommended for libraries with
│                            multi-quarter trajectories)
├── LICENSE                ← MIT
├── pyproject.toml
├── uv.lock                ← (or poetry.lock) — committed
├── .gitignore
├── .dockerignore          ← (when Docker images are part of the distribution)
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── ci.yml
│       └── publish.yml
├── <package-name>/        ← source (Python: importable package; same name
│   ├── __init__.py            as the library by convention)
│   └── ...
└── tests/
    ├── unit/
    └── integration/
```

TypeScript:

```
juvantlabs/<library-name>/
├── README.md
├── ARCHITECTURE.md
├── CHANGELOG.md
├── CONTRIBUTING.md
├── SECURITY.md
├── LICENSE                ← MIT
├── package.json
├── package-lock.json      ← committed
├── tsconfig.json
├── eslint.config.mjs
├── .gitignore
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── ci.yml
│       └── publish.yml
├── src/
│   ├── index.ts            ← public surface (re-exports)
│   └── ...
└── tests/
```

## Required files

| File | Purpose |
|---|---|
| `README.md` | First-read; install, quickstart, full API surface example, link to ARCHITECTURE for design rationale |
| `ARCHITECTURE.md` | Design rationale, key invariants, performance characteristics, "why this design over the alternatives" |
| `CHANGELOG.md` | Keep a Changelog format, semver-aligned |
| `CONTRIBUTING.md` | Issue / PR / dev-loop instructions |
| `SECURITY.md` | Disclosure channels + SLOs + supported versions; follows the [`SECURITY-template.md`](../security/SECURITY-template.md) baseline. Pointer back to [`docs/security/disclosure-process.md`](../security/disclosure-process.md) is mandatory |
| `LICENSE` | MIT (or compatible if upstream forces; non-MIT is exceptional and requires CA + CSO joint approval) |
| `ROADMAP.md` | Optional — recommended for libraries that ship across multiple quarters with a coherent trajectory |
| Language manifest | `pyproject.toml` (Python), `package.json` (TS), etc. |
| Lockfile | Committed for reproducible installs |

`pyproject.toml` for Python libraries documents:

- Dependencies (with version pins)
- Optional dependency groups (e.g. `[redis]`, `[dev]`)
- Build system (PEP 517/518)
- Project metadata (authors, license, URLs, classifiers)
- Tool configurations (`[tool.ruff]`, `[tool.mypy]`, `[tool.pytest]`)

## Conventions

### Public API surface

- Every public symbol exported from the package's top level must appear
  in `__init__.py` (Python) / `src/index.ts` (TS) re-exports. Internal
  symbols stay in submodules and are NOT importable from the top level.
- Public API is **documented in the README** with at least one runnable
  example per primary use case.
- Type hints / TypeScript types are mandatory. No `Any` (Python) or `any`
  (TS) on public surfaces.
- API stability: bump major version on breaking changes. Internal
  refactors (private surfaces, performance) are minor or patch.

### Tests

- Unit tests on every public function / method. Coverage target ≥ 80%.
- Integration tests where the library interacts with external systems
  (network, filesystem, subprocess). Mock external calls in unit tests;
  exercise real systems in integration tests.
- CI runs both layers on every PR.

### Documentation

- README is the entry point — must include install + quickstart + key
  primitives.
- ARCHITECTURE.md is the second-read — explains "why this shape, why this
  caching strategy, why these invariants" for contributors and future
  maintainers.
- Per-method docstrings (Python) or JSDoc (TS) for every public symbol.
- Optional but recommended: rendered docs site (mkdocs / typedoc) for
  libraries with > 5 public surfaces.

### Performance

- Document performance characteristics in ARCHITECTURE.md when they
  matter (e.g. "first call: ~1s LLM round trip; cached: < 1ms").
- Benchmark harness in `benchmarks/` for libraries where performance is
  a stated property of the design.

### Backwards compatibility

- Deprecate before removing. A deprecation warning is added in the minor
  version that introduces the replacement; removal happens in the next
  major.
- Deprecation messages cite the issue / PR that introduced the
  deprecation, the replacement, and the planned removal version.

### CI requirements

Every PR and push to `main`:

1. Lint (`ruff` for Python, ESLint for TS).
2. Format check (`ruff format --check` for Python; `prettier --check` for TS).
3. Type check (`mypy --strict` for Python; `tsc --noEmit` for TS).
4. Unit + integration tests.
5. Coverage report (target ≥ 80%; fail if below for ≥ 2 consecutive PRs).
6. `pip-audit` (Python) or `npm audit` (TS) — fail on moderate or higher
   advisory severity.
7. Tracked-secret detector (same as juvantlabs / juvant-os ships in
   `.github/workflows/lint.yml`).

Publish runs on tag push (`v*`). PyPI / npm credentials live in GitHub
Actions secrets.

## Anti-patterns

1. **Coupling to a single consumer.** A library that only the canonical
   adopter can use is not a library — it's a private dependency. If the
   only realistic consumer is a single product, demote to a `juvantio/`
   private repo (or vendor the code into that product directly).
2. **Optional features as required.** `pip install <library>[redis]` is
   correct for "Redis-backed cache is optional"; making redis a
   top-level dependency is wrong.
3. **Catch-all exception handling.** Library code that swallows
   `Exception:` (or `catch (any)`) and returns sentinel values mocks
   contract clarity. Either propagate the error or define a typed
   exception class with documented semantics.
4. **Mutable global state.** Module-level singletons that consumers can
   accidentally mutate make multi-threaded / multi-tenant use unsafe.
   Use explicit instances (constructor-style) for any non-trivial state.
5. **Surface that conflates "easy" with "powerful".** A library that
   exposes both `simple_call(x)` and `advanced_call(x, y, z)` for the
   same operation must clearly document when to use which. Prefer one
   well-documented surface with optional kwargs over two confusable
   ones.
6. **Undocumented breaking changes.** Every breaking change has a
   CHANGELOG entry under the `### Removed` or `### Changed` heading,
   with the rationale and migration path.
7. **Pinning dependencies too tightly.** Libraries that pin `~=X.Y.Z`
   make consumer dep-resolution painful. Pin to compatible-major
   ranges (`~=X.Y` or `>=X.Y,<X+1.0`) unless a specific patch fixes a
   known bug for our use case.
8. **Hidden network calls.** A library function that names suggests
   pure computation but in fact hits the network is a footgun. Document
   I/O behavior in the API surface name (e.g. `fetch_remote_x` not
   `get_x`) or in the docstring's first line.

## Canonical reference

**[Engram](https://github.com/juvantlabs/engram)** — Python-native
browser automation with cached LLM-resolved selectors. MIT, Python ≥ 3.10,
distributed via PyPI as `engram-browser`.

Engram exemplifies the spec because it ships:

- README (12 KB) with comprehensive quickstart
- ARCHITECTURE.md (14 KB) with full design rationale (caching strategy,
  selector resolution, invalidation pattern)
- ROADMAP.md, SECURITY.md, CONTRIBUTING.md, CHANGELOG.md (the full
  hygiene set)
- `pyproject.toml` with structured deps, optional `[redis]` extra
- Tests under `tests/`
- CI workflow under `.github/workflows/`
- Native Docker support (`.dockerignore`, Dockerfile)

When adding a new library, copy Engram's structure, replace the package
name + content, and you have a conforming repo within minutes.

## Naming convention

`juvantlabs/<library-name>`

- `<library-name>` is short, lowercase, hyphenated, descriptive
  (`engram`, `flux`, `lattice`).
- No suffix (no `-lib`, no `-sdk` — those are noise).
- The Python / npm package name SHOULD match the repo name. When the
  registry name is taken (as Engram experienced with `engram` on PyPI
  → `engram-browser`), document the discrepancy in the README and the
  reason in ARCHITECTURE.md.

## Lifecycle

1. **Init** — open an issue under `juvantlabs/handbook` (or the relevant
   PM tracker) describing the library's scope, primary language, target
   consumers, and trajectory. CA + at least one reviewer approve before
   the repo is created.
2. **Scaffold** — copy Engram's structure (or the TS equivalent once a
   canonical TS library ships). Initial commit must include all required
   files; empty stubs are fine.
3. **Implement** — incremental ships preferred. Each minor adds a
   feature; each patch fixes bugs without API change.
4. **Publish** — registry release on tag push (`v*`). First public
   release is `v0.1.0`; v1.0 indicates committed API stability.
5. **Maintain** — quarterly dependency audit + bumps; deprecate before
   removing; respond to security advisories within 7 days.
6. **Deprecate** — README banner + final release with `### Deprecated`
   notes + repository archive once consumers have migrated. Cite
   replacement library if applicable.
