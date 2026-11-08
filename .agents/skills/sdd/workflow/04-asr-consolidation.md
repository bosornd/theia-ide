# Stage ④ — ASR Consolidation (`spec/ASR.md`)

**Prerequisites:** Blocking ACs at `designed` or `deferred` with linked ADRs `approved` (Readiness Gate G3–G4 pass), or no blocking ACs remain.

**Gate artifact:** `{workspace}/spec/ASR.md` — complete ASR registry approved by the user.

**Next stage:** `05-spec-writing.md` (after user approves `ASR.md`)

---

## Procedure

1. **Create or open `ASR.md`** using the template below.
2. **From PRD:** For each goal, constraint, deliverable form, or success criterion that is architecturally significant and not already covered by an AC outcome, register an ASR with **Source:** `PRD` and a testable **Statement**.
3. **From designed ACs:** For each blocking AC at `designed` (ADR **or** Direct Input), register or merge an ASR with **Source:** `AC-00x`, copying **Resolution** into **Statement** / **Decision summary**. Link **Related AC** and **Related ADRs** on the ASR; set the **ASR** back-link on the AC in `AC.md`.
4. **Deduplicate:** One ASR must not contradict another; merge overlapping PRD and AC outcomes.
5. Fill **Dependency Order** for Spec writing (which ASRs must appear in which `{requirement-slug}.md` first).
6. Present `ASR.md` for **explicit user approval** before stage ⑤.

Do not write `{requirement-slug}.md` content here — only the ASR registry.

---

## ASR.md Template

```markdown
# Architecturally Significant Requirements

Consolidated registry for {project-slug}. Approved in stage ④.

## Summary

| ID | Title | Source | Related AC | Related ADRs |
|----|-------|--------|------------|--------------|
| ASR-001 | {short title} | PRD \| AC-00x | AC-00x | adr/{slug}.md |

## Dependency Order (recommended Spec encoding path)

1. ASR-00x → ASR-00y → …

## ASR Detail

### ASR-001 — {Title}

- **Source:** PRD | AC-00x — {section or AC title}
- **Statement:** {Testable architecturally significant requirement}
- **Decision summary:** {If from AC/ADR/Direct Input — plain-language outcome}
- **Depends on:** ASR-00x (optional)
- **Related AC:** AC-00x (optional)
- **Related ADRs:**
  - `{workspace}/adr/{concern-slug}.md` — approved
- **Notes:** {optional}
```
