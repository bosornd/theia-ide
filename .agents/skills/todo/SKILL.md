---
name: todo
description: >-
  Manages deliverable progress with Scrum boards — initialize, track sprint
  tasks in TODO.md, sync with workspace artifacts, and pick the next actionable
  task. Maintains scrum/product-backlog.md (story status), sprint-NN.md (plans),
  and scrum/TODO-NN.md (closed-sprint archives). Use when starting or resuming
  a deliverable session, progress or next-step questions, sprint/backlog
  planning, or alongside any deliverable workflow.
---

# TODO — Deliverable progress Tracking Skill

Tracks deliverable progress on disk so sessions can resume, report status, and pick the next task without relying on chat memory.

| Role | Path |
|------|------|
| **Progress SSOT** | `{workspace}/TODO.md` (current sprint only) |
| **Product backlog** | `scrum/product-backlog.md` |
| **Sprint plans** | `scrum/sprint-NN.md` |
| **Closed boards** | `scrum/TODO-NN.md` |

**Contract:** On-disk boards are authoritative. Do not use the agent’s built-in `todowrite` tool. Companion skills own artifact content; this skill owns task/board state and when to advance.

---

## When to Use

| Situation | Action |
|-----------|--------|
| New deliverable | **Initialize** |
| Resume / mid-session | **Orient** |
| Draft done or gate approved | **Execute** → **Sync** → if Recommended next is a **draft**, continue the companion skill’s next stage in the same turn (do not only recommend and wait) |
| "where are we?" / "what's next?" | **Report** from `TODO.md` (+ backlog if product-wide) |
| Scope change within sprint | **Sync** |
| Backlog / sprint plan needed | **Plan** / **Plan Next Sprint** |
| Boundary task is Current focus / Recommended next | **Sprint completion** → after explicit approval, **Sprint Boundary** |

---

## Scrum Principles

Core loop: small unit → **validate** → repeat.

| Rule | Guidance |
|------|----------|
| **Small units** | Split into units that can be verified and that unlock the next step — not one mega-`PB-*`. |
| **Heavy work on the board** | Refactors, rewrites, migrations get explicit `PB-*` with concrete **Story:** / DoD. |
| **Light sprint load** | Commit a small set of `PB-*` that meets Sprint Goal and Demo Checklist — not “finish the backlog.” |
| **Verify before advancing** | Checkable acceptance (**Story:**, DoD, demo gates). Close PB when verified — at **Sprint Boundary** (or when the user explicitly confirms DoD mid-sprint). |

If a story is too big to demo in one sprint, split into smaller `PB-*` (append IDs; never renumber).

---

## Task Model

### States

| Signal | Meaning |
|--------|---------|
| `- [ ]` | Open |
| `- [x]` | Closed (done, or `_(skipped)_`) |
| Snapshot **Current focus** | At most one in-progress open task |
| `_(blocked: …)_` | Open but blocked |
| `_(skipped)_` | Closed without doing the work |

Use checkboxes + Current focus only — no `pending` / `in_progress` / `done` tokens on headlines. Same signals apply to **PB** stories on the backlog.

| Kind | Form | Stability |
|------|------|-----------|
| Task | `T-{nnn}` | Stable within one board; never renumber/reuse. May restart at `T-001` when `TODO.md` is rewritten for a new sprint. |
| Story | `PB-{nnn}` | Stable for the deliverable; never renumber/reuse. Append only. |

### Task entry

```text
- [ ] **T-00n** — {short title}
  - Phase: {phase or "cross-cutting"}
  - Artifact: `{path}` or —
  - Depends: T-00m, T-00k
  - Notes: {optional; may note type: draft | gate | unit | research | sync | boundary}
```

| Type | Purpose |
|------|---------|
| `draft` | Write/generate an artifact |
| `gate` | Explicit user approval before proceeding |
| `unit` | Atomic slice within a phase |
| `research` | Investigation; may run **in parallel** while a gate waits **only if the user asks** (or the companion skill requires it) — does not close or bypass the gate |
| `sync` | Keep artifacts consistent |
| `boundary` | Sprint close gate — always last; see **Boundary task** |

