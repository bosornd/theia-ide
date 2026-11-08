# Workspace exploration

Prefer dedicated tools over shell cat/grep/find/ls for workspace files:
- ~{${GET_WORKSPACE_FILE_LIST_FUNCTION_ID}} — list a directory
- ~{${FILE_CONTENT_FUNCTION_ID}} — read a file
- ~{${FIND_FILES_BY_PATTERN_FUNCTION_ID}} — glob find
- ~{${SEARCH_IN_WORKSPACE_FUNCTION_ID}} — content search
- ~{${UPDATE_CONTEXT_FILES_FUNCTION_ID}} — bookmark important files

## Workspace paths (\`{workspace}/…\`)

All workspace file/directory arguments use **root-prefixed** paths: \`{workspace}/<relativePath>\`.
\`{workspace}\` is the workspace root **folder name** (first path segment), not a literal string and not an absolute OS path.

| Do | Don't |
|----|-------|
| \`{workspace}/src/index.ts\` | \`src/index.ts\` (missing root name) |
| \`{workspace}/docs/\` | \`/home/…/docs/\` or \`file://…\` for normal workspace files |
| Reuse a path exactly as a tool returned it | Invent or strip the root prefix |

Rules:
1. **Discover the root name first** if unknown: call ~{${GET_WORKSPACE_FILE_LIST_FUNCTION_ID}} with \`path\` \`""\` or \`"."\` — keys are root names. Use that name as \`{workspace}\`.
2. **List / read / write / bookmark** take the full \`{workspace}/…\` path (e.g. list \`{workspace}/src\`, read \`{workspace}/AGENTS.md\`).
3. **Glob / content search** patterns are usually root-relative (e.g. \`**/*.ts\`, \`src/**/*.md\`); **results** come back already root-prefixed — pass those strings unchanged into get/write/update tools.
4. Verify by listing; do not guess bare relative paths.

# File create / change (immediate — Agent Mode style)

**Always** read with ~{${FILE_CONTENT_FUNCTION_ID}} before editing a file you have not read in this session.
Use these tools to create or change files on disk immediately (no suggest/change-set review path):
- ~{${WRITE_FILE_REPLACEMENTS_ID}} — targeted replacements (preferred for edits)
- ~{${WRITE_FILE_CONTENT_ID}} — full file write / new files / after replacements fail

Do **not** invent a suggest-only edit path. Do not dump large code blocks in chat when a write tool should apply them.

# Validation & tasks

- ~{${GET_FILE_DIAGNOSTICS_ID}} — syntax/lint/type diagnostics
- ~{${LIST_TASKS_FUNCTION_ID}} — discover workspace tasks
- ~{${RUN_TASK_FUNCTION_ID}} — run lint/build/test tasks when available

Prefer workspace tasks over ad-hoc shell for build/lint/test when a matching task exists.

# Parallel execution

When multiple independent operations are needed, execute them **all in a single response**.
Only run sequentially when there are true dependencies.