# Repo type — MCP server

Spec for repos that wrap a vendor's REST or webhook API as a Model Context
Protocol server consumable by Juvant OS agents (or any MCP-aware client).

> **Related**: [ADR 0002 — MCP abstract roles + binding policy](../adr/0002-mcp-abstract-roles.md)
> codifies how abstract role qualifiers (`bank`, `fattura_elettronica`,
> `m365-graph`) bind to concrete provider MCP servers at company init,
> and the process for introducing new abstract roles. Read alongside
> this spec when shipping a new MCP server that fulfills an abstract
> role.

## Purpose

Use this type when:

- An external vendor exposes data / capabilities through a REST API,
  GraphQL, or webhook stream, AND
- A Juvant OS agent (CFO, CMO, etc.) needs to consume that data through
  the MCP tool-call abstraction, AND
- No existing `juvantlabs/*-mcp-server` already wraps the vendor.

Do NOT use this type for:

- Pure libraries with stable Python / TypeScript APIs that don't need
  MCP framing → use [library](library.md).
- Bundles of CLI scripts that humans run directly → use [toolbox](toolbox.md).
- Reference implementations of an entire framework → use [framework](framework.md).

## Distribution model

- **npm publish** as `@juvantlabs/<vendor>-mcp-server` (scoped under the
  juvantlabs npm namespace).
- **Runnable via `npx`** — adopters bind the server in their per-company
  Juvant OS instance via `.juvant/config.json`:

  ```json
  {
    "<role>": {
      "provider": "<vendor>",
      "mcp_server": "npx @juvantlabs/<vendor>-mcp-server",
      "scope": "read"
    }
  }
  ```

- No long-running daemon. The MCP server is spawned on demand by the
  Claude Code / Juvant OS runtime; it exits when the consumer disconnects.

## Stack

| Concern | Choice | Rationale |
|---|---|---|
| Language | TypeScript | Uniform with existing MCP server family + Microsoft / Anthropic SDK ecosystem |
| Node minimum | ≥ 20 (LTS) | Modern fetch, native AbortController, top-level await |
| MCP SDK | `@modelcontextprotocol/sdk` ≥ 1.25.2 | Patches ReDoS (`GHSA-8r9q-7v3j-jr4g`) and DNS rebinding (`GHSA-w48q-cv73-mx4w`) — earlier versions are vulnerable |
| HTTP client | Vendor official SDK if available, else `axios` ≥ patched | Microsoft Graph → `@microsoft/microsoft-graph-client`. Generic REST → `axios` at the latest patched version (avoid SSRF advisories on `< 1.12.x`). |
| OAuth (when needed) | `@azure/msal-node` (Microsoft) or vendor-official | Use vendor SDK; never roll auth |
| Token storage | `@napi-rs/keyring` | `keytar` is archived / unmaintained; do NOT use it |
| Test framework | `vitest` | Fast, ESM-native, TS-first |
| Lint | ESLint with strict TS rules + `eslint-plugin-no-console-log-stdout` (or equivalent custom rule) | Stdout discipline is non-negotiable for MCP framing |
| Build | `tsc` (no bundler) | MCP servers are short-lived; bundling adds no value |

## Required directory structure

```
juvantlabs/<vendor>-mcp-server/
├── README.md
├── ARCHITECTURE.md          ← design rationale + scope decisions
├── CHANGELOG.md             ← Keep a Changelog format
├── CONTRIBUTING.md
├── SECURITY.md              ← disclosure policy + threat model
├── LICENSE                  ← MIT
├── package.json
├── package-lock.json        ← committed; reproducible installs
├── tsconfig.json
├── eslint.config.mjs        ← (or .eslintrc.cjs)
├── .gitignore
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── ci.yml           ← lint + test + audit + dead-code grep
│       └── publish.yml      ← npm publish on tag
├── src/
│   ├── index.ts             ← MCP server entry + stdio transport
│   ├── auth/                ← OAuth / API-key handling
│   ├── tools/               ← one file per MCP tool
│   ├── client/              ← HTTP / SDK wrapper layer
│   └── types/               ← shared TS types
└── tests/
    ├── unit/                ← tool handlers, validators
    └── integration/         ← against vendor sandbox tenant
```

## Required files

