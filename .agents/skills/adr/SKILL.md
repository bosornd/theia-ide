---
name: adr
description: >-
  Creates Any Decision Record (ADR) for the user to analyze, review, and
  approve among alternatives for a specific concern before committing. Select
  when the user asks for options, tradeoffs, a recommended direction, or needs
  to decide among valid approaches — architectural or otherwise. Explicitly
  identifies and guides the user through downstream/derived concerns that
  arise from a chosen solution.
---

# ADR — Any Decision Record Skill

An ADR is a short, reviewable document for **one concern** (a specific question the user must resolve) with **viable options**, tradeoffs, and the user's approved choice. Scope is not limited to software architecture — any consequential choice that affects a deliverable qualifies (technology, structure, workflow, naming, scope, etc.).

| | Concern | Options |
|---|---------|---------|
| **What** | The question under review | Valid alternatives (A, B, C, …) |
| **Example** | Which database to use for item storage | MySQL, MongoDB, PostgreSQL, … |

**One ADR file = one concern.** Options are answer candidates for that concern, not separate files.

---

## When to Create an ADR

Apply these filters **in order**. Fail any → do not create (or chain) an ADR.

1. **Scope** — Does it materially affect the current task's goal, requirements, deliverable, or acceptance criteria?
2. **Consequence** — Would a poor choice create meaningful cost, risk, quality impact, user-visible behavior, or expensive rework (including hard-to-reverse commitments)?
3. **Tradeoff** — Are there ≥ 2 viable options with materially different consequences? If not, use the clear default and tell the user.
4. **Depth** — Is this a **direct** consequence of a parent decision and **necessary** to finish the current task? Skip speculative or merely adjacent branches.

Failed concerns get one of these labels (no ADR):

| Label | Meaning |
|-------|---------|
| **out of scope** | Defer without analysis |
| **directly resolved** | Pick a conventional default; notify the user |
| **covered** | Already settled by an existing ADR, requirement, policy, or project convention |
| **deferred assumption** | Record the assumption and the trigger to revisit it |

State why the chain stopped. There is no target ADR count — depth follows task risk, not completeness.

---

## Dedup Before Creating

Before saving, search `{workspace}/adr/` (filename, `## Concern`, **Related** links):

- **Same/similar concern** → merge into the existing file, or set `Status: superseded` and link to the replacement if the concern is reframed.
- **No match** → create `{workspace}/adr/{concern-slug}.md`.

---

## Chaining

- Parent decisions may spawn several follow-up concerns; each distinct concern gets its own ADR, linked under **Related**.
- Open **at most one** follow-up ADR at a time (highest-impact `ADR candidate`), and only after the user agrees — never auto-open or batch speculative ADRs.
- A follow-up must independently pass the filters above.

---

## Workflow

### 1. Identify the Concern

State the concern as a clear question. Capture briefly: **Concern**, **Context**, **Stakes**.

### 2. Enumerate Options

Present **≥ 2** real (non-strawman) options. Each option: short name + 2–4 bullets (what it is, Pro, Con).

### 3. Compare Tradeoffs

Add a scannable **Tradeoffs** section (table or bullet pairs). Optionally add an agent **Recommendation**.

### 4. Save ADR

Write with the template below. Set `Status: proposed`, add a dated **Notes** entry, save to `{workspace}/adr/{concern-slug}.md`.

**Language:** Body prose (Concern, Context, Decision, Options, Tradeoffs, Recommendation, Consequences, Downstream Concerns, Related, Notes) uses the user's current language. Section headings, status values, and classification labels stay English as in the template.

### 5. User Review Gate

Guide the user through options with structured questions. **Never** set `approved` without explicit user confirmation. If the user says "just pick one" on a qualifying concern, still present alternatives and tradeoffs; only non-qualifying concerns may be resolved by default.

### 6. Record Approval & Triage Downstream

On explicit approval:

1. `Status: approved`
2. **Decision** = chosen option + one-line rationale
3. Triage **Downstream Concerns** — only in-scope, direct, consequential questions that pass the filters. Label each:
   - `ADR candidate` — needs its own option comparison
   - `directly resolved` / `covered` / `out of scope` / `deferred assumption` — with default, basis, or revisit trigger
4. Update **Related** (parent / sibling / follow-up paths)
5. Append a dated **Notes** entry

### 7. Optionally Chain One Follow-Up

If an `ADR candidate` remains, ask whether to open **one** ADR for the highest-impact item. If none remain, close the chain and state why.

---

## File Template

```markdown
# {Concern Title}

## Concern
{The question under review — e.g. Which database should we use for item storage?}

## Status
proposed | approved | rejected | superseded

## Context
- {Why this concern matters now}
- {Relevant constraints or prior decisions}

## Decision
{Empty while proposed. After approval: chosen option + one-line rationale.}

## Options
### Option A — {Short Name}
- {What it is}
- Pro: {key benefit}
- Con: {key cost or risk}

### Option B — {Short Name}
- {What it is}
- Pro: {key benefit}
- Con: {key cost or risk}

## Tradeoffs
| | Option A | Option B |
|---|----------|----------|
| {dimension} | … | … |

## Recommendation (optional)
- {Agent suggestion and why}

## Consequences
- {What follows if the approved option is chosen}
- {What we are explicitly not doing}

## Downstream Concerns
- [ ] {ADR candidate: in-scope consequential question}
- [x] {directly resolved: default + rationale}
- [x] {covered | out of scope | deferred assumption: basis or revisit trigger}

## Related
- {Paths to parent, sibling, or follow-up ADRs / related artifacts}

## Notes
- {YYYY-MM-DD}: {created as proposed | status → approved/rejected/superseded | Decision or Downstream Concerns updated | options revised}
```

**Notes** is append-only (newest last). Record creation, status changes, Decision updates, and material option/tradeoff edits — not wording polish.

**Status meanings:**

| Status | When |
|--------|------|
| `proposed` | Options under review |
| `approved` | User explicitly chose an option |
| `rejected` | User declined all options / abandoned the concern without a successor |
| `superseded` | Replaced by another ADR that reframes the same concern; link the successor in **Related** |

---

## Example (structure only)

Prose below is English because this skill file is English. When saving, write body slots in the user's language — do not copy these sentences verbatim.

File: `{workspace}/adr/application-performance-cache.md`

```markdown
# Application Performance Cache

## Concern
How should we improve response latency for the item catalog API?

## Status
approved

## Context
- Database reads hit high CPU during peak hours.
- Catalog data changes infrequently (a few times a day).

## Decision
**Option A — In-Memory Redis Cache**
Offload read traffic from the primary DB with a dedicated caching layer.

## Options
### Option A — In-Memory Redis Cache
- Standalone Redis in front of the DB.
- Pro: Sub-millisecond reads, high throughput.
- Con: Extra network hop; stale-data risk.

### Option B — Database Read Replicas
- Horizontal scale via replica nodes.
- Pro: Familiar SQL; no app caching logic.
- Con: Higher infra cost; replica lag.

## Tradeoffs
| | Redis Cache | Read Replicas |
|---|-------------|---------------|
| Latency | Very low | Lower than primary, higher than cache |
| App complexity | Cache invalidation needed | Mostly transparent |
| Cost | Modest extra service | Replica nodes |

## Consequences
- Reads may be served from cache; invalidation policy still open.
- Not adopting read replicas for this concern.

## Downstream Concerns
- [ ] **ADR candidate — Cache Eviction & TTL:** How long should catalog data live, and how should manual invalidation work?
- [x] **directly resolved — Cache Size:** Use existing platform memory limit; revisit if this task adds capacity risk.
- [x] **out of scope — Multi-region cache topology:** Outside current task boundary.

## Related
- Follow-up (pending user OK): `{workspace}/adr/cache-eviction-policy.md`

## Notes
- 2026-03-01: Created as proposed with Options A (Redis) and B (read replicas).
- 2026-03-02: Status → approved; Decision set to Option A; Downstream Concerns triaged.
```