**Story → tasks:** Each scoped `PB-*` becomes one or more `T-*` (drafts, units, gates). Prefer Notes or title to show the parent (e.g. `PB-012`). Closing all `T-*` for a story does **not** auto-close the PB — close PB at **Sprint Boundary** (or when the user explicitly confirms DoD).

### Boundary task

Whenever live `TODO.md` is seeded or rewritten for a sprint (**Initialize**, **Plan** activate, **Sprint Boundary** → next board), append exactly one final task:

```text
- [ ] **T-00n** — Sprint Boundary → sprint-{NN+1}
  - Phase: cross-cutting
  - Artifact: `scrum/TODO-{NN}.md`
  - Depends: {prefer: every other planned sprint T-*; minimum: the last scope/demo gate}
  - Notes: type: boundary — archive board, update backlog, load next sprint
```

| Rule | Guidance |
|------|----------|
| **Always last** | No `T-*` after it. Mid-sprint user additions go before it; extend Depends. |
| **Depends** | Prefer listing every other planned `T-*`. If the list is huge, Depends on the last demo/DoD gate is enough — but then Sync must not leave earlier open tasks when that gate closes. |
| **When selected** | Apply **Sprint completion** → after approval, **Sprint Boundary**; mark `[x]` on the **archived** copy. |
| **Final sprint** | No further backlog: title `Sprint Boundary → complete`. Archive + close backlog; do not invent a next sprint. |
| **Recovery** | If a live board has no open tasks and no boundary task (legacy/broken seed), treat as end-of-sprint: **Sprint completion**, then Boundary. Re-seed always includes a boundary task. |

---

## TODO.md Structure

```markdown
# TODO: {Title}

> **Project type:** `{type}`
> **Project root:** `{workspace}`
> **Updated:** {YYYY-MM-DD}
> **Current sprint:** sprint-{NN} — [{goal}](./scrum/sprint-{NN}.md)
> **Product Backlog:** [`scrum/product-backlog.md`](./scrum/product-backlog.md)

## Snapshot

| Done | Open | Blocked | Skipped |
|------|------|---------|---------|
| {n}  | {n}  | {n}     | {n}     |

**Sprint goal:** {one line}
**Current focus:** {T-id or "—"}
**Recommended next:** {T-id} — {one-line description}

## Tasks

{task entries — current sprint only}

## Completed sprints   # keep across rewrites — never wipe

| Sprint | Goal | Closed | Archive |
|--------|------|--------|---------|
| 00 | {one-liner} | {YYYY-MM-DD} | [`TODO-00.md`](./scrum/TODO-00.md) |

## Notes
…
```

---

## Plan File Templates (`scrum/`)

### product-backlog.md

```markdown
# Product Backlog — {Title}

> **Updated:** {YYYY-MM-DD}
> **Execution board:** [`../TODO.md`](../TODO.md)

## Stories

- [ ] **PB-001** — {short title}
  - Story: {concrete Done-when / who-wants-what}
  - Spec: `{spec/….md}` §{section}   # or —
  - Sprint: {sprint-NN or —}
  - Depends: —
  - Notes: …
```

| Field | Rule |
|-------|------|
| **Story** | Required. Acceptance-oriented; not a vague label. |
| **Spec** | Spec-driven: required path + section once Specs exist (extra refs in Notes). Series: `—`. Discovery: `—` until known. |
| **Sprint** | Set when committed to a sprint; keep after done for history. |
| **Depends** | Other `PB-*` IDs, or `—`. |

**Series layout:** `PB-000` (skeleton / cross-cutting) + one story per publication unit. **Spec-driven:** epic grouping only in Notes if useful; prefer vertical slices.

**Carry-over:** Story misses DoD → leave `[ ]`, set **Sprint** to the next sprint (or `—` until replanned), note carry-over in both boards' Notes. Done at Sprint Boundary step 2.

### sprint-NN.md

Zero-pad (`sprint-00.md`, …). Plan only — do not edit Goal/DoD/Demo during execution to “mark done.”

```markdown
# Sprint {NN} — {short goal title}

> **Product Backlog:** [`product-backlog.md`](./product-backlog.md)
> **Execution board:** [`../TODO.md`](../TODO.md)

## Sprint Goal
{1–3 sentences}

## Scope

| ID | Story |
|----|-------|
| PB-xxx | {short title — match backlog headline} |

**Out of sprint:** {next slice or —}

## Definition of Done
1. …
2. …

## Demo Checklist
- …   # mirror as gate tasks on TODO.md
```

