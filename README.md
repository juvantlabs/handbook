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
