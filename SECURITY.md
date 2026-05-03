# Security

## Reporting a vulnerability in this handbook repo

This repository contains documentation only — no executable code. The
realistic vulnerability surface is small, but if you spot an issue
(e.g. a process documented here that, if followed literally, would
expose someone to harm), please report it privately via:

1. **GitHub Security Advisory** — `Security` tab → `Report a vulnerability`.
2. **Email** — `security@juvant.io`.

Do **not** open a public issue with reproduction details before
coordinated disclosure.

## Reporting a vulnerability in any `juvantlabs/*` repo

Each `juvantlabs/*` repo (juvant-os, the MCP servers, libraries,
toolboxes) has its own `SECURITY.md` at the repo root. Use the channels
documented there.

## The disclosure process this handbook governs

This handbook is the **canonical reference** for how `juvantlabs`
handles security disclosures — both incoming (someone reports a vuln
to us) and outgoing (we audit a community alternative and find vulns
to report upstream). Full process: [`docs/security/disclosure-process.md`](docs/security/disclosure-process.md).

Templates:

- [`SECURITY-template.md`](docs/security/SECURITY-template.md) — every
  `juvantlabs/*` repo's `SECURITY.md` follows this baseline.
- [`audit-report-template.md`](docs/security/audit-report-template.md)
  — structure for outgoing audit reports (public gists).

## What we commit to

| State | Target |
|---|---|
| Acknowledge receipt | ≤ 7 days |
| Initial triage + severity | ≤ 14 days |
| Patch prepared (high/critical) | ≤ 30 days |
| Patch prepared (moderate) | ≤ 90 days |
| Public advisory + CVE | Patch + 1–7 days |

Reporters are credited by name unless they request anonymity. Past
disclosures: none yet (handbook is new); when the first arrives, an
acknowledgment list will be added here.
