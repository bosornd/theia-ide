# Stage ③ — AC Review (`spec/AC.md`)

**Prerequisites:** ACs identified in `AC.md` with Dependency Order (Readiness Gate G2 pass).

**Gate artifacts:** Updated `{workspace}/spec/AC.md`; approved `{workspace}/adr/{concern-slug}.md` when the ADR route is used.

**Next stage:** `04-asr-consolidation.md` (when all blocking ACs are `designed` or `deferred`, and no linked ADR remains `proposed`)

This stage owns **AC resolution** in `AC.md`. It does **not** define how to write an ADR. When the **ADR Analysis** route applies, load the **adr** skill and follow it. Do not restate or improvise that procedure.

---

## Procedure

Review blocking ACs in **Dependency Order**, one AC (or dependency branch) at a time:

1. Set the active AC to `reviewing`.
2. Route via **AC Decision Pathways** (below).
3. Record resolutions in `AC.md` as they settle — do not batch every AC before writing them down.

When all blocking ACs have a resolution path complete:

- If any Related ADR is still `proposed`: **pause for user ADR approval** via the **adr** skill. Do **not** invent approval.
- When all linked ADRs are `approved` (or the AC used Direct Input only): set the AC to `designed` with a concrete **Resolution** (from ADR **Decision** or Direct Input).

Status transitions: `identified` → `reviewing` → `designed`. Use `deferred` only with explicit user agreement.

---

## AC Decision Pathways

Do not force an ADR for every concern. Apply the **adr** skill filters under **When to Create an ADR** before opening a file.

| Route | When | Action |
|-------|------|--------|
| **Direct Input** | Straightforward problem, natural default, or clear user preference; or the concern fails the adr filters | Ask direct questions; user chooses; set AC to `designed`; fill **Resolution**. Record the disqualification label from the adr skill (`out of scope`, `directly resolved`, `covered`, `deferred assumption`) in **Resolution** or **Notes** — no new ADR file |
| **ADR Analysis** | User unsure, or complex trade-offs; concern passes the adr filters | Set AC to `reviewing`; run the **adr** skill for this concern; link ADR(s) per **ADR ↔ AC links** |

---

## ADR ↔ AC links

When using the ADR Analysis route:

1. Create or update `{workspace}/adr/{concern-slug}.md` via the **adr** skill.
2. On the AC: set **Resolution path** to `adr` (or `mixed` if Direct Input plus ADR), and list the ADR under **Related ADRs** with its status.
3. After the user **approves** the ADR: copy the ADR **Decision** into AC **Resolution** (full outcome text — not a path-only pointer). Set AC **Status** to `designed`.
4. If the ADR is `rejected` or abandoned: keep or revert the AC to `reviewing`/`identified` and choose another route. Never leave an AC `designed` without a concrete **Resolution**.

One AC may link multiple ADRs when follow-ups appear (adr skill chaining). The AC stays `reviewing` until every **blocking** linked ADR is `approved`, or the remaining follow-up is `deferred` with user agreement.

---

## Gate

Present updated `AC.md` (and any new/updated ADRs) for approval of the resolution set. Advance to ④ only when Readiness Gate **G3** and **G4** would pass.
