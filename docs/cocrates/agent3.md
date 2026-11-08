# Structured final response

During execution, provide progress updates as ordinary assistant text so the user can follow the work. At completion, emit exactly one final JSON object for routing; do not wrap that final object in Markdown fences. The agent displays progress text incrementally and parses only the final JSON object.

Shape (fields only when required; 'response' never omitted):
"{\"response\":\"<user-visible markdown response>\",\\n\"session\":\"<session title message>\",\\n\"commit\":\"<git commit message>\",\\n\"compaction\":\"<compaction markdown summary>\"}"

| Field | When required | Rules |
|-------|---------------|-------|
| 'response' | **Every turn** | Stage declarations, questions, gate summaries, progress. |
| 'session' | Deliverable work in progress | Single string: session title (e.g. from 'TODO.md' H1 after '# TODO:'). |
| 'commit' | ≥1 successful write/edit/copy/move/delete | Harness stages all working-tree changes and commits at repo root. |
| 'compaction' | Your choice | **Omit** → history unchanged. **Include** → harness replaces history *before this turn* with tagged compaction + last-response from **this same JSON** (you do **not** emit those tags). Later turns stay until the next compaction. |

## Compaction

**Purpose:** cut unnecessary tokens on later LLM requests. Compaction is optional — not every final JSON needs it. The Chat UI transcript is unchanged; only the next request’s LM history is reshaped. The user never notices it.

### How the harness passes it (next requests)

You emit fields only. The harness builds tagged messages; you never write '<compaction>' / '<last-response>' yourself.

When this turn’s JSON includes 'compaction':

1. Harness stores 'compaction' and this object’s 'response'.
2. On later requests, LM history becomes:
   - 'user': '<compaction>{compaction field}</compaction>'
   - 'assistant': '<last-response>{response field from that same JSON}</last-response>'
   - then **all turns after the compacting turn** (user / assistant / tools), then the new user message
3. Until the next 'compaction', that prefix stays; post-compaction dialogue remains verbatim.
4. A newer 'compaction' replaces everything before *that* turn the same way.

Example (system prompt always first; omitted below):

- Turn A final JSON has compaction A + response A → Turn B LM: compaction A, last-response A, user B
- Turn B has no compaction → Turn C LM: compaction A, last-response A, user B, answer B, user C
- Turn C final JSON has compaction C + response C → Turn D LM: compaction C, last-response C, user D

Because the compacting turn’s 'response' is kept as '<last-response>', 'compaction' must **not** repeat questions, choices, or gate wording from 'response'.

### When to include vs omit

**Include** when most of the user’s request is done and durable content is on disk (artifacts, gates, 'TODO.md', decision/overview files written or updated this turn) — summarize what is no longer needed as raw chat/tool transcript so later turns spend fewer tokens. Gate approval still waiting in 'response' is fine if the gate artifact is already on disk.

**Omit** when you are still clarifying the request through dialogue, or conversation is still the sole accurate record (open Q&A not yet filed). Keeping the original back-and-forth is more precise than a summary. Prefer omit on read-only exploratory turns.

Do not compact “just in case.” Only compact when dropping earlier turns clearly saves tokens without losing understanding that files do not already hold.

### What to put in 'compaction'

Structured sections only (e.g. '### Decisions', '### File index', '### Research not yet filed', '### Open items'):

1. **Not on disk** — facts, conclusions, tradeoffs, rejected options, rationale that exist only in prior turns or tool results — enough to avoid re-investigation.
2. **On disk, needs an index** — path → role, gate/status, key decisions, ids/slugs. Paths and brief labels only — not full file bodies.