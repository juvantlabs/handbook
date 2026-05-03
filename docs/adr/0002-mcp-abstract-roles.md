# ADR 0002 — MCP abstract roles + binding policy

## Status

Accepted (2026-05-03). Codifies a pattern implicit since the
`agent_tool_matrix` v0 default seed was authored in
[`juvantlabs/juvant-os-pm/docs/session-commit-p1.md`](https://github.com/juvantlabs/juvant-os-pm/blob/main/docs/session-commit-p1.md)
on 2026-04-23 and operationalized via the FEAT-011 (Finom) and
FEAT-012 (Aruba) re-scoping decisions on 2026-05-03.

## Context

Some MCP capabilities the agent system needs are **provider-agnostic
in principle**: any of several real-world vendors can fulfill the same
abstract requirement.

- **Banking** — Finom, Mercury, Revolut Business, Wise. CFO needs
  `bank:read` regardless of which the company uses.
- **E-invoicing for SDI** (Italy) / equivalent regimes elsewhere —
  Aruba / Fatture in Cloud / TeamSystem in Italy; Spain SII; Mexico
  CFDI; Poland KSeF; France Chorus Pro. CFO needs
  `fattura_elettronica:read` regardless of which is used.
- **Microsoft Graph** — strictly speaking single-vendor, but the
  abstraction `m365-graph:rw` may be fulfilled by the official
  Microsoft Work IQ MCP servers OR by `juvantlabs/m365-graph-mcp-server`
  OR by future alternatives. The scope qualifier is what matters; the
  binding is a per-adopter choice.

Without a clean abstraction, two failure modes appear:

1. **Agent template provider lock-in** — if `cfo.md` referenced
   `finom:read` directly, every adopter who uses Mercury would have to
   fork the template. Provider-portability is lost.
2. **Ad-hoc role proliferation** — without a defined process, anyone
   shipping a new MCP server might invent a new qualifier and bind it
   to an agent unilaterally. The matrix loses coherence; the
   inventory becomes out-of-date.

The pattern that emerged across FEAT-011 / FEAT-012 / FEAT-014 is:
**abstract roles in the matrix**, **concrete provider bound at company
init via `.juvant/config.json`**, **inventory cross-check at the
boundary**. This ADR codifies that pattern.

## Decision

### 1. Abstract role qualifier syntax

The agent_tool_matrix references **abstract role qualifiers**, not
concrete provider names. Each qualifier has a documented entry in
[`juvantlabs/juvant-os/docs/MCP_INVENTORY.md`](https://github.com/juvantlabs/juvant-os/blob/main/docs/MCP_INVENTORY.md).

Naming rules for qualifiers (consistent with
[`docs/naming.md`](../naming.md)):

- **Single-word abstract roles**: lowercase, no underscores —
  `bank`, `buffer`, `github`, `turso`, `ms-graph`.
- **Multi-word abstract roles**: hyphens preferred over underscores —
  `m365-graph`, `fattura-elettronica`. (Note: existing matrix entries
  may use underscores like `fattura_elettronica`; new entries use
  hyphens. Existing entries are not renamed without an MCP_INVENTORY
  update + tool-matrix-change decision.)
- **Scope qualifier** appended with `:`: `<role>:<scope>`. Scope is
  one of `read` / `write` / `rw`. Examples: `bank:read`,
  `github:read`, `github:write`, `m365-graph:rw`.

### 2. Catalog of qualifiers (as of this ADR)

The canonical catalog lives in
[`juvantlabs/juvant-os/docs/MCP_INVENTORY.md`](https://github.com/juvantlabs/juvant-os/blob/main/docs/MCP_INVENTORY.md).
Today it has eight rows; this ADR references the inventory as
authoritative rather than duplicating the table.

For each qualifier, the inventory row records:

- **Server** (the qualifier itself, e.g. `bank`)
- **Scope** (`r` / `rw`)
- **Owning agent(s)** — which roles the matrix grants this to
- **Distribution** — concrete provider(s) and their packaging
- **Auth env vars** — what variables the MCP server expects
- **Status** — `shipped` / `pending FEAT-XXX` / `not yet specified`

### 3. Binding semantics

At company init, the wizard collects the concrete provider for each
abstract qualifier and writes it to `.juvant/config.json`:

```json
{
  "<role>": {
    "provider": "<concrete-provider-name>",
    "mcp_server": "<concrete-mcp-server-binding>",
    "scope": "read"
  }
}
```

Examples:

```json
{
  "bank": {
    "provider": "finom",
    "mcp_server": "npx @juvantlabs/finom-mcp-server",
    "scope": "read"
  },
  "fattura_elettronica": {
    "provider": "aruba",
    "mcp_server": "npx @juvantlabs/aruba-fattura-mcp-server",
    "scope": "read"
  },
  "m365-graph": {
    "provider": "juvantlabs",
    "mcp_server": "npx @juvantlabs/m365-graph-mcp-server",
    "scope": "rw"
  }
}
```

The agent template doesn't care which provider is bound; it consumes
`<role>:<scope>` tools through the MCP boundary. This is the
provider-portability property.

### 4. Universal Boundaries (per `SYSTEM_INVARIANTS.md` §4)

Some abstract roles are subject to **Universal Boundaries** — the CA
cannot grant write scope under any rationale, even if a vendor provides
write capability:

- `bank:write` — forbidden (would require ratifying a future
  `treasury` role; not on roadmap).
- `m365-mail` send — forbidden except v1.1 portal variants.
- `github:write` — forbidden except COO (single-writer invariant §4).

These boundaries are codified in
[`juvantlabs/juvant-os/SYSTEM_INVARIANTS.md`](https://github.com/juvantlabs/juvant-os/blob/main/SYSTEM_INVARIANTS.md)
§4 and re-stated in
[`juvantlabs/juvant-os/docs/MCP_INVENTORY.md`](https://github.com/juvantlabs/juvant-os/blob/main/docs/MCP_INVENTORY.md).
The inventory cross-check at company-init Step 8.5 rejects matrix
entries that violate them.

### 5. Process — introducing a new abstract role

When a new capability emerges that warrants an abstract role (multiple
potential providers, used by ≥ 1 agent template, durable enough to
warrant matrix-level entry), the process is:

| Step | Actor | Output |
|---|---|---|
| 1. Propose | CA | `tool-matrix-change` decision in Turso `decisions` table; rationale + proposed qualifier name + initial concrete provider candidate(s) |
| 2. Review | CA + CSO joint | CSO checks security implications (Universal Boundaries, scope minimization, audit-implications); CA checks architectural fit (lean canonical, abstract-role appropriateness, naming convention) |
| 3. Approve | CEO | Approval recorded on the `decisions` row |
| 4. Document | CA | Update `juvantlabs/juvant-os/docs/MCP_INVENTORY.md` with the new row in the appropriate spot (between existing rows, preserving alphabetical or domain-grouped order) |
| 5. Update matrix | CA via `install-spec` | `agent_tool_matrix` v0 default seed updated; immutable supersession (new row inserted, previous row's `superseded_by` set) |
| 6. Execute | COO | Opens the PR for the inventory + matrix change; CHRO + CA + CSO + CEthO review |

**Single-author additions are not permitted.** The CA + CSO joint
review is non-negotiable; without both, the addition lacks the security
review that prevents Universal Boundary violations.

### 6. Process — adding a new concrete provider for an existing abstract role

When a second provider for an already-defined abstract role ships
(e.g. Mercury MCP server alongside Finom for `bank:read`), the process
is lighter:

| Step | Actor | Output |
|---|---|---|
| 1. Build | CA-proposed FEAT issue, MCP server delivered per [`docs/repo-types/mcp-server.md`](../repo-types/mcp-server.md) | A `juvantlabs/mercury-mcp-server` (or equivalent) ready to publish |
| 2. Document | CA | Update `MCP_INVENTORY.md` row's "Distribution" column to add the new concrete provider alongside existing |
| 3. Approve | CEO | Lighter — no Universal Boundary check needed (the abstract role's scope is already decided); CA's architectural review on the new MCP server suffices |
| 4. Adopters bind | per-company | Each adopter chooses which concrete provider to bind in their `.juvant/config.json` at company init or via re-binding |

Per-adopter re-binding (switch from Finom MCP to Mercury MCP) is a
runtime operation: edit `.juvant/config.json`, run the wizard
"Re-bind <role>" sub-flow, restart agents. No agent_tool_matrix change.

### 7. Concrete-only roles

Some roles are intentionally **concrete, not abstract** — they bind to
exactly one provider with no realistic alternative:

- `turso` — one canonical provider. The matrix could be re-architected
  around an abstract `state-store` qualifier later, but the cost of the
  abstraction outweighs the benefit at single-vendor scale.
- `github` — same reasoning. The matrix could abstract to `git-host` if
  GitLab / Gitea binding ever ships, but that's hypothetical.

Concrete roles are flagged in `MCP_INVENTORY.md` with a "concrete"
note. Promotion of a concrete role to abstract follows the new-role
process (§5) when a second concrete provider lands.

## Consequences

### Positive

- **Provider portability** — adopters pick the bank / e-invoicing
  provider that fits their jurisdiction or commercial relationship;
  the agent template is unchanged.
- **Process clarity** — adding a new abstract role has a defined path;
  no ad-hoc proliferation.
- **Audit-friendly** — every abstract role has an inventory row with
  scope, owners, distribution, status. CSO Layer 5 audit checks the
  matrix against the inventory.
- **Structural enforcement of Universal Boundaries** — the inventory's
  "NOT exposed" section + Step 8.5 cross-check prevents accidental
  scope creep.

### Negative

- **Indirection cost** — an extra layer between agent template and
  concrete API. New contributors need to understand the abstract /
  concrete distinction. Mitigated by clear documentation in
  `MCP_INVENTORY.md` and this ADR.
- **Inventory drift** — if `MCP_INVENTORY.md` falls out of sync with
  shipped MCP servers, the cross-check fails on a false positive.
  Mitigated by making inventory updates part of every MCP server's
  ship checklist (per `docs/repo-types/mcp-server.md`).
- **Re-binding requires CEO awareness** — switching the bound provider
  for an abstract role is not silent (it's a config change committed to
  `.juvant/config.json` and acknowledged by the agent at next session).
  This is a feature, not a bug, but adds a small overhead vs. seamless
  re-binding.

## Implementation status

- ✅ Abstract role syntax codified in this ADR + `docs/naming.md`.
- ✅ Initial catalog in `juvantlabs/juvant-os/docs/MCP_INVENTORY.md`
  (8 rows, shipped at v0.5.0).
- ✅ Universal Boundaries in `juvantlabs/juvant-os/SYSTEM_INVARIANTS.md` §4.
- ✅ Wizard Step 8.5 cross-check (shipped at v0.5.0 in JUVANT_OS.md).
- ⏳ Concrete `juvantlabs/finom-mcp-server` (FEAT-011, Beta) — pending.
- ⏳ Concrete `juvantlabs/aruba-fattura-mcp-server` (FEAT-012, Beta) — pending.
- ⏳ Concrete `juvantlabs/m365-graph-mcp-server` (FEAT-014, Beta) — pending.
- ⏳ Process formalization (this ADR's §5 + §6) — **in force from
  2026-05-03**. New abstract roles or new concrete providers go through
  these flows from the next FEAT onward.

## Re-evaluation triggers

- **Multiple concrete providers shipped for the same abstract role.**
  When `juvantlabs/mercury-mcp-server` or `juvantlabs/revolut-mcp-server`
  lands alongside `juvantlabs/finom-mcp-server` (`bank:read`), validate
  that the abstract / concrete split actually delivers
  provider-portability in practice. Adjust the binding semantics if
  per-adopter re-binding turns out to be more friction than expected.
- **`bank-mcp-spec` normative interface.** When ≥ 2 concrete `bank`
  providers ship, consider opening a FEAT for a normative `bank-mcp-spec`
  document that defines the shared tool surface. Until then, each
  bank-provider MCP server defines its own tool list (with the
  matrix's `<role>:<scope>` qualifier as the only common contract).
- **Cross-framework abstract roles.** If `juvant-edu-os` or another
  framework variant ships and shares abstract roles with `juvant-os`,
  promote the inventory to a cross-framework location. Until then,
  `juvantlabs/juvant-os/docs/MCP_INVENTORY.md` is the single inventory.

## References

- [`juvantlabs/juvant-os/docs/MCP_INVENTORY.md`](https://github.com/juvantlabs/juvant-os/blob/main/docs/MCP_INVENTORY.md)
  — the canonical catalog this ADR points at.
- [`juvantlabs/juvant-os/SYSTEM_INVARIANTS.md`](https://github.com/juvantlabs/juvant-os/blob/main/SYSTEM_INVARIANTS.md)
  §4 (Single-Writer Invariant + Universal Boundaries) and §6 (Spec
  Authorization Matrix) — the framework-level invariants this ADR
  builds on.
- [`docs/repo-types/mcp-server.md`](../repo-types/mcp-server.md) — the
  spec for building new concrete MCP servers.
- [`docs/naming.md`](../naming.md) — naming rules for abstract role
  qualifiers (cross-reference).
- ADR 0001 — [GitHub account and organization structure](0001-account-and-org-structure.md).
- FEAT-011 issue at [`juvantlabs/juvant-os-pm#25`](https://github.com/juvantlabs/juvant-os-pm/issues/25)
  — the case that surfaced the lean canonical preference for `bank` provider MCPs.
- FEAT-012 issue at [`juvantlabs/juvant-os-pm#26`](https://github.com/juvantlabs/juvant-os-pm/issues/26)
  — the case that surfaced the licensing / hybrid pattern for community MCP alternatives.
- Project memory `feedback_lean_canonical_mcp.md` — the per-conversation
  feedback memory that captures the operational rule "MIT alone is not
  sufficient; binding requires audit".
