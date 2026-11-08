# Stage ⑤ — Spec Writing (`spec/{requirement-slug}.md`)

**Prerequisites:** User-approved `{workspace}/spec/ASR.md` (Readiness Gate G5 pass).

**Gate artifact:** Draft `{workspace}/spec/{requirement-slug}.md` ready for stage ⑥ review.

**Next stage:** `06-spec-review.md` (formal Spec approval happens there — not in this stage)

---

## Rules

1. **One File, One Scope:** Each distinct component, feature, or independent deliverable gets its own Spec. Before creating one, search `{workspace}/spec/` (filename, title) for the same scope — **merge**, do not duplicate. Do not put Spec content in `PRD.md`, `AC.md`, or `ASR.md`.
2. **Self-contained SSOT draft:** A reader (or stage ⑦) must get every decided requirement and design decision from the Spec alone.
   - Encode **functional** requirements, **non-functional** requirements, and **structurally important design decisions**.
   - Deep-copy decided ADR / Direct Input / ASR outcomes **in full** (parameters, values, behaviors) into Spec bullets.
   - Encode every blocking ASR from `ASR.md` in Decisions, Requirements, or Constraints.
   - **Prohibited:** ADR/ASR paths as stand-ins for requirements (e.g. `- Auth: See adr/auth.md`).

---

## Procedure

1. **Identify scope:** Map the current requirement to `{workspace}/spec/{requirement-slug}.md`. PRD changes belong in stage ① — do not rewrite product scope here.
2. **Resolve ambiguity (Socratic):** Ask when scope, audience, format, or quality bars are unstated. Do not present agent assumptions as Spec facts.
3. **Gather inputs:** Approved `ASR.md` (all blocking ASRs); user goals/non-goals/constraints; approved ADR / Direct Input outcomes (deep-copy only). If resuming after ⑧ failure, pull **Deviations** / **Undocumented ASRs** from `{workspace}/verification/{requirement-slug}.md`.
4. **Write or update:** Bullets over prose. Every `## Requirements` line must be **testable** for stage ⑧. Document `## Out of Scope`. On verification-driven edits, change only affected sections; preserve user edits.
5. **Hand off to review:** Report path and summary; advance to ⑥. Do not start implementation from an unreviewed Spec.

---

## File Template

```markdown
# {Requirement Title}
- {One clear statement — deliverable, audience, success in one or two sentences.}

## Context
- {Why this is needed now}
- {Link to parent {workspace}/spec/PRD.md if applicable}

## Decisions
- {Structurally important design decisions — full parameters deep-copied from ADR / Direct Input / ASR}
- {e.g., Redis Cache layer with 60-second TTL and LRU eviction}

## Requirements
- {Functional must-have: testable bullet}
- {Non-functional must-have: testable bullet}

## Constraints
- {Technical, legal, time, format, or quality constraints}

## Out of Scope
- {Explicit non-goals confirmed by the user}

## Open Questions
- {Deferred item — and whether it blocks implementation}

## Related
- {workspace}/adr/{concern-slug}.md — optional audit trail only
- {workspace}/spec/PRD.md
- {workspace}/spec/ASR.md — registry only; not required reading if Spec is self-contained
```

Omit empty sections except **Requirements**.
