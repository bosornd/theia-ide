---
name: sdd
description: >-
  Spec-driven deliverable pipeline under {workspace}/spec/. Use when defining
  requirements, building from a Spec, or verifying against Specs. Concept chain:
  PRD → AC → (ADR | Direct Input) → ASR → Spec → review → implement → verify
  (→ feedback). Pair with the scrum skill: a Spec-readiness sprint (sprint-00 on
  a new deliverable; the next sprint when requirements are added/changed on an
  existing board) covers Spec + backlog; the following sprint implements. Use
  the adr skill when a concern needs option analysis and user approval.
---

# SDD: Spec-Driven Development Skill

Stage-gated pipeline under `{workspace}`. Load **this file first**, then load the matching `workflow/*.md` before writing that stage’s artifacts. Advance only after the stage’s **explicit user approval** (gate).

## Operating principles

1. **Spec is SSOT** for what to build and verify: `{workspace}/spec/{requirement-slug}.md` (product context: `spec/PRD.md`). Not the chat prompt, not `AC.md`/`ASR.md`, not `adr/`, not `docs/`, not `AGENTS.md`.
2. The **user reviews and approves** the Spec (stage ⑥) before implementation.
3. **Implement and verify** only against the approved Spec (and current sprint Scope when scrum is active).
4. **Changes** → update Spec → **user re-confirms Spec** → then implement (and re-verify). Do not implement against an unconfirmed Spec edit.
5. Spec encodes **functional and non-functional** requirements **and** structurally important design decisions (deep-copied outcomes — not link-only pointers to ADR/ASR).
6. **Work / sprint state** → **scrum** skill. **Decisions that need option review** → **adr** skill (via stage ③). Do not use scrum boards as requirements SSOT, or ADR files as implementation SSOT.

Concrete procedures live only in `workflow/*.md`. This file defines the concept chain, stage map, scrum mapping, and Readiness Gate.

---

## Concepts (pipeline)

```
PRD → AC → (ADR | Direct Input) → ASR → Spec → ⑥ review → ⑦ implement → ⑧ verify → ⑨ if needed
```

| Step | Meaning |
|------|---------|
| **PRD** | Product definition (goals, constraints, success criteria, deliverable form). |
| **AC** | Architectural concerns — structural questions the user must settle before Spec is complete. |
| **ADR \| Direct Input** | Resolve each AC: **adr** skill when trade-offs need a recorded choice; otherwise Direct Input on the AC (straightforward default, clear preference, or fails adr “When to Create an ADR”). |
| **ASR** | Settled architecturally significant list: PRD must-haves + designed AC outcomes (ADR and Direct Input). Skeleton for Spec writing — not SSOT in ⑦⑧. |
| **Spec** | Self-contained requirements + design decisions. **SSOT** for ⑦⑧. |
| **⑥ → ⑧** | User approves Spec → implement → verify. **⑨** remediates defects via Spec-first alignment. |

| Artifact | Path | Role |
|----------|------|------|
| **PRD** | `spec/PRD.md` | What the product is. |
| **AC** | `spec/AC.md` | Living registry of open/settled concerns — **append**; never wholesale replace. PRD must-haves with **no** open design question are **not** ACs (they enter ASR from the PRD). |
| **ADR** | `adr/{concern-slug}.md` | Options + approved choice for one concern. Procedure: **adr** skill. Deep-copy into Spec — not SSOT in ⑦. |
| **Direct Input** | *(on the AC)* | Resolution on the AC without an ADR file. |
| **ASR** | `spec/ASR.md` | Consolidation consumed by Spec writing. Not SSOT in ⑦⑧. |
| **Spec** | `spec/{requirement-slug}.md` | **SSOT** for implementation and verification. |

**Supporting (not requirements SSOT):** `docs/` research; `AGENTS.md` orientation/method (create/update in ⑦); `spec-review/` and `verification/` gate records; scrum boards (`TODO.md`, `scrum/`) for task/sprint state.

```
{workspace}/
├── AGENTS.md
├── TODO.md                      # current sprint (scrum)
├── scrum/                       # product-backlog, sprint-NN, TODO-NN archives
├── spec/PRD.md, AC.md, ASR.md, {requirement-slug}.md
├── spec-review/{requirement-slug}.md
├── adr/{concern-slug}.md
├── docs/
├── verification/{requirement-slug}.md
└── … deliverables
```

ADR and requirement filenames: English kebab-case. `{workspace}` / project slug may use the user’s language. Templates live in each stage’s workflow file.

---

## Stages