Scope lists committed `PB-*` only. Seed `TODO.md` from Scope + companion workflow, ending with a boundary task — do not maintain a second story table on the plan file.

### TODO-NN.md

Exact copy of `TODO.md` at Sprint Boundary for sprint-NN. Immutable after a successful copy. Linked from **Completed sprints** → Archive.

---

## Core Workflows

### 1. Initialize

1. Resolve `{workspace}` and companion workflow.
2. Choose profile → create `scrum/` + backlog in small verifiable units (open `[ ]`, Story/Spec rules) + first `sprint-NN.md` with a light Scope → seed `TODO.md` from Scope ending with a boundary task (empty Completed sprints) → set **Sprint:** on scoped PB items only.
3. Current focus = first open task; **Report**.

### 2. Orient

1. Read `TODO.md` (missing → Initialize).
2. Optionally skim current `sprint-NN.md` Goal/DoD; read backlog when product-wide status matters.
3. Drift vs disk → Sync.
4. **Report**.

### 3. Execute

1. Exactly one Current focus. **Recommended next** = Depends satisfied + gate over draft + ID order within phase (`research` only while a gate waits per Task type); keep Current focus if still open and unblocked.
2. Draft done → `[x]`; move to gate or next.
3. Gate → `[x]` only with explicit user approval (batch: flip matching headlines in place).
4. **After a gate `[x]`:** if Recommended next is a **draft** for the companion skill’s next stage, do not stop after Sync/Report — follow that stage and produce the next gate artifact in the same turn (unless the user asked to pause or approved only part of the draft).
5. Refresh Snapshot + `Updated` → Sync.
6. If Current focus / Recommended next is the boundary task → **Sprint completion**.

### 4. Sync

1. Compare disk artifacts to current `## Tasks`.
2. Align with **committed sprint scope** only: drop removed scope with `[x]` + `_(skipped)_`.
3. Append or reshape `T-*` only when (a) the user explicitly requested it, or (b) Initialize / Plan / Sprint Boundary seeded the board from Scope. Keep the boundary task last; update its Depends if earlier tasks change. (Filler tasks: forbidden — see **Sprint completion** §2–3.)
4. Recompute Snapshot.
5. Future episodes / unscoped work stay off this board until after Boundary.

### 5. Plan / Plan Next Sprint

**Plan:** Companion artifacts → backlog stories in small units → `sprint-NN.md` Scope (light committed set) → set PB **Sprint:** → seed/switch `TODO.md` only when that sprint is active — always end with a boundary task.

**Plan Next:** If next `sprint-NN.md` is missing (or user pre-plans): pick open/carried Stories → create the plan file. Do not rewrite live `TODO.md` until that sprint starts (unless Initialize / Boundary).

### 6. Sprint Boundary

**When:** After **Sprint completion** approval only.

1. Archive: copy `TODO.md` → `scrum/TODO-{NN}.md` (zero-padded NN); mark the boundary task `[x]` on the archived copy.
2. **Backlog:** Scope story met DoD → `[x]`; intentionally dropped → `[x]` + `_(skipped)_`; incomplete → carry-over. Bump backlog **Updated:**.
3. Append **Completed sprints** row (Goal, Closed date, Archive link).
4. Next `sprint-*.md` **exists:** rewrite live `TODO.md` (tasks, Snapshot, Current sprint) from that plan with a new trailing boundary task; **keep** Completed sprints; do not leave the closed task list on the live board.
5. Next plan **missing:** Plan Next Sprint, then rewrite as above.

After Boundary, Current focus / Recommended next come from the new sprint’s seeded tasks.

### 7. Report Progress

```markdown
### Progress — {Title}

- **Sprint:** {sprint-NN} — {goal}
- **Phase:** {phase}
- **Done:** {done}/{total} ({percent}%)
- **Current focus:** {T-id} — {title}
- **Recommended next:** {T-id} — {title} ({why})
- **Backlog:** {done}/{total} stories closed   # when product-wide status matters
```