| File | Purpose | Notes |
|---|---|---|
| `README.md` | First-read; install, quickstart, env var list, scope justification | Must list **every env var actually wired into `process.env` reads in `src/`** — a CI rule fails the build if README documents an unused env var |
| `ARCHITECTURE.md` | Design rationale: why this MCP, what's in scope, what's not, threat model summary | Cite referenced spec / handbook docs |
| `CHANGELOG.md` | Keep a Changelog format, semver bumps | Public history; tag-driven releases |
| `CONTRIBUTING.md` | How to file bugs, security disclosures, PRs | Point at `juvantlabs/handbook` for org-level governance |
| `SECURITY.md` | Disclosure channels + SLOs + supported versions; follows the [`SECURITY-template.md`](../security/SECURITY-template.md) baseline | Required; pointer back to [`docs/security/disclosure-process.md`](../security/disclosure-process.md) is mandatory |
| `LICENSE` | MIT (canonical text, do not paraphrase) | |
| `package.json` | Dependencies pinned; `engines: { "node": ">=20" }` mandatory | |
| `tsconfig.json` | `strict: true`, `noImplicitAny: true`, `strictNullChecks: true` | No `any` casts in production code; if temp use is essential, justify in PR |
| `.github/workflows/ci.yml` | Run on every PR + push to main | See "CI requirements" below |

## Conventions

### Auth