| Stage | Workflow | Gate (user approves) |
|-------|----------|----------------------|
| — | *(this file)* | Readiness Gate |
| ① PRD | [`01-prd.md`](workflow/01-prd.md) | `spec/PRD.md` |
| ② AC identification | [`02-ac-identification.md`](workflow/02-ac-identification.md) | `spec/AC.md` |
| ③ AC review (ADR \| Direct Input) | [`03-ac-review.md`](workflow/03-ac-review.md) | `spec/AC.md`, optional `adr/*.md` |
| ④ ASR consolidation | [`04-asr-consolidation.md`](workflow/04-asr-consolidation.md) | `spec/ASR.md` |
| ⑤ Spec writing | [`05-spec-writing.md`](workflow/05-spec-writing.md) | Draft `spec/{requirement-slug}.md` (formal Spec approval is ⑥) |
| ⑥ Spec review | [`06-spec-review.md`](workflow/06-spec-review.md) | `spec-review/{requirement-slug}.md` + **Spec approved as SSOT** |
| ⑦ Implementation | [`07-implementation.md`](workflow/07-implementation.md) | Deliverable(s); `AGENTS.md` updated |
| ⑧ Verification | [`08-verification.md`](workflow/08-verification.md) | `verification/{requirement-slug}.md` |
| ⑨ Feedback | [`09-feedback-loop.md`](workflow/09-feedback-loop.md) | Spec + deliverable **aligned** (not docs alone) |

---

## Scrum mapping

Use the **scrum** skill for every non-trivial deliverable. SDD owns Spec/ADR content; scrum owns board state and when sprints advance.

| Sprint | Scope | Exit |
|--------|-------|------|
| **sprint-00** (new deliverable only) | ① → ⑥ (draft + gate tasks). After Spec is approved **and** the user authorizes implementation planning: define `scrum/product-backlog.md` as small verifiable `PB-*` with Spec refs | User requests **Sprint Boundary** |
| **sprint-01+** (implementation) | ⑦ (and ⑧/⑨ as scoped) for the committed sprint Scope only | Sprint Boundary → next plan |
| **Continuation** (requirement add/change on an existing board) | Do **not** re-Initialize as sprint-00. Continuously number the **next** sprint as **Spec readiness** (needed ①–⑥ for the delta + append `PB-*`), then the **following** sprint as **implementation** of that added backlog | Sprint Boundary between readiness and implementation |

**Rules**

1. **New deliverable:** **Initialize** scrum with **sprint-00**. Seed `TODO.md` from stages ①–⑥ (draft/gate tasks), ending with a boundary task. Do **not** put implementation units on a Spec-readiness sprint.
2. After **G7** (⑥ Spec approved) and explicit **implementation planning** approval: still in the Spec-readiness sprint, split Spec into small `PB-*` (Story / Spec path+section / DoD). Optionally draft the next implementation `sprint-NN.md` Scope; do **not** rewrite live `TODO.md` for that sprint until Boundary.
3. **⑦ starts only after** the Spec-readiness sprint’s Boundary completes (sprint-01+ on first pass; or the sprint after a continuation readiness sprint). Never implement product scope while a Spec-readiness sprint is current.
4. **Requirement add/change on an already-scrum'd deliverable** (new cycle for Spec delta): keep existing `scrum/` history and continuous `sprint-NN` numbering. Do **not** restart sprint-00 or wipe Completed sprints.
   - After the current sprint Boundary (or when the user starts the change cycle): **Plan Next** as a Spec-readiness sprint — only the stages needed for the delta (①–⑥ as applicable), G7, append new `PB-*`; no product ⑦ in that sprint.
   - Plan the **following** sprint to implement (and verify as scoped) only the newly approved backlog.
   - Append `PB-*` IDs; never renumber or reuse. Carry unchanged open stories forward per the scrum skill.
5. Tiny one-shot work may skip scrum **only if** the user says so; default is to use it.

---

## Readiness Gate

Run **once** at session start (or when resuming an open defect). Jump to the first failing gate, then advance **forward**. Do not re-run the full gate until a **new** build request or a **⑨** remediation that needs re-routing. On a **continuation** (requirement add/change), re-evaluate gates for the **delta** only; do not treat missing first-pass sprint-00 as a reason to re-Initialize.

**Before gates:** Ensure `{workspace}/spec/` exists (create `{workspace}` only with user confirmation if missing).

Evaluate **G1 → G7 in order**; stop at the first failure.

| Gate | Pass when | On fail → |
|------|-----------|-----------|
| **G1** | `spec/PRD.md` exists; user approved it | ① |
| **G2** | Blocking ACs registered in `AC.md`; **Dependency Order** set (`none` if no blocking concerns); user approved the list | ② |
| **G3** | Every **blocking** AC is `designed` or `deferred` (none `identified` or `reviewing`) | ③ |
| **G4** | No linked ADR remains `proposed` (Direct Input–only ACs need no ADR) | ③ |
| **G5** | `spec/ASR.md` lists PRD + designed AC outcomes; user approved it | ④ |
| **G6** | In-scope `{requirement-slug}.md` file(s) encode all blocking ASRs (`AC.md`/`ASR.md` are not Specs) | ⑤ |
| **G7** | User approved Spec as SSOT; `spec-review/{requirement-slug}.md` records that approval; no open blocking SSOT/consistency issues | ⑥ |

**All pass →** finish Spec-readiness backlog definition if not done, then Boundary → **⑦** in the next implementation sprint (see **Scrum mapping**).

Artifact **counts are not fixed** — include what fitness for purpose requires (one Spec or several; zero or many ACs).
