# ADR 0003 — Scope boundaries for MCP servers

## Status

Accepted (2026-05-04). Codifies a pattern that emerged during the
FEAT-014 [`juvantlabs/m365-graph-mcp-server`](https://github.com/juvantlabs/m365-graph-mcp-server)
scoping decisions, specifically:

- The decision to ship file + calendar capabilities together while
  excluding mail send and Teams chat from the same MCP server (recorded
  in
  [`m365-graph-mcp-server/ARCHITECTURE.md` § Scope](https://github.com/juvantlabs/m365-graph-mcp-server/blob/main/ARCHITECTURE.md#scope)).
- The recognition that outbound Teams notifications already ship via
  the `notification.sh` hook in the Juvant OS scaffolder, using a
  pre-shared Adaptive Cards webhook URL — without OAuth, without a
  Graph-mediated MCP server.

This ADR generalizes both choices into framework-level guidance for
future MCP server work.

## Context

A vendor like Microsoft Graph exposes a huge surface — files, calendar,
mail, Teams chat, Teams channels, OneNote, Planner, contacts, presence,
device management, tenant administration. Wrapping all of it in one MCP
server would be technically possible but would produce an MCP with:

- A large OAuth permission surface (`Files.ReadWrite.All`,
  `Mail.Send`, `ChannelMessage.Send`, `Tasks.ReadWrite`, …).
- Mixed threat models (file ops are confined to the user's drive; mail
  send broadcasts outside the tenant; Teams channel posts reach every
  channel member).
- A monolithic blast radius — a bug in one tool's input validation
  could be exploited to issue requests across any of the consented
  scopes.
- A binary trust decision for adopters: enable the MCP and accept all
  capabilities, or disable it and lose all of them.

Two distinct concerns emerged during FEAT-014 that this ADR consolidates:

### Concern 1 — Threat-model heterogeneity within one vendor

`m365-graph-mcp-server` ships file + calendar tools only because those
two capability domains share a *similar* threat model:

- Side effects scoped to the authenticated user's own drive / calendar
- Reversible on a short timescale (file delete recoverable from the
  recycle bin; event cancellable / recreatable)
- Confidentiality is the dominant concern; deliverability and
  reputation are not

Mail send (`POST /me/sendMail`) has a *materially different* threat
model:

- Broadcasts to external recipients, outside the tenant boundary
- SPF / DKIM / DMARC reputation impact on the company's mail domain
- Phishing leverage if the agent is compromised
- Irreversible (a sent mail cannot be unsent)

Teams chat / channel posts (`/teams/.../messages`) have a *third*
threat model:

- Reaches potentially-large internal audiences (channels with hundreds
  of members)
- Different OAuth scopes (`ChannelMessage.Send`, `Chat.ReadWrite`)
- Mass-broadcast accident class

A single MCP server holding all three would force every adopter to
either accept the full union of risks or do without — no granular
trust possible.

### Concern 2 — Capability mechanism heterogeneity

Some "Microsoft 365 capabilities" don't actually need an MCP server at
all. Outbound Teams notifications — the dominant use case for "agent
informs the user about something" — already ship as part of the
Juvant OS scaffolder's `notification.sh` hook:

- A Teams Adaptive Cards **incoming webhook** is created once per
  channel by the channel owner via Teams "Connectors → Incoming
  Webhook". The result is a per-channel HTTPS URL that authorizes
  POSTs of card payloads.
- The hook reads `JUVANT_NOTIFY_CHANNEL` env var, looks up
  `teams_webhooks.<channel-key>` in `.juvant/config.json`, POSTs an
  Adaptive Card to the URL.
- Zero OAuth flow. Zero Graph permissions. The webhook URL is the auth
  itself.

Building an MCP server for Teams notifications would be strictly worse:
more tokens to manage, more attack surface, more admin-consent friction,
without delivering any capability the webhook doesn't already cover.

## Decision

### 1. Scope each MCP server to a single threat-model boundary

When proposing a new MCP server (or extending an existing one), the
proposal must explicitly identify the **threat-model boundary** the
server's tools share. A boundary groups tools that:

- Have similar side-effect blast radius (same audience, same
  reversibility class)
- Require the same or near-identical OAuth scopes
- Could plausibly be enabled / disabled together by an adopter
  evaluating risk

Tools that don't fit the boundary go into a **separate MCP server**, not
into the same one. Concretely, for Microsoft Graph:

| Capability | MCP server | Status |
|---|---|---|
| OneDrive + SharePoint files + Calendar | `juvantlabs/m365-graph-mcp-server` | ✅ shipped (FEAT-014) |
| Mail send | `juvantlabs/m365-mail-mcp-server` (hypothetical) | Not built; only ship if a real Juvant OS need surfaces |
| Interactive Teams chat / channel post | `juvantlabs/m365-teams-mcp-server` (hypothetical) | Not built; same condition |

The naming convention `<vendor>-<capability>-mcp-server` makes the
split visible from the package name — adopters reading
`@juvantlabs/m365-mail-mcp-server` immediately understand it's
mail-only, not the full M365 surface.

### 2. Use webhooks for outbound-only notification capabilities

Before scoping a new MCP server, check whether the capability is
**outbound-only and one-way** (agent → external system, no read-back).
For those, prefer pre-existing webhook patterns over building an
OAuth-mediated MCP server:

| Capability | Mechanism | Why webhook over MCP |
|---|---|---|
| Teams notification → channel | Adaptive Cards incoming webhook | Pre-shared URL is the auth; no OAuth flow; per-channel scoped; channel owner controls who has the URL |
| Slack notification | Slack incoming webhook | Same |
| PagerDuty event | PagerDuty events API | Pre-shared integration key |
| Email send | SMTP relay (send-only) | Pre-shared SMTP creds; no OAuth surface |

The principle: **if the capability is one-way and bounded to a
specific destination the user pre-authorized, a webhook URL is
strictly safer than OAuth-issued credentials for that destination.**
Webhook URLs are:

- Per-destination scoped (one channel, one mailbox, one alert routing)
- Revokable independently (delete the webhook, no broader impact)
- Stateless (no token cache to manage, no refresh token rotation)
- Auditable (the webhook URL appears in the destination system's audit
  log; OAuth tokens don't reveal which agent issued each call)

An MCP server is only justified when the agent must **read state back**
from the system or **interact with conversations** in two-way fashion.
For Teams specifically, that means: a hypothetical
`m365-teams-mcp-server` is only warranted if there's a need for the
agent to read chat history or post in chat threads where humans expect
back-and-forth. Pure "agent informs user" use cases don't need it.

### 3. New MCP server proposals must justify their scope

Every FEAT issue proposing a new `juvantlabs/<vendor>-mcp-server` repo
must include a short scope justification answering:

1. **What threat-model boundary does this MCP cover?** (audience,
   reversibility class, OAuth scopes)
2. **What's intentionally out of scope for this MCP?** (the
   architecture's "Out of scope" section in `ARCHITECTURE.md`)
3. **For each excluded capability: would a webhook serve, or does it
   warrant a separate MCP later?**

The CA review on the FEAT issue checks these answers before
authorizing scaffolding. The scope justification lands as the
`ARCHITECTURE.md § Scope` section of the new repo.

### 4. Trigger to split an existing MCP into smaller ones

If a proposed *new tool* in an existing MCP server would broaden the
threat model — e.g. someone proposes adding a `send_email` tool to
`m365-graph-mcp-server` — the addition is **rejected**. The proposer
opens a separate FEAT issue for a new MCP server in the appropriate
threat-model boundary.

The CA + CSO joint review (per ADR 0002 § 5) checks for this on every
new tool proposal touching an existing MCP. Universal Boundaries
(`SYSTEM_INVARIANTS.md` § 4) catches the most egregious cases
structurally; this ADR's scope-discipline rule catches the subtler
cases that don't violate UB but still don't belong.

### 5. Concrete-only roles still apply

This ADR does not override [ADR 0002 § 7](0002-mcp-abstract-roles.md#7-concrete-only-roles).
A new MCP server can be either an abstract-role provider (e.g. another
`bank` provider alongside Finom and Mercury) or a concrete-only role
(e.g. Github, Turso). The scope-discipline rule applies to both:
even a concrete-only MCP must have a clean threat-model boundary
documented.

## Consequences

### Positive

- **Granular adopter trust.** An adopter wary of mail-send blast
  radius can enable `m365-graph-mcp-server` for files + calendar
  without also taking on mail-send risk. They make per-capability
  decisions, not all-or-nothing.
- **Smaller blast radius per MCP.** A bug in one MCP server's input
  validation can only do harm within that MCP's scope. A path-traversal
  bug in `m365-graph` can corrupt files but cannot send phishing
  email — those scopes aren't even loaded.
- **Cleaner OAuth consent prompts.** Each MCP requests only its own
  narrow set of scopes. The user / tenant admin sees exactly what's
  being asked for.
- **Reusable patterns.** Webhook-based notifications work the same way
  for Teams, Slack, PagerDuty, SMS providers — a single hook
  (`notification.sh`) handles the common shape regardless of
  destination.
- **Codifies an existing pattern.** The webhook → notification path
  has been working since v0.4 of Juvant OS; this ADR formalizes it so
  future contributors don't accidentally bypass it.

### Negative

- **More MCP server repos to maintain.** If Juvant OS ever needs
  mail-send + Teams chat alongside the existing m365-graph, that's two
  more `juvantlabs/*` repos with their own CI / publish / lifecycle.
  Mitigated by the scaffolder
  ([`juvantlabs/juvant-tools` → `scaffold mcp-server`](https://github.com/juvantlabs/juvant-tools))
  which makes the per-repo overhead small.
- **More `.juvant/config.json` entries.** Per-company instance config
  has one entry per active abstract role × MCP server. Mitigated by
  the wizard (Step 6 of company init), which collects all bindings in
  one pass.
- **Naming proliferation.** `m365-graph`, `m365-mail`, `m365-teams`
  qualifiers all coexist. Mitigated by the naming convention in
  [`docs/naming.md`](../naming.md) and the abstract-role catalog in
  [`MCP_INVENTORY.md`](https://github.com/juvantlabs/juvant-os/blob/main/docs/MCP_INVENTORY.md).

### Considered alternatives

- **Single `m365-mcp-server` covering the entire Microsoft 365
  surface.** Rejected — fails the threat-model boundary test.
- **One MCP server with capability flags** (e.g.
  `M365_ENABLE_MAIL_SEND=1` to opt into mail). Rejected — doesn't
  reduce the blast radius of a compromise (the code paths are still
  loaded), and obscures from adopters what scopes they're actually
  consenting to.
- **No formal scoping discipline; reviewed case-by-case.** Rejected —
  scope creep tends to win without a written rule. ADR 0002 already
  established the pattern of writing down what was previously
  implicit; this ADR continues that approach.

## Implementation status

- ✅ `juvantlabs/m365-graph-mcp-server` v0.1.3 ships with file +
  calendar tools; mail and Teams chat are documented as out-of-scope
  in its `ARCHITECTURE.md`.
- ✅ Webhook-based Teams notifications shipped in
  [`juvantlabs/juvant-os` `hooks/notification.sh`](https://github.com/juvantlabs/juvant-os/blob/main/hooks/notification.sh)
  since v0.4 and remain the canonical "agent informs user / channel"
  path.
- ⏳ This ADR's scope-discipline rule (§ 3, § 4) is **in force from
  2026-05-04**. New FEAT issues for MCP servers go through the scope
  justification check; new tool additions to existing MCPs are
  rejected when they cross the threat-model boundary.
- ⏳ `juvantlabs/m365-mail-mcp-server` and
  `juvantlabs/m365-teams-mcp-server` are **not built** and not on the
  roadmap. They will only be built if a Juvant OS use case actually
  surfaces (per § 1's "real Juvant OS need" gate).

## Re-evaluation triggers

- **A new vendor surface that's plausibly multi-threat-model.** When
  scoping (e.g.) a `juvantlabs/google-workspace-mcp-server`, apply
  this ADR: split Drive + Calendar from Gmail + Chat just as we did
  for M365.
- **An existing MCP server's threat model evolves.** If
  `m365-graph-mcp-server` adds a tool that expands its blast radius
  (e.g. tenant-wide search via the Search API touching mail bodies),
  re-check whether the new tool fits the original boundary or warrants
  promotion to a separate MCP. The CA + CSO review on each new tool
  is the gate.
- **Webhook patterns prove insufficient.** If a notification use case
  emerges that genuinely needs read-back from the destination (e.g.
  "did the user acknowledge the alert?"), revisit § 2 — that may
  warrant promoting from webhook to a small interactive MCP server.
  Until then, webhooks remain the default.

## References

- [ADR 0002 — MCP abstract roles + binding policy](0002-mcp-abstract-roles.md)
  — the abstract-role pattern this ADR builds on.
- [`docs/repo-types/mcp-server.md`](../repo-types/mcp-server.md) — the
  spec for building MCP servers, which references this ADR for scope
  discipline.
- [`docs/naming.md`](../naming.md) — naming rules for
  `<vendor>-<capability>-mcp-server` packages.
- [`juvantlabs/m365-graph-mcp-server/ARCHITECTURE.md` § Scope](https://github.com/juvantlabs/m365-graph-mcp-server/blob/main/ARCHITECTURE.md#scope)
  — the worked example that produced this ADR.
- [`juvantlabs/juvant-os/hooks/notification.sh`](https://github.com/juvantlabs/juvant-os/blob/main/hooks/notification.sh)
  — the canonical webhook-based notification path referenced in § 2.
