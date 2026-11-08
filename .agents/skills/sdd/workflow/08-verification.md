# Stage ⑧ — Verification (`verification/{requirement-slug}.md`)

**Prerequisites:** Implemented artifact(s) exist; governing Spec(s) approved (Readiness Gate G7 pass).

**Gate artifact:** `{workspace}/verification/{requirement-slug}.md`

**Next stage:** Done when all in-scope items pass and the user approves; otherwise `09-feedback-loop.md`.

---

## Authority

| Source | Role |
|--------|------|
| `spec/{requirement-slug}.md` + `spec/PRD.md` | **Pass/fail criterion** — only these |
| `{workspace}/AGENTS.md` **Verification guide** | How to check (commands, fixtures) — **not** requirements |
| Chat / original prompt | Must not override Spec |

**Not criteria:** `adr/`, `AC.md`, `ASR.md`, `docs/`. ADR/ASR-link-only Spec bullets are **Spec gaps** → route via ⑨ to Spec fix + re-confirm, not “pass by ADR.” Undocumented ASRs = significant choices the Spec never stated.

---

## Procedure

### 1. Determine Scope

| Request | Verify against |
|---------|----------------|
| Named file or slug | `PRD.md` + that Spec |
| "All specs" / full project | `PRD.md` (product context) + every `{requirement-slug}.md` under `spec/` (exclude `AC.md`, `ASR.md`) |
| Unspecified | Ask which `{requirement-slug}` and single vs multiple deliverables |
| After ⑨ remediation | **Scoped re-verify** — failed / changed items only; update the existing report |

When scrum is active, further narrow to bullets covered by the current sprint’s committed `PB-*` / implemented slices.

### 2. Load Spec(s) & Artifact(s)

From `PRD.md` and target Spec(s), extract a checklist: Goals / audience / deliverable form (PRD); Decisions, Requirements, Constraints, Out of Scope (Spec).

Read the full implemented deliverable(s) — do not skim. If `AGENTS.md` has a **Verification guide**, follow its method; Spec remains the pass/fail criterion.

Do not edit `AGENTS.md` only because a verification report was written. Update the Verification guide or **Notes** only when the method changed, or a still-true fact a later session must know is missing from `spec/`, `adr/`, `docs/`, and `TODO.md` (same rules as stage ⑦).

### 3. Verify Item by Item

Build an inventory first: **one Spec/PRD checklist bullet = one row**. Then check each row.

| Status | Meaning |
|--------|---------|
| `pass` | Evidence shows the bullet is met |
| `fail` | Required behavior missing or contradicted |
| `partial` | Some but not all of a must-have is met |
| `not-verifiable` | Cannot check from the artifact alone (state what is missing) |

For each row record: Spec item, Status, Evidence (path / section / line), Notes.

Rules:

- Material in the artifact that the Spec puts **Out of Scope** → `fail`.
- When the Spec or project requires automated tests, record run results as evidence.
- Prefer concrete evidence over assertion.

### 4. Design Review & Undocumented ASRs

Beyond inventory compliance, scan once for:

- Structural choices that contradict Spec **Decisions**.
- Maintainability / safety issues material to the PRD/Spec quality bar.
- Architecturally significant choices the Spec **never** addressed (**Undocumented ASRs** only — drop noise).

For each undocumented ASR: what was decided, where in the artifact, category, Spec gap, risk, recommended action. Present to the user — silent choices are not approved.

### 5. Classify Deviations

Consolidate `fail` / `partial` items:

| Severity | When |
|----------|------|
| **Critical** | Violates a must-have Requirement or Decision |
| **Major** | Violates a Constraint, or only partially meets a must-have |
| **Minor** | Cosmetic difference not explicitly forbidden |

### 6. Write Verification Report

Save to **`{workspace}/verification/{requirement-slug}.md`**. If the file exists, read it first to preserve **User Review** notes (and prior rows when doing scoped re-verify).

```markdown
# Verification: {Requirement Title}

**Spec:** {workspace}/spec/{requirement-slug}.md (aligned with PRD.md)
**Artifact(s):** {paths}
**Verified:** {ISO date}
**Summary:** {N} pass, {N} fail, {N} partial, {N} not-verifiable

## Inventory & Results
| # | Spec item | Status | Evidence / Notes |
|---|-----------|--------|------------------|
| 1 | [PRD] High-level Goal | pass | ... |
| 2 | [Spec] Requirement A | fail | Missing implementation |

## Design Review Notes
{Findings from step 4 that are not inventory rows}

## Deviations (Non-compliance)
{each fail/partial — Spec item, actual behavior, severity}

## Undocumented ASRs (Specification Gaps)
{each unstated ASR — decision, location, category, risk, recommended action}

## Recommended Next Steps
- All pass → await user approval (done)
- Failures → see workflow/09-feedback-loop.md
```

### 7. Present & Route

Summarize: counts, critical/major failures, undocumented ASRs. Match the user’s language.

| Outcome | Action |
|---------|--------|
| All in-scope items `pass`; user approves | **Done** for this scope (scrum: update board if needed) |
| Failures / partials; user already asked to fix, or this turn continues an open defect | Load **`09-feedback-loop.md`** and **apply the fix** in this continuation — do not stop after the report |
| Failures; user has not chosen fix / accept / document-only | Present options; wait for the choice |
| Undocumented ASR with an unowned ADR-threshold trade-off | Invite ⑨ route: one ADR via ③, then stop until approved |
| User accepts a deviation as-is | Record acceptance under **User Review** / Deviations; no artifact change |

A written report is the gate record, not the remedy. Closing defects is stage ⑨’s job.
