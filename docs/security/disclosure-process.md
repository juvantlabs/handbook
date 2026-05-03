# Security Disclosure Process

How `juvantlabs` handles **incoming** vulnerability reports (someone tells
us about a vuln in our code) and **outgoing** vulnerability disclosures
(we audit a community alternative and find vulns we want to report).

This document is the canonical reference. Every `juvantlabs/*` repo's
`SECURITY.md` follows the [SECURITY-template.md](SECURITY-template.md)
in this directory and points back here for process detail.

## Incoming reports — someone reports a vuln to us

### Channels (preferred order)

1. **GitHub Security Advisory** — the canonical channel. Every
   `juvantlabs/*` repo enables "Privately report a vulnerability" via
   `Settings → Security and analysis → Private vulnerability reporting`.
   Reporter clicks "Report a vulnerability" on the repo's Security tab
   and submits a draft advisory visible only to reporter + maintainer.
2. **Email** — fallback when private GitHub reporting isn't yet enabled
   on a repo: `security@juvant.io` (route to the primary identity, today
   Antonio Gatti). Do NOT use issues for vulnerability reports — they
   are public from the moment of submission.

### Triage timeline (Service Level Objectives)

| State | Target | Notes |
|---|---|---|
| Acknowledge receipt | ≤ 7 days | Reply to the reporter confirming we received the report and that we'll triage |
| Initial triage + severity classification | ≤ 14 days | CVSS v3.1 score + impact assessment (which repos / versions affected, exploitability) |
| Patch prepared | ≤ 30 days for **high/critical**, ≤ 90 days for **moderate**, best-effort for **low** | If a patch isn't feasible in window, communicate the constraint to the reporter and propose a longer embargo |
| Public disclosure (advisory + release) | Patch + 1–7 days | Buffer lets dependents update before the advisory goes public; CVE assignment via GitHub |

### Process

1. **Receive + acknowledge** — reply within 7 days. If you can't fully
   triage yet, just acknowledge.
2. **Triage** — classify severity (CVSS v3.1), confirm reproducibility,
   identify affected versions, identify dependents (downstream Juvant
   OS instances, npm consumers, etc.).
3. **Coordinate with reporter** — agree on disclosure timeline. Default
   embargo follows the timeline above; reporter may ask for shorter
   (their threat model justifies faster public exposure) or longer
   (their employer requires more notice). Reasonable requests are
   accepted; document the rationale in the GitHub Security Advisory
   draft.
4. **Patch** — develop the fix in a private fork (or in the advisory's
   built-in private fork feature). Include tests that exercise the
   vulnerability path. CSO Layer 5 review the patch before merge.
5. **Pre-release notification** — for high/critical severity affecting
   downstream Juvant OS instances, notify `juvantio/juvant` (and any
   known external adopters) ~24h before public disclosure so they can
   prepare to upstream-sync.
6. **Release + advisory** — merge the patch, publish a new release with
   patched version (semver patch bump), publish the GitHub Security
   Advisory (which auto-requests a CVE), update CHANGELOG.md with a
   `### Security` section linking to the advisory.
7. **Credit the reporter** — unless they request anonymity, name them
   in the advisory. Reporter contributions are valued and visibly
   acknowledged.
8. **Post-mortem** (high/critical only) — open a follow-up handbook
   issue documenting: how the vuln got introduced, what testing /
   review missed it, what process change prevents recurrence. The
   post-mortem is public; embarrassment is the price of credibility.

### What NOT to do

- **Do not silently fix.** Every security-impact bug fix gets a public
  advisory. Silent fixes appear faster but break the trust contract:
  downstream consumers can't tell that a patch matters.
- **Do not blame the reporter.** Even if the reporter's framing is
  hostile or imprecise, respond professionally. Hostile responses
  poison the disclosure pipeline for everyone.
- **Do not bypass the embargo.** If you ship a patch with vulnerability
  details in the commit message before the embargo expires, you've
  effectively published the vuln. Patches reference advisory IDs, not
  attack details, until the embargo ends.

## Outgoing disclosures — we audit a community alternative and find vulns

