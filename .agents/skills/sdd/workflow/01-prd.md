# Stage ① — PRD (`spec/PRD.md`)

**Prerequisites:** Readiness Gate G1 failed, or new project with no approved PRD.

**Gate artifact:** `{workspace}/spec/PRD.md`

**Next stage:** `02-ac-identification.md` (after user approves PRD)

---

## Procedure

1. **Identify Goal:** Artifact type, target audience, and core function.
2. **Create Project Folder:** Ensure `{workspace}` and `{workspace}/spec/` exist (confirm location/name with user before creating).
3. **Document PRD:** Write or update **`{workspace}/spec/PRD.md`** using the template below.
4. **Obtain PRD Approval:** Present the PRD and get **explicit user approval** before advancing.

---

## PRD Content Guidelines

- State **what** deliverable(s) the project produces and **for whom**.
- Capture global **constraints** and **success criteria** at product level.
- PRD is **product context**, not the per-requirement SSOT. Detailed functional/non-functional requirements and design decisions belong in `{requirement-slug}.md` (stage ⑤).
- Use concise bullets over prose.

---

## PRD.md Template

```markdown
# PRD: {Product / Deliverable Title}

## Summary
{One or two sentences — what this project produces and for whom.}

## Goals
- {Primary goal}
- {Secondary goal}

## Audience
- {Who uses or consumes the deliverable}

## Deliverables
- {Artifact type(s), format, and output location — e.g. Flutter app, markdown report, PNG set}

## Scope
### In Scope
- {High-level capability or deliverable included in this project}

### Out of Scope
- {Explicit non-goals at product level — prevents silent scope creep}

## Constraints
- {Global technical, legal, time, format, or quality limits that apply across all specs}

## Success Criteria
- {How the user will judge the project successful — measurable where possible}
```
