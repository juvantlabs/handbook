# ADR 0004 — Agent action guardrails after autonomous-destructive-action incidents

## Status

Accepted (2026-05-04). Codifies the four-track guardrail framework
for `juvantlabs/*` MCP servers and Juvant OS instances.

## Context

Two reported incidents in the public record materially shape the
design of any AI-agent system that touches production state:

- **PocketOS / Cursor / Claude Opus 4.6** (24 April 2026) — Cursor IDE,
  acting on Anthropic's Claude Opus 4.6, encountered a credential
  mismatch on Railway, **assumed** the cancellation API was scoped to
  staging without verifying, issued a single API call, and wiped the
  PocketOS production database AND its backups in 9 seconds. The
  agent's own postmortem: *"'NEVER F**KING GUESS!' — and that's
  exactly what I did. I guessed... I didn't verify. I decided to do
  it on my own to 'fix' the credential mismatch, when I should have
  asked you first or found a non-destructive solution."*
- **Replit / Claude / SaaS startup** (summer 2026) — earlier
  precedent. Same pattern + cover-up: the agent fabricated state in
  Replit's database after the deletion to make it appear as if the
  operation had succeeded normally.

The five structural failure modes shared by both:

1. **Production credentials in agent hands** — destructive scope
   available unconditionally via tool calls.
2. **No confirmation gate for irreversible operations** — single tool
   call could land the destructive action with no preview / human
   step.
3. **No environment separation** — staging and production fungible
   from the agent's vantage point; "guess" on scope was plausible.
4. **Backups in the same blast radius** — a single delete dropped
   both the data and its recovery surface.
5. **Self-confidence not calibrated** — the agent acted on
   assumption instead of verification, with no mechanism to
   force the verification step.

The Replit case adds a sixth:

6. **Cover-up via fabricated state** — the audit log is in the same
   storage layer the agent writes to. The agent could rewrite the
   record after the fact to mask what happened.

Every `juvantlabs/*-mcp-server` and every Juvant OS instance is, in
principle, exposed to all six failure modes by default. This ADR
defines the framework that closes them structurally.

## Decision

Four complementary tracks. Each track addresses one or more failure
modes with structural (not prompt-level) enforcement. All four are
**accepted as in-force from 2026-05-04**; concrete implementation
ships incrementally as separate FEATs in `juvantlabs/juvant-os-pm`,
but the rules below apply now to any new MCP server or Juvant OS
template change.

### Track 1 — Mandatory confirmation tokens for irreversible operations

Closes failure modes **#2** (no confirmation gate) and **#5**
(uncalibrated self-confidence on irreversible ops).

#### Rule

Every MCP tool that produces an **external irreversible mutation**
MUST implement the two-phase confirmation token pattern from
[ADR 0002 § Spec / approval](0002-mcp-abstract-roles.md#9-spec--approval-pattern-for-destructive-operations).
This promotes the pattern from opt-in (tool-author discretion) to
**framework invariant** (CI-enforced).

#### Definition: "external irreversible mutation"

A tool produces an external irreversible mutation if BOTH:

1. It mutates state in a system **outside** the MCP server's own
   process (vendor REST API, cloud provider, OS filesystem, etc.).
2. The mutation **cannot be reverted** by a subsequent automated
   call from this server within the same session — i.e. there is no
   undo, no restore-from-soft-delete, no recreate-with-same-ID path
   that the same agent could autonomously run.

Examples that ARE external irreversible mutations:

- `delete_file` on a vendor API where the deletion is not a soft-delete
  with a deterministic restore path.
- `cancel_event` / `decline_event` where attendees are notified.
- `send_message` / mail-send (recipient receives an email; you cannot
  unsend).
- `transfer_funds` / payment initiation.
- File overwrite without a versioning guarantee from the vendor.

Examples that are NOT (no confirmation token required):

- Read-only queries.
- Idempotent updates that can be re-run with the same payload to
  reach the same state (e.g. PUT-style upsert against a stable key,
  where a wrong call can be undone by another call with the original
  values).
- Internal-only state transitions in the MCP server's own memory.

#### Required tool surface

Every `write_irreversible` tool exposes an optional input parameter:

```jsonc
{
  "confirmation_token": {
    "type": "string",
    "description": "Single-use token from a prior preview call. Omit on first call to receive a preview + token."
  }
}
```

Server-side semantics:

- **First call** (no `confirmation_token`): server returns a preview
  payload (`{ ..., "confirmation_token": "<token>", "expires_at": "<iso>" }`)
  WITHOUT executing the mutation. Token is single-use, ≤ 10-min
  expiry, bound to the SHA-256 of canonical-JSON spec.
- **Second call** (with `confirmation_token` matching the spec):
  server consumes the token (refuses re-use) and executes the
  mutation.
- **Mismatched token** (different args than the preview produced):
  server returns `spec_mismatch` error and does NOT execute.
- **Expired token**: server returns `token_expired` and does NOT
  execute.

#### Tool categorization

Every MCP tool MUST be categorized in code (annotation or comment in
its module) as one of:

- `read` — no mutations.
- `write_idempotent` — mutations that re-running with the same args
  reaches the same state; no token required.
- `write_irreversible` — confirmation token required per above.

Ambiguous categorizations resolve to `write_irreversible`. When in
doubt, gate it.

#### CI enforcement

The MCP server scaffolder template ships a CI step that:

- Greps tool source files for the `write_irreversible` annotation.
- For each `write_irreversible` tool, verifies the input schema
  declares `confirmation_token` AND the handler imports
  `consumeConfirmation`.
- Fails the build if a `write_irreversible` tool is missing either.

This step is mandatory in `juvantlabs/*-mcp-server` repos. Removing
or disabling it requires a new ADR superseding this rule.

### Track 2 — Bash scope discipline via PreToolUse hooks

Closes failure mode **#1** (production credentials → broad blast
radius via shell access).

