# Stage ② — AC Identification (`spec/AC.md`)

**Prerequisites:** Approved `{workspace}/spec/PRD.md` (Readiness Gate G1 pass).

**Gate artifact:** `{workspace}/spec/AC.md` with blocking ACs registered.

**Next stage:** `03-ac-review.md` (after ACs identified and Dependency Order filled)

---

## Procedure

1. **Open or create the living registry** — If `{workspace}/spec/AC.md` exists, **read and update it in place**. On first run, create it from the template below. Never replace, truncate, or rewrite the file so prior ACs are dropped.
2. From the approved PRD (including newly added scope), user context, and **universal AC categories** (below), identify **Architectural Concerns (AC)** — structural questions that need design review before the Spec is complete. Skip concerns already registered and still adequate; reopen or add only when new scope introduces a new structural question.
3. **Append** each new blocking AC as `identified` to **Summary** and **AC Detail** (next unused ID, title, category, statement, why it matters). Preserve every prior row and detail block.
4. **Extend Dependency Order** so newly identified ACs appear in dependency-safe order for review. Keep settled ACs in the record.
5. Present the recommended review path (highlight newly added vs already designed/deferred).

Register every identified AC in `{workspace}/spec/AC.md` — not only in conversation. Incremental PRD updates **append** to the same registry.

**AC vs requirement:** Register an AC only when there is an **open structural design question** the user must decide. PRD must-haves with no meaningful trade-off are **not** ACs — they become ASRs from the PRD in stage ④. Option analysis for ACs that need it is stage ③ via the **adr** skill.

---

## Universal AC Categories (always consider)

| Category | What to surface | Examples |
|----------|-----------------|----------|
| **Structure & organization** | How is content/function arranged? | API layers; document sections; module boundaries |
| **Integration & dependencies** | What must this connect to? | Existing codebase; external APIs; templates |
| **Quality bar (architectural)** | Structural definition of "done" | Test strategy shape; observability model |
| **Constraints (structural)** | Hard limits that force design choice | Single-tenant vs multi-tenant; sync vs async |
| **Process & staging** | Approval gates or iterative delivery shape | Draft-review-publish; feature flags |
| **Deliverable form (when ambiguous)** | Artifact type or format not fixed in PRD | REST vs GraphQL; monolith vs services |

Skip categories already **fully decided** in the PRD with no meaningful trade-off.

For large projects, map foundational concerns first (e.g. Storage Model → API Surface → Client UI), record that under **Dependency Order**, and present one AC at a time.

---

## AC.md Template

**`{workspace}/spec/AC.md` is a single living registry.** Use this template only to **create** a missing file, or as the shape for rows/sections you append.

```markdown
# Architectural Concerns

Living registry for {project-slug}. Status of each AC must stay current.

## Summary

| ID | Title | Category | Status | Related ADRs | ASR (after ④) |
|----|-------|----------|--------|--------------|---------------|
| AC-001 | {short title} | {category} | identified \| reviewing \| designed \| deferred | adr/{slug}.md | ASR-00x |

## Dependency Order (recommended review path)

1. AC-00x → AC-00y → …

## AC Detail

### AC-001 — {Title}

- **Category:** {from universal categories}
- **Status:** identified | reviewing | designed | deferred
- **Statement:** {What structural question must be resolved}
- **Why it matters:** {Impact if left to silent default}
- **Depends on:** AC-00x (optional)
- **Related ADRs:**
  - `{workspace}/adr/{concern-slug}.md` — {proposed | approved}
- **Resolution path:** direct-input | adr | mixed
- **Resolution:** {Concrete design outcome when designed}
- **Notes:** {optional}
```

### Field ownership by stage

| Stage | Fields updated |
|-------|----------------|
| ② (this stage) | **Append** new AC rows; **Statement**, **Why it matters**, **Category**, **Depends on**; extend **Dependency Order** |
| ③ | **Status**, **Related ADRs**, **Resolution path**, **Resolution** |
| ⑨ | Reopen or **append** new ACs (full template fields as needed) |
