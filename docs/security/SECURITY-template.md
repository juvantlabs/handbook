# SECURITY.md template

Every `juvantlabs/*` repo ships a `SECURITY.md` at the repo root that
follows this template. Customize the placeholder sections (project
name, supported versions, etc.); keep the disclosure-process pointer
unchanged.

---

```markdown
# Security

## Reporting a vulnerability

Please report vulnerabilities **privately** via one of these channels:

1. **GitHub Security Advisory** (preferred) — go to this repo's
   `Security` tab → `Report a vulnerability`. Your report stays private
   between you and the maintainer until we publish a coordinated
   advisory.
2. **Email** — `security@juvant.io`. PGP encryption is not required
   but welcomed; key fingerprint will be published here when the key
   is generated. Reports go to the primary maintainer.

**Please do NOT** open a public issue or pull request that contains
reproduction details for the vulnerability. Once a public artifact
exposes the issue, the coordinated-disclosure window collapses.

## What we commit to

This repo follows the
[juvantlabs Security Disclosure Process](https://github.com/juvantlabs/handbook/blob/main/docs/security/disclosure-process.md).
Summary of the SLOs you can expect:

| State | Target |
|---|---|
| Acknowledge receipt | ≤ 7 days |
| Initial triage + severity classification | ≤ 14 days |
| Patch prepared (high/critical) | ≤ 30 days |
| Patch prepared (moderate) | ≤ 90 days |
| Public advisory + CVE | Patch + 1–7 days |

## Supported versions

| Version | Supported? |
|---|---|
| `<latest>` (current major) | ✅ |
| `<previous major>` | Security fixes only, until [date or release] |
| `< <previous major>` | ❌ End of life |

## Out of scope

- Issues in dependencies — please report those upstream. We track
  upstream advisories via Dependabot and bump promptly.
- Issues in adopter customizations of this repo (when this is a
  framework template; per-adopter mirrors are out of our scope).
- Theoretical vulnerabilities without a reproduction path.

## Crediting

Reporters are credited by name in advisories unless they request
anonymity at report time. We are happy to coordinate timeline if a
reporter has scheduling constraints (conference talk, employer
disclosure policy, etc.).

## Acknowledgments

Past reports and the people who made them are listed in
[`SECURITY-CREDITS.md`](SECURITY-CREDITS.md) (created on first
disclosure).
```

---

## When to deviate

This template is a baseline. Repos may add sections without removing
the required ones (Reporting / What we commit to / Supported versions /
Out of scope / Crediting). Useful additions for some repos:

- **Threat model** — short narrative of what attacker capabilities the
  repo defends against (relevant for MCP servers, framework templates).
- **Security-relevant dependencies** — list of dependencies whose vulns
  would be especially impactful (e.g. `@modelcontextprotocol/sdk` for
  MCP servers, `axios` if used, `msal-node` for OAuth flows).
- **Audit history** — links to past CSO Layer 5 audits or external
  reviews when they exist.

Removing a required section is **not** acceptable; the disclosure
pipeline depends on consistent expectations across all `juvantlabs/*`
repos.