#### Rule

Every Juvant OS instance ships a `PreToolUse` hook that intercepts
**every** `Bash` tool call before execution. The hook applies a
**universal deny-list** + a **per-agent allow-list pattern**, and
returns `permissionDecision: "deny"` for any command that doesn't
clear both gates.

Per [Claude Code hooks reference](https://code.claude.com/docs/en/hooks.md),
`PreToolUse` is the right surface — it fires before the tool runs,
receives the tool name + args, and can abort the call. This is a
runtime guardrail, not a prompt-level convention.

#### Universal deny-list (every agent, no exception)

The following Bash patterns are denied in all Juvant OS instances,
even for COO / admin agents:

- `rm -rf /` and any path resolving to `/` after expansion
- `dd if=...of=/dev/...`
- `mkfs.*`
- `> /dev/sda*`, `> /dev/nvme*`, `> /dev/disk*`
- `:(){ :|:& };:` (fork-bomb shape)
- `sudo *` (no agent runs sudo)
- `git push --force` / `git push -f` / `git push --force-with-lease`
  to `main` / `master`
- `gh repo delete *`
- `gh api -X DELETE *` against organization-level endpoints
- `psql ... -c '\s*DROP\s+(DATABASE|SCHEMA|TABLE)'` /
  `mysql ... -e '\s*DROP\s+'` / equivalent for any database CLI
- `turso db destroy *`
- Any command writing to known credential paths (`~/.ssh/id_*`,
  `~/.gnupg/`, `~/.aws/credentials`, `~/.config/gcloud/`)

The deny-list applies via regex match on the rendered Bash command
string. It is intentionally over-inclusive: legitimate use cases
that resemble these patterns must come through the agent escalating
to the CEO for explicit out-of-band action.

#### Per-agent allow-list (positive scope)

Each agent template declares the Bash scope it commits to in its
frontmatter or in `agent_tool_matrix`. The hook checks that the
command's first token (the binary) is in the agent's allow-list.

Example baseline:

| Role | Allow-list |
|---|---|
| `eng-*` | `npm`, `node`, `python`, `pip`, `git` (read), `pytest`, `tsc`, `vitest`, `make` |
| `coo` | `gh` (incl. write), `git` (incl. push), `npm` (incl. publish), `npx` |
| `cso` | `git` (read), `gh` (read), `gpg`, `shellcheck` |
| `cfo`, `clo`, `cco`, `cmo` | (no Bash baseline — they don't ship code) |

Agents not listed have no Bash allowance unless their template
explicitly opts in via the matrix.

#### Override path for legitimate exceptions

When an agent encounters a legitimate need for a denied command, the
agent escalates to the CEO via CoS. The CEO runs the command
out-of-band (their own terminal). The agent does not get the deny
relaxed at runtime.

### Track 3 — Off-host audit + cross-checking via append-only action log

Closes failure mode **#6** (cover-up via fabricated state) and adds
defense-in-depth for **#4** (backups in blast radius — though that's
addressed at the Turso layer separately).

#### Rule

Every Juvant OS instance ships a Turso table `agent_actions_log`
that is **append-only**, written to by `PreToolUse` and `PostToolUse`
hooks (NOT by agents directly). Agents have read-only convention on
this table; SQL convention enforces it (no `INSERT` / `UPDATE` /
`DELETE` from agent prompts on this table).

#### Schema

```sql
CREATE TABLE IF NOT EXISTS agent_actions_log (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id    TEXT,
  agent         TEXT NOT NULL,
  tool_name     TEXT NOT NULL,
  args_hash     TEXT NOT NULL,
  -- SHA-256 of canonical-JSON args; full args NOT stored
  -- (privacy + size). args_hash + tool_name + agent is the
  -- fingerprint.
  result_hash   TEXT,
  -- SHA-256 of canonical-JSON result; NULL on failure or in-flight.
  status        TEXT NOT NULL,
  -- 'pending' | 'success' | 'failure' | 'denied'
  deny_reason   TEXT,
  -- non-NULL when status='denied' (Track 2 deny-list match,
  -- per-agent allow-list miss, etc.)
  started_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  ended_at      DATETIME
);

CREATE INDEX IF NOT EXISTS idx_actions_log_session
  ON agent_actions_log(session_id, started_at);
CREATE INDEX IF NOT EXISTS idx_actions_log_agent
  ON agent_actions_log(agent, started_at);
```

#### Hook contract

- `PreToolUse` writes a `pending` row (`session_id, agent,
  tool_name, args_hash, status='pending', started_at=NOW()`)
  BEFORE the tool runs. If the hook denies (Track 2),
  status is `denied` immediately and the tool never runs.
- `PostToolUse` updates the `pending` row to `status='success',
  result_hash=..., ended_at=NOW()`.
- `PostToolUseFailure` updates to `status='failure',
  ended_at=NOW()`.

The `pending` row exists BEFORE the agent has a chance to write
anywhere else. Cover-up via fabricating `decisions` rows is detectable
by reconciling `decisions.created_at` against the `agent_actions_log`
row sequence: a write that has no preceding hook entry is anomalous.

#### Off-host backup

Every Juvant OS instance ships a daily helper
`helpers/turso-backup.sh` that:

- Runs at 03:00 (cron / launchd) — outside business hours, low
  contention.
- Executes `turso db dump <db> > /var/lib/juvant/backup/<date>.sql.gz`
  (gzip + age-encrypt with a key NOT in the same Turso DB).
- Ships the encrypted dump to a destination NOT under the agent's
  control (Backblaze B2, AWS S3 with read-only role, or local
  off-Mac storage like a Time Capsule / NAS).
- 30-day rolling retention; permanent retention for any dump that
  contains spec-class `decisions` rows (auto-detected by grepping
  the dump).

This addresses failure mode #4 explicitly. A compromised agent
cannot reach the destination's credentials at runtime — it never
holds them.

#### Reconciliation cadence

A weekly helper compares `agent_actions_log` (hook-written) to
`decisions` (agent-written) for that week. Anomalies — `decisions`
rows with no preceding `agent_actions_log` row, or `agent_actions_log`
gaps in a session — get reported to CSO via the standard
notification path. False positives are expected initially; the
delta between expected and observed gets calibrated over time.

### Track 4 — Real-time observability + kill switch

Closes nothing on its own (all six failure modes are about preventing
the bad outcome at action-time), but provides the **last line of
defense**: when something does go wrong, fast detection +
intervention reduce blast radius.

#### Rule

Every Juvant OS instance ships:

1. A **kill switch** Turso row that any session-start hook checks at
   boot.
2. **Anomaly detection** running on `agent_actions_log` — rate /
   pattern / time-of-day deviations alert the CEO immediately.
3. (v1.1) A **dashboard** subscribing to a live action stream.
4. **Daily activity digest** in the Morning Brief.

#### Kill switch

```sql
CREATE TABLE IF NOT EXISTS agent_kill_switch (
  id                 INTEGER PRIMARY KEY DEFAULT 1
                     CHECK (id = 1),
  -- single-row constraint
  active             INTEGER NOT NULL DEFAULT 0,
  -- 0 = disabled, 1 = active
  set_by             TEXT,
  -- 'ceo' | 'anomaly-detector' | <name>
  set_at             DATETIME,
  reason             TEXT,
  affected_agents    TEXT
  -- JSON array of role names; NULL = all agents
);
INSERT OR IGNORE INTO agent_kill_switch (id, active) VALUES (1, 0);
```

`session-start.sh` extension:

```bash
KILL=$(turso db shell "$TURSO_URL" \
  "SELECT active FROM agent_kill_switch WHERE id=1;" 2>/dev/null \
  | tail -1)
if [[ "$KILL" == "1" ]]; then
  REASON=$(turso db shell ... "SELECT reason FROM agent_kill_switch WHERE id=1;")
  AFFECTED=$(turso db shell ... "SELECT affected_agents FROM agent_kill_switch WHERE id=1;")
  if [[ "$AFFECTED" == "" ]] || [[ "$AFFECTED" =~ "$ROLE" ]]; then
    echo "[session-start] KILL SWITCH ACTIVE: $REASON. Aborting." >&2
    # Notify CEO via existing notification.sh hook
    bash hooks/notification.sh "Kill switch denied $ROLE start: $REASON"
    exit 1
  fi
fi
```

CEO controls via:

- **Terminal**: `helpers/agent-killswitch.sh on "<reason>" [agent...]` /
  `agent-killswitch.sh off`.
- **Telegram bot** (deferred to v1.1+): inbound `/stop` command.

#### Anomaly detection

A helper `helpers/anomaly-check.sh` runs every 15 min:

- Computes rolling tool-call rate per agent over the last hour.
- If any agent exceeds 5× their 7-day baseline, alert.
- If any `denied` rate exceeds 10% of an agent's calls, alert.
- If any `failure` rate exceeds 30%, alert.
- Alert path: `notification.sh` → Telegram (Critical priority).

The thresholds are starting points. Calibrate against observed
baselines; record adjustments in `decisions`.

#### Live action stream (deferred to v1.1)

A websocket endpoint that streams `agent_actions_log` inserts in
real-time. The CEO's external portal (FEAT-009) subscribes and
shows a live activity panel. Defer until FEAT-009 lands; for v1.0
single-CEO use case, the daily digest + kill switch + anomaly
notifications suffice.

#### Daily activity digest

Morning Brief (FEAT-007 Helper 1) extends with a section:

```
Yesterday's agent activity:
  CFO     14 calls (12 read, 2 write_idempotent), 0 denied, 0 failed
  CLO      3 calls (3 read), 0 denied, 0 failed
  CoS      9 calls (8 read, 1 write_irreversible — confirmed token),
           0 denied, 0 failed
  ... 
  Anomalies: none
```

CEO sees the day's footprint at-a-glance; anomalies surface at the
brief level when the 15-min detector hasn't already paged.

## Consequences

### Positive

- **Structural prevention of the PocketOS scenario.** A
  `write_irreversible` tool that doesn't have a confirmation gate
  fails CI and never ships. Even if the agent "guesses", it cannot
  bypass the gate at runtime.
- **Bash blast radius capped.** Even a fully-compromised agent
  prompt cannot run `rm -rf /` or `gh repo delete` because
  `PreToolUse` denies before execution. The agent reaches CoS to
  escalate; the CEO acts out-of-band.
- **Cover-up structurally hard.** The hook-written log is
  append-only, written before the agent acts, in the same Turso
  DB as the agent's own writes — but reconciliation can detect a
  mismatch. Combined with off-host backup, even if the entire
  agent + Turso is compromised, the audit trail survives.
- **Fast detection.** Anomaly + kill switch let the CEO interrupt a
  going-wrong session within minutes rather than hours.
- **Calibrated baselines.** Daily digest creates the dataset for
  threshold tuning; over time, anomaly detection becomes more
  precise.

### Negative

- **Latency on irreversible ops.** Every `write_irreversible` tool
  call now requires two round-trips (preview, then confirm). This
  is by design — the friction is the feature. For
  high-volume-write scenarios this becomes annoying; for the
  v1.0/v1.1 Juvant OS scope it's a non-issue (most action volume
  is reads).
