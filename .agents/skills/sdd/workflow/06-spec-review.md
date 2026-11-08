# Stage ⑥ — Spec Review (`spec-review/{requirement-slug}.md`)

**Prerequisites:** Spec file(s) written (stage ⑤ complete; Readiness Gate G6 pass).

**Gate artifact:** `{workspace}/spec-review/{requirement-slug}.md` recording that the user **approved the Spec as SSOT**.

**Next stage:** After Spec approval — finish sprint-00 backlog definition if needed, then Sprint Boundary; implementation is `07-implementation.md` in sprint-01+ (see SDD **Scrum mapping**). Do not start ⑦ while sprint-00 is current unless the user explicitly skipped scrum.

---

## Purpose

Final pre-implementation gate: confirm the Spec is fit to be the **sole source of truth** for build and verify. This stage checks **Spec quality and alignment**, not artifact compliance (that is stage ⑧).

**Cross-check inputs**

| Source | What to verify |
|--------|----------------|
| `spec/PRD.md` | Goals, scope, deliverable form, constraints |
| `spec/ASR.md` | Every blocking ASR **Statement** encoded in Spec |
| `spec/AC.md` | Designed ACs reflected via ASR → Spec (no open blocking AC) |
| `adr/{concern-slug}.md` | Approved decisions deep-copied into Spec (not linked-only) |
| `spec/{requirement-slug}.md` | Self-contained functional + non-functional requirements and structural design decisions |

---

## Procedure

### 1. Determine Scope

Clarify which `{requirement-slug}` (or full project) to review. Load `PRD.md`, `ASR.md`, `AC.md`, linked ADRs, and target Spec file(s).

### 2. SSOT Sufficiency Check

Confirm a reader (or stage ⑦) can implement **without** opening ADR/ASR files or guessing:

- Every blocking ASR outcome appears in the Spec (Decisions / Requirements / Constraints).
- No Spec bullets defer to ADR/ASR paths instead of stating decided values.
- `## Requirements` bullets are **testable** — each can become one verification inventory row in stage ⑧.
- `## Out of Scope` is explicit where scope boundaries matter.
- No `## Open Questions` remain that **block** implementation (or each is explicitly deferred with user agreement).

### 3. Pre-Implementation Design Review

Scan for decisions that should be settled **before** implementation but are still missing or thin:

- Structural or organizational choices not covered by any ASR or Spec section.
- Quality bars, format, audience, or output location unstated in PRD or Spec.
- Cross-cutting concerns (error handling, i18n, security, performance) implied but not specified when architecturally significant.

Classify each finding as **blocking** (must resolve before ⑦) or **non-blocking** (document and proceed with user agreement).

### 4. Cross-Document Consistency Check

| Check | Fail signal |
|-------|-------------|
| PRD goal vs Spec scope | Spec delivers something PRD does not call for, or omits a PRD must-have |
| ASR Statement vs Spec | ASR Statement present but Spec lacks the stated outcome |
| AC Resolution vs ASR/Spec | AC `designed` but no matching ASR or Spec encoding |
| ADR Decision vs Spec | Approved ADR outcome missing, contradicted, or only linked |
| Spec vs Spec | Multiple `{requirement-slug}.md` files contradict each other |

Record each inconsistency with source references (file + section).

### 5. Ambiguity & Clarity Scan

Flag vague or untestable Spec language:

- Subjective terms without measurable criteria (*"fast"*, *"user-friendly"*, *"appropriate"*).
- Optional language treated as requirements (*"should"*, *"may"*, *"if possible"*).
- Undefined terms or jargon without Context explanation.
- Requirements that admit multiple incompatible implementations.
- Missing acceptance criteria for complex behaviors.

### 6. Write Spec Review Report

Save to **`{workspace}/spec-review/{requirement-slug}.md`**. If the file exists, read it first to preserve **User Review** notes.

```markdown
# Spec Review: {Requirement Title}

**Spec:** {workspace}/spec/{requirement-slug}.md (aligned with PRD.md)
**Reviewed:** {ISO date}
**Summary:** {N} pass, {N} fail, {N} needs-clarification

## SSOT Sufficiency
| # | Check | Status | Notes |
|---|-------|--------|-------|
| 1 | Blocking ASRs encoded in Spec | pass / fail | ... |
| 2 | No ADR/ASR-link-only bullets | pass / fail | ... |
| 3 | Requirements are testable | pass / fail | ... |

## Pre-Implementation Design Gaps
{Each gap — what is missing, blocking?, recommended action}

## Cross-Document Inconsistencies
{Each mismatch — sources, description, severity}

## Ambiguities & Clarifications Needed
{Each vague item — location, issue, suggested fix or question}

## Recommended Next Steps
- **Pass + user approves Spec →** Complete sprint-00 backlog (`PB-*`) if needed → Sprint Boundary → ⑦ in sprint-01+
- **Fail (Spec gap) →** Re-enter `05-spec-writing.md`, then re-run ⑥
- **Fail (ASR gap) →** Re-enter `04-asr-consolidation.md`, then `05` → `06`
- **Fail (design gap) →** Re-enter `03-ac-review.md` (load **adr** when an ADR is required), then ④ → ⑤ → ⑥
```

### 7. Present & Route

Summarize findings. **Wait for explicit user approval of the Spec as SSOT** before any implementation planning or ⑦.

| Outcome | Action |
|---------|--------|
| All checks pass; user approves Spec | Record approval under **User Review**; then sprint-00 backlog / Boundary per SDD **Scrum mapping**, then **⑦** |
| Spec gaps (wording, missing bullets, ADR not deep-copied) | Re-enter **⑤**; re-run **⑥** after update |
| Missing or wrong ASRs | **④** → **⑤** → **⑥** |
| Missing design decisions | Register or reopen AC → **③** (load **adr** when required) → **④ → ⑤ → ⑥** |
| PRD scope mismatch | Update **① PRD** or Spec with user agreement, then re-run **⑥** |
