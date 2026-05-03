# Audit report template

Structure baseline for outgoing audit reports (Path γ public gists per
[`disclosure-process.md`](disclosure-process.md)). The 2026-05-03
ftaricano audit
([gist](https://gist.github.com/juvantlabs/a9fe0a76a23b0c1260b1e0ad3194a6da))
follows this template.

Use as a starting point; remove sections that don't apply, add
sections where the audit subject warrants.

---

```markdown
# Security review — `<owner>/<repo>`

**Date**: YYYY-MM-DD
**Repository**: https://github.com/<owner>/<repo>
**Reviewed at commit**: `<branch>` head as of YYYY-MM-DD (`<short-sha>`)
**Reviewer**: `juvantlabs` security review process — conducted as part
of [pre-binding due diligence | other rationale].
**Distributed under**: CC0 / public domain. Re-use freely with or
without attribution.

---

## Summary

<One paragraph: what is the project, what was audited, headline
findings, verdict (PASS / PASS-WITH-CONCERNS / FAIL).>

A consistent theme worth surfacing if applicable: <e.g. "the codebase
contains a defense-in-depth layer that is well-designed but unreachable
from actual handlers — wiring it would mitigate findings X, Y, Z">.

The review was conducted statically (file-by-file read) plus standard
dependency tooling (`npm install`, `npm audit`, `npm test`, `npm run
lint`, `npm run build` — all completed successfully).

---

## Methodology

The review covered repository hygiene (lockfile, `.gitignore`, license,
README accuracy), source code (auth, tool handlers, error handling,
logging, network calls, validators), supply chain (`npm audit`,
transitive dependency review, maintainer activity), and test coverage.
Files audited are listed at the end.

Each finding is **`Cn`** (critical, security blocker class), **`Sn`**
(significant), or **`Mn`** (minor / nit). Each finding cites a file
path and line range traceable to the reviewed commit.

---

## Critical findings

### C1 — <one-line title>

**Path**: `<file>:<line-start>-<line-end>`

<Description: what the issue is, what attack scenario it enables, what
the impact is.>

**Severity**: high (CVSS:_____ if computed). **Suggested fix**: <concrete
remediation pointer>.

### C2 — <next>

...

---

## Significant findings

### S1 — <title>

...

---

## Minor findings

- <bullet list — file:line format>

---

## Supply chain

- **Maintainer**: <real account / throwaway? activity history>
- **Repo activity**: created date, last commit, stars / forks /
  watchers, issues filed all-time, PR review patterns
- **Bus factor**: <number of distinct contributors with substantial
  history>
- **Security disclosures**: any prior advisories filed
- **`npm audit`** / `pip-audit`: vulnerable package count + severity
  breakdown
- **Lockfile**: present? `resolved` URLs at canonical registries?
  reproducible installs?
- **Native deps**: any unmaintained / archived packages

---

## Suggested remediation summary

<Numbered list: most-impactful fixes first, with one-line rationale per
item. Keep concise — the detailed findings above carry the depth.>

---

## Files reviewed

<Comma-separated or bulleted list of every file path opened during the
review. Include manifest files (`package.json`, `pyproject.toml`),
config (`.eslintrc`, `tsconfig.json`), all source files, scripts,
tests sampled.>

Plus: `<commands run>` (e.g. `npm install`, `npm audit`, `npm test`)
— all succeeded / failed-as-noted. GitHub API queried for repo and
user metadata.

---

*This review was conducted in good faith as <stated rationale> and is
shared publicly to help the project. The reviewer welcomes a private
GitHub Security Advisory channel if the maintainer prefers.*
```

---

## Notes on writing the report

- **Third person, neutral tone.** "The reviewer" / "the audit" / "the
  codebase". Avoid "I think" or evaluative language.
- **Cite line numbers for every finding.** Without traceable
  citations, the report is opinions, not evidence.
- **Acknowledge what works.** If the codebase has well-designed
  components alongside the issues, say so. Audits that read as pure
  attack are perceived as hostile and reduce productive engagement.
- **Severity classification is yours, not the maintainer's.** Don't
  ask the maintainer to confirm severity before publishing — the
  audit is the audit; the maintainer responds.
- **Avoid speculation.** Stick to what the code does in the reviewed
  commit. "If a future change introduced X, this would also be
  vulnerable" is out of scope for an audit of the current state.
- **License the report CC0.** Audit reports are public-domain
  contributions to the OSS commons. Reuse-friendly.

## Distribution

The report typically lives as a **public GitHub gist** under
`juvantlabs/*`, filename pattern
`<owner>-<repo>-security-review-<YYYY-MM-DD>.md`. Description includes
project name + "security review" + audit summary + finding count for
search discoverability.

Cross-reference the gist URL from:

- The relevant `juvantlabs/juvant-os-pm` FEAT issue (when the audit
  was triggered by FEAT due-diligence)
- The MCP server's anti-pattern checklist (when findings inform
  what NOT to do in our own implementation)
- Project memory (`feedback_lean_canonical_mcp.md` or equivalent)
- ADRs that depend on the audit's verdict