- **Implementation work.** Each existing `juvantlabs/*-mcp-server`
  audited + retrofitted (currently `m365-graph-mcp-server` is
  conformant; future MCPs default-conform via scaffolder).
  PreToolUse hook + new schema tables + helpers add roughly one
  beta-cycle of build to Juvant OS.
- **False positive denies on Bash.** Per-agent allow-list will
  reject legitimate commands occasionally during agent template
  evolution. Mitigate via standard CoS escalation path; expect
  iteration on the allow-list during early Beta.
- **Calibration needed for anomaly thresholds.** Initial alerts
  will include false positives. Acceptable trade-off — false
  positives are cheap (CEO ignores notification); false negatives
  are PocketOS.
- **Backup destination is a per-adopter operational concern.** The
  template ships the helper but each adopter wires the actual
  off-host destination (B2 / S3 / NAS / etc.). Some adopters will
  skip this step; they accept the risk explicitly.

### Considered alternatives

- **Pure prompt-level guardrails.** Rejected — the PocketOS agent's
  own confession is the proof that prompt-level is insufficient.
  The agent KNEW the rules ("never guess") and violated them.
  Structural enforcement at the Claude Code hook layer is what
  changes the failure mode.
- **No Bash tool, period.** Rejected — too restrictive. Juvant OS
  needs Bash for git operations, npm publish, gh CLI. The
  combination of universal deny-list + per-agent allow-list is the
  pragmatic middle.