- API keys / tokens load from env vars at server start. Never read from
  `.juvant/config.json` directly (that is the consumer's concern).
- OAuth flows use vendor-official client libraries (`msal-node` for
  Microsoft, etc.). Never re-implement OAuth from scratch.
- Token persistence (refresh tokens) uses `@napi-rs/keyring`. The MCP
  server process loads + refreshes tokens; tokens never enter the
  agent's context window.
- Tenant ID, when used, must be regex-validated before string-
  interpolating into the authority URL. Pattern:
  `^(common|organizations|consumers|[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})$`.

### Stdout discipline

The MCP stdio transport multiplexes JSON-RPC frames over stdout. Any
non-protocol output corrupts the stream.

- `console.error` for diagnostics, `console.log` for protocol output ONLY
  (which the SDK handles internally).
- A lint rule blocks `console.log` in `src/` — non-negotiable. Add as a
  custom ESLint rule or use `eslint-plugin-no-console-log` configured to
  whitelist only the protocol path.
- This is enforced by CI; PRs that introduce `console.log` are rejected.

### Sandboxing (when filesystem ops exist)

Tools that read or write the local filesystem (e.g. download_file,
upload_file) MUST sandbox to a per-tenant root. `path.resolve` + prefix
check + symlink guard. Never accept arbitrary `localPath` from the caller.

### No general-purpose URL forwarder

Tools must be typed, named, schema-validated operations. NEVER expose
a `batch_operations`-style tool that accepts arbitrary `{method, url,
body, headers}` and forwards them with the user's bearer token. That
pattern is a privilege-bypass primitive (see anti-pattern #C5 below).

### Per-tenant subprocess

The MCP server process must be **per-tenant**. No shared global cache,
no module-level state that crosses tenants. Cache state is process-scoped.

### Streaming + size guards

- Downloads use streaming (no whole-file `arraybuffer`).
- Hard max-file-size cap (e.g. 200 MB) enforced server-side.
- Async vendor operations (copy / move on Microsoft Graph) poll until
  completion; never return "initiated successfully" as the final result.

### Scope minimization

Only request the OAuth / API scopes actually needed by the implemented
tools. Document each scope's justification in `ARCHITECTURE.md`. A
broad scope like `Files.ReadWrite.All` requires explicit rationale.

### Tool design

- Each tool is a typed, schema-validated MCP tool with a clear single
  purpose.
- Tool names are short, lowercase, snake-case (e.g. `get_balance`,
  `list_invoices`, not `bank-get-balance` or `getBalance`).
- Tool handlers wire real validators on every untrusted input. Validators
  cannot exist in source as dead code (CI grep enforces this).
- Delete-class operations gate through a spec / approval pattern (cf. the
  `m365-delete-spec` pattern in FEAT-014).

### CI requirements

Every PR and push to `main` runs:

1. Lint (ESLint, strict, no `--max-warnings N` cosmetic ceiling).
2. Type-check (`tsc --noEmit`).
3. Unit tests (vitest) — coverage target ≥ 80% on real code paths.
4. Integration tests against vendor sandbox (skippable if sandbox
   unavailable; never against live).
5. `npm audit --audit-level=moderate` — fail on moderate or higher.
6. **Dead-code check**: grep the source for exported validators /
   security helpers that are NOT imported in any production handler.
   Any unused security export fails the build.
7. **Stdout discipline check**: grep for `console.log\(` in `src/` —
   any hit fails the build.
8. **README accuracy check**: every env var documented in README must
   be referenced in `src/` via `process.env.X`. Mismatches fail the build.

Publish runs on tag push (`v*`) — separate workflow, requires manual
approval gate.

## Anti-patterns

These are documented as the **anti-pattern checklist** because they were
all observed in
[`ftaricano/mcp-onedrive-sharepoint`](https://github.com/ftaricano/mcp-onedrive-sharepoint)
during the 2026-05-03 security audit ([gist](https://gist.github.com/juvantlabs/a9fe0a76a23b0c1260b1e0ad3194a6da)).
Each one corresponds to a finding from that audit.

1. **Arbitrary local-filesystem write/read** through caller-supplied paths
   (audit findings C1, C2, C3, C4). Mitigation: sandbox to per-tenant
   root, validate every `localPath` / `outputPath`.
2. **General-purpose URL forwarder primitive** (`batch_operations` style;
   audit C5). Privilege-bypass class. Never expose; if "batch" is a
   genuine vendor capability, model it as a typed, schema-validated tool
   with a hardcoded URL allowlist.
3. **`console.log` on stdout in MCP-stdio servers** (audit C6). Corrupts
   the JSON-RPC framing. Always `console.error`.
4. **Outdated MCP SDK** (audit C7). MCP SDK < 1.25.2 has known ReDoS +
   DNS-rebinding advisories. Pin ≥ 1.25.2 from day one.
5. **Vulnerable axios reachable from tool surface** (audit C8). If using
   axios, pin to the latest patched version and verify `npm audit`
   returns clean.
6. **Vulnerable `jws` transitively** (audit C9). Periodically run `npm
   audit` and bump.
7. **Defense-in-depth code that is unreachable from handlers** (audit
   S1). Validators / sanitizers must be wired into the actual call graph;
   CI grep enforces.
8. **Env vars documented but unwired** (audit S2). README lies about
   configuration knobs that don't actually do anything. CI rule fails the
   build.
9. **OData / URL injection through unencoded user input** (audit S3).
   Use vendor SDK or properly escape every interpolated parameter.
10. **`keytar` for token storage** (audit S5). Archived since 2022. Use
    `@napi-rs/keyring` or OS file with mode `0600`.
11. **Whole-file buffering on download** (audit S7). OOM hazard on
    adversarial input. Stream + max-size cap.
12. **No async-op polling** (audit S8). `copy`/`move` operations that
    return 202 must poll the monitor URL until completion; never tell the
    caller "initiated successfully" as the final result.

## Canonical reference

When the first MCP server ships, its repo URL goes here. Until then:

- (Planned) `juvantlabs/finom-mcp-server` — read-only banking MCP for
  Finom Partner API. FEAT-011, ~1 day implementation effort, blocked on
  obtaining a real Finom API key (sandbox literal documented in OpenAPI
  spec is broken; see [FEAT-011 issue comments](https://github.com/juvantlabs/juvant-os-pm/issues/25)).
- (Planned) `juvantlabs/m365-graph-mcp-server` — read+write Microsoft
  Graph MCP for OneDrive/SharePoint/Calendar. FEAT-014, ~3 days
  implementation, no blocker.
- (Planned) `juvantlabs/aruba-fattura-mcp-server` — read-only Italian
  SDI e-invoicing MCP. FEAT-012, ~2 days.

## Naming convention

`juvantlabs/<vendor>-mcp-server`. Full naming rules (vendor / family
prefixes, suffix mandate, npm scoped package names) live in
[`docs/naming.md`](../naming.md) — refer there for the canonical
patterns. This spec adds no exceptions.

## Lifecycle

1. **Init** — open a FEAT issue under `juvantlabs/juvant-os-pm`. Document
   the vendor scope, abstract role binding (`bank`, `fattura_elettronica`,
   `m365-graph`, etc.), authentication model, and tools planned. Surface
   any prior art (community alternatives) and document audit findings if
   evaluation surfaces blockers.
2. **Scaffold** — create the repo per this spec. Initial commit must
   include all required files (README skeleton, LICENSE, etc.) — empty
   stubs are fine but the structure must be in place.
3. **Implement** — incremental ships preferred (auth → first tool →
   subsequent tools → tests → publish). Each ship is a tag.
4. **Publish** — npm publish on tag. CI runs full audit on every publish.
5. **Maintain** — quarterly `npm audit` + dependency bumps; respond to
   security advisories within 7 days; bump MCP SDK when major versions
   ship and our tests pass.
6. **Deprecate** — when a vendor sunsets the wrapped API, archive the
   repo with a README banner pointing at the replacement (or at the
   vendor's own MCP if one ships). Update `juvantlabs/juvant-os/docs/MCP_INVENTORY.md`
   to note the deprecation.