When evaluating community alternatives (per
[`mcp-server.md`](../repo-types/mcp-server.md) "Lean canonical MIT
preference" pattern, or any audit of upstream code), audit findings
follow this process.

### Decision tree post-audit

After completing an audit (verdict PASS / PASS-WITH-CONCERNS / FAIL):

1. **PASS** → bind the dependency directly. No disclosure needed.
2. **PASS-WITH-CONCERNS** → bind with the documented stipulations + file
   issues / PRs against the upstream for the concerns. Helpful, not
   urgent.
3. **FAIL** → don't bind. Decide on disclosure separately based on
   maintainer signals (next section).

### Maintainer-signal decision tree

When the audit returns FAIL, decide whether and how to disclose by
reading the maintainer's engagement signals on the upstream repo:

| Signal | Disclosure path |
|---|---|
| Maintainer has enabled Private Vulnerability Reporting on the repo | **Path α**: file private GitHub Security Advisory with full details. Standard 90-day embargo. Coordinate fix. |
| Maintainer has SECURITY.md with email or GitHub Security Advisory channel | **Path α** (same as above) |
| Maintainer responds to public issues actively (recent issues with engagement, not all PRs self-merged) | **Path β**: file a public meta-issue (no vuln details) requesting they enable Private Vulnerability Reporting; offer to switch to private once channel exists |
| Maintainer signals "closed shop" (zero issues filed all-time, all PRs self-merged, no SECURITY.md, no public email, no twitter) | **Path γ**: publish a public audit gist under `juvantlabs/*` (CC0); do NOT file an issue on the upstream repo. Information is preserved for other potential adopters; the closed-shop signal is respected. The 2026-05-03 ftaricano case followed this path |

### When path γ applies

`ftaricano/mcp-onedrive-sharepoint` audit (2026-05-03) is the canonical
case for Path γ. Maintainer signals as observed:

- No `SECURITY.md` in the repo
- "Privately report a vulnerability" not enabled
- 0 issues filed all-time on the repo
- 7 PRs total, all author-authored, all closed-merged (no community
  review participation)
- No email on GitHub profile
- No Twitter on GitHub profile
- Commit message style "audit phase 3 follow-ups (iteration 2)" — author
  audits the repo unilaterally on their own terms

These signals together meant: a private advisory would likely sit
ignored, a public meta-issue would likely get closed without action.
The Path γ choice (public gist, no upstream issue) preserved
information value for downstream adopters without burning effort on
unlikely-yield engagement.

### Path γ procedure

1. **Write the audit report** in clean third-person, neutral, with
   file:line citations for every finding. Severity classification per
   finding. Use the [audit-report-template.md](audit-report-template.md)
   as the structure baseline.
2. **Publish as public gist** under `juvantlabs` user account, CC0
   licensed. Filename `<vendor>-<project>-security-review-<YYYY-MM-DD>.md`.
   Concise description that's discoverable via Google search for the
   target project name + "security review".
3. **Cross-reference** the gist URL from any internal artifacts that
   informed the FAIL decision (e.g. FEAT issue body, MCP server's
   anti-pattern checklist, ADRs).
4. **Do NOT file an issue on the upstream repo.** The closed-shop signal
   says it would not be productive. Information lives in the gist; the
   upstream maintainer can find it via the same search any other
   adopter would use.
5. **Document the decision** in project memory or in the relevant FEAT
   issue: explain why Path γ was chosen over Path β (record the
   maintainer signals).

### Path β procedure (public meta-issue + offer of private channel)

1. Open a **public issue** on the upstream repo with:
   - Brief intro of who you are (juvantlabs context)
   - Statement that you have security findings to share
   - **No vuln details whatsoever in the public issue body**
   - Request to enable Private Vulnerability Reporting on the repo
   - Offer alternative private channels (email, etc.) if they prefer
2. Set internal **30-day timer**. If maintainer responds: switch to
   private advisory. If maintainer ignores past 30 days: re-evaluate
   (escalate to public disclosure with details + 60-day notice, OR
   downgrade to Path γ + close the meta-issue).
3. **Never disclose vuln details in the public meta-issue.** Once you
   click submit on a public issue with reproduction details, the embargo
   is gone — you've published the vulnerability.

### Path α procedure (private GitHub Security Advisory)

1. File the advisory at `https://github.com/<owner>/<repo>/security/advisories/new`.
2. Fill in: title, description, severity (CVSS), CWE classification,
   affected versions, suggested patches, reproduction details.
3. Standard 90-day embargo.
4. Coordinate with maintainer on fix + release timeline.
5. Public disclosure happens via the advisory's own publish-date —
   automatic CVE assignment from GitHub.

## Templates

- [`SECURITY-template.md`](SECURITY-template.md) — every `juvantlabs/*`
  repo's SECURITY.md follows this template.
- [`audit-report-template.md`](audit-report-template.md) — structure
  baseline for outgoing audit reports (Path γ gists).

## Communication conventions

- **Tone**: neutral, technical, professional. Findings are about code,
  not character. Hostile language poisons disclosure pipelines.
- **Acknowledgment**: reporters credited by name in advisories unless
  they request anonymity.
- **Embargo respect**: never publish vuln details before the agreed
  date. If we publish a patch, the patch references the advisory ID
  but not the attack details — those wait for the public advisory date.
- **Joint disclosure on shared dependencies**: when our advisory affects
  a transitive dependency that's also vulnerable elsewhere (e.g. axios
  SSRF), coordinate with the upstream's own advisory schedule.

## Related

- ADR 0001 — [GitHub account and organization structure](../adr/0001-account-and-org-structure.md)
- The 2026-05-03 [ftaricano/mcp-onedrive-sharepoint audit](https://gist.github.com/juvantlabs/a9fe0a76a23b0c1260b1e0ad3194a6da) — canonical Path γ case study
- Repo type spec — [`mcp-server.md`](../repo-types/mcp-server.md) — pre-binding due diligence audit pattern (the audit step that may produce outgoing findings)