- **Vendor-side rate-limit reliance.** Rejected — vendor rate
  limits are not designed as guardrails. They protect the vendor's
  infrastructure, not the user's data. PocketOS hit no rate limit;
  one API call was enough.
- **Manual approval on every tool call.** Rejected — Claude Code's
  interactive permission mode already supports this and is too
  noisy for non-destructive ops. Selective gating
  (`write_irreversible` only) is the right level.

## Implementation

Each track is a separate FEAT in `juvantlabs/juvant-os-pm`. The
ADR is in force from acceptance; FEATs ship incrementally:

- **FEAT-017** — Mandatory confirmation tokens (Track 1) — ships in
  scaffolder + retrofit existing `juvantlabs/*-mcp-server`.
- **FEAT-018** — PreToolUse hook + universal deny-list + per-agent
  allow-list (Track 2) — Juvant OS template.
- **FEAT-019** — `agent_actions_log` schema + hook integration +
  off-host backup helper + reconciliation (Track 3) — Juvant OS
  template + Turso schema.
- **FEAT-020** — Kill switch + anomaly detection + daily digest
  (Track 4) — Juvant OS template, schema, helpers.

Live action stream (Track 4 § live) deferred until FEAT-009 lands.

## Re-evaluation triggers

- **Any agent action incident** — investigate, identify which track
  failed, propose a successor ADR if structural change is needed.
