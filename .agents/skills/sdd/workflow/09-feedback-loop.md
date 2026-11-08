# Stage ⑨ — Feedback Loop (defect remediation)

**Prerequisites:** A defect is in scope — verification `fail`/`partial`, or the user reported a problem. A written ⑧ report is input, not the remedy.

**Gate artifact:** Spec and deliverable **aligned** for the failing items (plus AC / ADR only when the route needs a decision). Status or registry edits alone are not the gate.

**Next stage:** Scoped re-verify via `08-verification.md`. Done when in-scope failures are cleared, or the user accepts a deviation as-is.

**Purpose:** Restore alignment between the governing Spec and the deliverable. Fixing only the artifact while Spec is wrong — or updating only docs while the artifact still fails — leaves the defect open. Route by cause.

**Invariant (matches SDD operating principle 4):** If Spec content changes, **user must re-confirm the Spec** before implementation continues. Do not implement against an unconfirmed Spec edit.

---

## Alignment (hard)

A defect stays **open** until Spec (when it was wrong or incomplete) is confirmed and the deliverable agrees, and scoped ⑧ no longer fails — or the user explicitly accepts the deviation as-is.

| Closes the defect | Does **not** |
|-------------------|--------------|
| Spec correct (and re-confirmed if edited) + artifact changed so the finding no longer holds | Updating only `AC.md`, `ASR.md`, `TODO.md`, verification, sprint notes, or Spec without confirmation / without the artifact |
| Scoped re-verify (⑧) **after** that aligned change | Re-running the Readiness Gate or a status check and stopping |
| | Declaring “re-entering ⑤” and ending the turn |

If Spec is already correct, change only the artifact. If Spec must change: edit Spec → **present the Spec delta and get explicit user confirmation** → then implement → scoped ⑧.

If a prior turn already registered this finding or updated Spec/ASR/AC/verification for it but left the artifact wrong, **do not status-check again** — finish confirmation (if Spec still unconfirmed) and the artifact change now.

Status files (`AC.md`, `TODO.md`, verification) update **after** the aligned change — never instead of it.

---

## Spec re-confirmation

Whenever this stage edits Spec content:

| Change size | Confirmation path |
|-------------|-------------------|
| Wording / missing bullet / deep-copy fix; requirement already agreed | Present Spec **delta**; get **explicit user approval** of the updated Spec; then ⑦. Full ⑥ report optional unless the user asks. |
| New/changed ASR, design decision, or scope | Update AC/ASR as needed → **⑤** → **⑥** (Spec review gate) → then ⑦ |

Never skip user confirmation of a Spec edit before ⑦.

---

## Routing

Pick one route and apply it. Stop only for an unowned ADR, when Spec confirmation is pending, or when the user paused / asked for docs-only.

| Situation | Action |
|-----------|--------|
| Artifact wrong, Spec correct (default when the user says “fix this”) | **⑦** scoped to failed items — do **not** rewrite Spec → scoped **⑧** |
| Spec gap or wrong Spec; correct requirement already agreed or a clear default | Edit Spec → **user confirms Spec** (see **Spec re-confirmation**) → **⑦** → scoped **⑧** |
| Missing ASR / AC decision | **④** and/or **③** (+ ADR if needed) → **⑤** → **⑥** → **⑦** → **⑧** (user approval at gates as usual) |
| Undocumented ASR with a real trade-off the user has not owned | Register AC or ASR → one ADR via **③** (load **adr** skill), then **stop**. After approval: **④ → ⑤ → ⑥ → ⑦ → ⑧** |
| Accept deviation as-is | Record acceptance on the verification report. No artifact change. Stop. |
| User asked to document only, or paused | Docs only. Say so. |

---

## Procedure

1. **Route** from the table. Touch AC/ASR only when that route needs a decision record — not as the work product.
2. **Spec first when Spec is wrong** — edit Spec, then obtain **user confirmation** before any further artifact work for that change.
3. **Implement** — scoped **⑦** against the confirmed Spec.
4. **Re-verify** the failed items only (⑧). Update the verification report to match the new artifact — after the fix, not instead of it.
5. **Then** mark status on files that have status fields (`AC.md`, `TODO.md`). Update `ASR.md` only when the route added or changed ASR **content** (④).
