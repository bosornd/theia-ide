# Stage ⑦ — Implementation (artifact production)

**Prerequisites:** Readiness Gate **G7** pass (Spec approved in stage ⑥). Scrum: current sprint is **sprint-01+** (sprint-00 Boundary done), unless the user explicitly skips scrum for a tiny one-shot.

**Gate artifacts:** Deliverable(s) at the path(s) named in Spec/PRD (or agreed with the user when Spec names only a form); `{workspace}/AGENTS.md` created or updated in the same turn.

**Next stage:** `08-verification.md`

---

## Authority

| Source | Role |
|--------|------|
| `spec/{requirement-slug}.md` + `spec/PRD.md` | **SSOT** — what to build |
| Live sprint Scope (`TODO.md` / `sprint-NN.md`) when scrum is active | **Which** Spec bullets / `PB-*` to implement now |
| Existing codebase + project conventions | Placement and style when Spec is silent |
| `AGENTS.md` | Orientation and method — **not** requirements |
| Chat / original prompt | Must not override Spec; if conflict, ask which source wins |

**Not inputs for requirements:** `docs/`, `adr/`, `AC.md`, `ASR.md` (already deep-copied into Spec). Do not open them as a substitute for Spec content.

---

## Procedure

### 1. Scope & load Spec(s)

| Request | Load |
|---------|------|
| Named slug | `PRD.md` + that `{requirement-slug}.md` |
| "All specs" / full project | `PRD.md` + every in-scope `{requirement-slug}.md` |
| Unspecified | Ask which deliverable set, then load it |

When scrum is active, **narrow** to bullets referenced by the current sprint’s committed `PB-*` / tasks — do not implement Spec scope outside that sprint.

Keep in working memory: Requirement title/statement, Decisions, Requirements, Constraints, Out of Scope. Spec overrides conversation history.

### 2. Scrum check

Load the **scrum** skill. Implement **only** the live board’s committed Scope.

- Skip the board only when the user explicitly allows a tiny one-shot without scrum.
- Never implement product scope while sprint-00 is current (SDD skill **Scrum mapping**).

### 3. Implement

Produce artifacts **strictly from the Spec** (and current sprint Scope when scrum is active):

1. Locate existing code and conventions before writing; match neighboring modules rather than inventing a new layout.
2. Honor every **in-scope** Requirements bullet; respect Decisions, Constraints, and Out of Scope. Add nothing the Spec does not call for.
3. When Spec is silent on style or placement, follow the repository’s patterns and `AGENTS.md` Implementation guide.
4. Write to path(s) Spec/PRD names. If Spec only describes form (e.g. “HTTP API”) and not a path, agree the path with the user before creating new top-level trees.
5. Add or update automated tests when the Spec or project conventions require them for the slices you touch.

### 4. Maintain `AGENTS.md`

In the **same turn** as the implementation, create or update `{workspace}/AGENTS.md`. Read it first; keep sections and notes that are still true. A later session should orient from this file before walking the tree.

| Section | Contents | Update when |
|---------|----------|-------------|
| **Repository structure** | One short paragraph on what the repo is; tree of **major folders only** with one-line roles; one or two sentences on how those folders depend. No file lists inside folders. | A major folder is added, removed, or its role changes |
| **Working principles** | Standing rules (skills to follow, where research goes, what belongs in notes) | A standing rule changes |
| **Implementation guide** | Module design/structure, then steps/commands/paths/conventions to build or change the deliverable | Module design, structure, or build method changes |
| **Verification guide** | How to check against the Spec (commands, fixtures, evidence) so stage ⑧ can follow it. Pass/fail record stays in `verification/`. | The verification method changes |
| **Notes** | Facts learned while implementing that are not already in `spec/`, `adr/`, `docs/`, or `TODO.md`, and that a later session must know | Such a fact appears or becomes false |

Do **not** edit `AGENTS.md` only because a file inside `spec/`, `adr/`, or `verification/` changed. Research write-ups go in `docs/` (a note may point at a doc). Write headings and body in the user’s language (or the language already used in that file). If the file is new or sections are missing, add them without wiping unrelated existing sections:

~~~markdown
## Repository structure

{One paragraph: what this repository is}

```text
{workspace}/
├── {major-dir}/    # {one-line role}
├── spec/           # requirements and feature specs
├── adr/            # architecture decisions
├── docs/           # research
└── verification/   # verification records against the Spec
```

{How major folders depend on each other. Do not list files.}

## Working principles

1. {Standing rule for this repository}

## Implementation guide

### Module design and structure

{How modules are split, how they depend, and where they live}

### How to implement

{Steps, commands, paths, and conventions to create or change the deliverable}

## Verification guide

{How to check the deliverable against the Spec}

## Notes

- {Fact a later session must know}
~~~

### 5. Blockers

| Situation | Action |
|-----------|--------|
| Spec ambiguous, contradictory, or missing a needed Decision | **Pause** artifact edits that would encode a guess. Ask the user; agree a Spec update, then **user re-confirms Spec** (⑥ or scoped Spec approval per ⑨) before continuing |
| Environment/tooling blocks progress | Document briefly; if it changes how to implement or verify, record a still-true note in `AGENTS.md` |
| Defect after ⑧, or bug mid-build | Load `09-feedback-loop.md` and follow its routing — do not invent a parallel process |

### 6. Pre-handoff check

Lightweight self-check **before** recommending ⑧ — not a substitute for formal verification:

1. Spot-check that in-scope Requirements / Decisions / Constraints have corresponding evidence in the artifact; fix obvious gaps.
2. Run automated tests the Spec or project defines for the touched slices; fix failures you introduced.
3. Confirm structure/behavior do not contradict Spec Decisions, and Out of Scope does not appear in the artifact.
4. If this check changes the build or verify method, update the matching guide in `AGENTS.md`.

### 7. Report

State deliverable paths (and `AGENTS.md` if changed); brief Requirements checklist (done / deferred to a later sprint); recommend stage ⑧. If scrum: note sprint progress / next sprint when the board closes.