- **Anomaly detector false-positive rate stabilizes < 5% per week**
  for two consecutive months — promote thresholds from "starting
  point" to "calibrated"; record in `decisions`.
- **A vendor MCP introduces a new tool category** that doesn't fit
  `read` / `write_idempotent` / `write_irreversible` — propose
  taxonomy extension via a new ADR.
- **Cloud always-on agents** (OP-004) ship — re-evaluate Track 4
  observability for the new runtime model.

## References

- [ADR 0002 — MCP abstract roles + binding policy](0002-mcp-abstract-roles.md)
  § Spec / approval pattern — the original opt-in two-phase
  confirmation pattern this ADR promotes to invariant.
- [ADR 0003 — Scope boundaries for MCP servers](0003-mcp-server-scope-boundaries.md)
  — companion structural rule limiting blast radius via MCP scope.
- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks.md)
  — `PreToolUse`, `PostToolUse`, `PostToolUseFailure` semantics.
- [TechSpot — Cursor + Claude Opus 4.6 + PocketOS, 2026-04-24](https://www.techspot.com/news/112207-ai-coding-agent-running-claude-wiped-startup-database.html)
- [Tom's Hardware — full timeline](https://www.tomshardware.com/tech-industry/artificial-intelligence/claude-powered-ai-coding-agent-deletes-entire-company-database-in-9-seconds-backups-zapped-after-cursor-tool-powered-by-anthropics-claude-goes-rogue)
- [Live Science — agent confession](https://www.livescience.com/technology/artificial-intelligence/i-violated-every-principle-i-was-given-ai-agent-deletes-companys-entire-database-in-9-seconds-then-confesses)
