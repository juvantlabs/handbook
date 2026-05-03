# juvantlabs / handbook

How juvantlabs operates: **governance, principles, processes, and decisions**
that span the OSS framework family (`juvantlabs/juvant-os` and the
`juvantlabs/*-mcp-server` siblings).

## What lives here

- [`docs/adr/`](docs/adr/) — Architecture Decision Records at the **org
  level** (organizational structure, contribution policy, security
  disclosure process, MCP-server-naming convention, etc.). Distinct from
  framework-level architecture, which lives at
  [`juvantlabs/juvant-os/docs/adr/`](https://github.com/juvantlabs/juvant-os/tree/main/docs/adr).
- [`docs/repo-types/`](docs/repo-types/) — **Specifications for the four
  canonical kinds of repository** under `juvantlabs/*`: MCP server,
  library, framework template, toolbox. The source of truth when
  creating a new repo: AI agents and human contributors read the
  matching spec and produce a conforming repo by following it literally.
- [`docs/security/`](docs/security/) — **Security disclosure process**
  for both incoming reports (someone reports a vuln in our code) and
  outgoing audits (we review a community alternative and find vulns).
  Includes templates: `SECURITY.md` for every `juvantlabs/*` repo + audit
  report structure for outgoing public gists.
- [`docs/naming.md`](docs/naming.md) — **Naming conventions** across the
  `juvantlabs` ecosystem: repo names per type, package / registry names,
  branch / tag / commit / issue conventions. Single source of truth that
  the four repo-type specs reference rather than duplicating.
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — **Meta contributor guide** for
  all `juvantlabs/*` repos: where to file what, PR process, commit
  style, AI-assisted contribution conventions, code-of-conduct pointer.
  Per-repo `CONTRIBUTING.md` files follow this template.
- [`docs/contributing/code-of-conduct.md`](docs/contributing/code-of-conduct.md)
  — Code of Conduct (Contributor Covenant 2.1, adapted) governing all
  `juvantlabs/*` interactions.

## What does NOT live here

- **Code.** This is documentation-only. Code lives at
  `juvantlabs/juvant-os` (the OSS template) and the various
  `juvantlabs/*-mcp-server` repos.
- **Framework-level architecture.** ADRs about how Juvant OS itself works
  (Skill-first orchestrator, Bootstrap Protocol, single-writer invariant,
  etc.) live at [`juvantlabs/juvant-os/docs/adr/`](https://github.com/juvantlabs/juvant-os/tree/main/docs/adr).
- **Commercial / per-company business operations.** Those are private at
  `juvantio/*` (Juvant Srls's own org) and at adopters' own orgs.

## Distinction at a glance

| Question | Where to look |
|---|---|
| How do I run my own Juvant OS instance? | [`juvantlabs/juvant-os`](https://github.com/juvantlabs/juvant-os) |
| Why does the framework work the way it does? (architecture) | [`juvantlabs/juvant-os/docs/adr/`](https://github.com/juvantlabs/juvant-os/tree/main/docs/adr) |
| How does juvantlabs (the OSS arm) govern itself? Org structure, contribution policy, security disclosure process? | **here**, `docs/adr/` |
| What's planned next? Open features, OPs? | [`juvantlabs/juvant-os-pm`](https://github.com/juvantlabs/juvant-os-pm) |
| What's a per-company instance look like? | (private; canonical reference: `juvantio/juvant`) |

## License

- **Documentation** (`docs/adr/`, `README.md`): CC0 1.0 Universal (public
  domain). Re-use freely, with or without attribution.
- **Code samples** (if any are added in the future): MIT.

See [`LICENSE`](LICENSE) for the canonical text.
