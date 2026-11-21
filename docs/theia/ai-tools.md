# Theia AI: Built-in Tools와 에이전트 도구 제공

조사 기준: 형제 저장소 `../theia/` (`@theia/ai-core`, `@theia/ai-chat`, `@theia/ai-ide`, `@theia/ai-terminal`, `@theia/ai-mcp` 등).  
프레임워크 버전 참고: applications가 끌어오는 `@theia/ai-*` (예: 1.76.0).  
관련: [adding-ai-agents.md](./adding-ai-agents.md) (agent·프롬프트·mode), [shipping-ai-skills.md](./shipping-ai-skills.md) (skills).

---

## 한눈에 보기

| 질문 | 답 |
|------|----|
| Tool이 어디에 등록되나 | `ToolProvider` → `bindToolProvider(...)` → `ToolInvocationRegistry` |
| Agent에 도구를 주는 방법 | **system prompt(또는 포함 fragment)에 `~{toolId}`** 를 넣는다 |
| `Agent.functions` 배열 | AI Configuration **문서용**. 프레임워크가 자동으로 tool을 넘기지 않음 |
| 프롬프트에서 `~{id}`가 하는 일 | (1) 본문을 tool id 문자열로 치환 (2) `ToolRequest`를 LLM tools 목록에 추가 |
| Deferred tool | `~{?toolId}` — 선로드하지 않고 provider tool-search로 on-demand |
| MCP | 서버 기동 시 `mcp_<server>_<tool>` 등록 + `{{prompt:mcp_<server>_tools}}` 로 노출 |

**핵심:** Registry에 등록만 해서는 agent가 쓰지 못한다. 해당 agent의 resolve된 prompt에 `~{toolId}`(또는 MCP/capability fragment가 그 참조를 포함)가 있어야 LLM에 tool schema가 전달된다.

---

## 1. 도구 등록 메커니즘

### 1.1 `ToolProvider` / Registry

```typescript
// packages/ai-core/src/common/tool-invocation-registry.ts
export interface ToolProvider {
    getTool(): ToolRequest;  // id, name, description, parameters, handler
}
bindToolProvider(MyTool, bind);  // DI + ToolProvider contribution
```

기동 시 `ToolInvocationRegistryImpl`이 모든 `ToolProvider` contribution의 `getTool()`을 등록한다.  
AI Configuration → Tools에서 전체 목록을 볼 수 있다.

### 1.2 Chat 요청까지의 흐름

1. Agent `invoke` → `PromptService.getResolvedPromptFragment(systemPromptId)`  
2. 템플릿의 `{{variables}}` 치환 후 `~{…}` 매칭 (`PROMPT_FUNCTION_REGEX`)  
3. Registry에서 `ToolRequest`를 찾아  
   - 본문: `toolRequestToPromptText` → **tool id 문자열**로 치환 (`~{getFileContent}` → `getFileContent`)  
   - `functionDescriptions: Map<id, ToolRequest>` 수집  
4. `AbstractChatAgent`가 system message tools + 사용자 메시지 `~fn` + `additionalToolRequests`를 합쳐 LM에 전달  

소스: `prompt-service.ts` (`getResolvedPromptFragment`), `chat-agents.ts` (`systemMessageDescription.functionDescriptions`).

### 1.3 프롬프트 문법 (도구 관련)

| 문법 | 의미 |
|------|------|
| `~{toolId}` | 해당 tool을 agent에 노출 + 본문에 id 표시 |
| `~{?toolId}` | deferred — 선로드하지 않음 (Anthropic/OpenAI tool search 등) |
| `{{capability:fragment-id default on\|off}}` | 켜지면 fragment 삽입; fragment 안의 `~{…}`도 도구로 합쳐짐 |
| `{{prompt:fragmentId}}` | 다른 fragment 인라인 (예: `mcp_github_tools`, `open-editors-hint`) |
| `{{skills}}` | skill 목록(XML). 본문 로드는 별도 `~{getSkillFileContent}` |

채팅 입력의 `~fn` / `#var` / `@agent`는 **사용자 메시지** 파서 경로다. system prompt 안에서는 `~{…}` / `{{…}}` 가 resolve 대상이다.

### 1.4 `Agent.functions`에 대한 주의

`Agent.functions: string[]`는 “이 agent가 항상 쓸 수 있는 함수”를 Configuration UI에 보여 주기 위한 메타데이터다.  
주석상 **Referenced functions are NOT automatically handed over** — 실제 노출은 prompt의 `~{…}` 또는 agent 구현의 `additionalToolRequests`로 해야 한다.

---

## 2. Built-in tool 카탈로그

패키지별로 Registry에 올라가는 도구. ID는 LLM function name과 동일하다.

### 2.1 워크스페이스 탐색·읽기 (`@theia/ai-ide`)

상수: `packages/ai-ide/src/common/workspace-functions.ts`

| Tool ID | 역할 |
|---------|------|
| `getWorkspaceDirectoryStructure` | 디렉터리 트리(디렉터리만, 파일 제외) |
| `getWorkspaceFileList` | 특정 디렉터리의 즉시 자식 목록 |
| `getFileContent` | 파일 본문 읽기 |
| `findFilesByPattern` | glob으로 파일 찾기 (`**/*.ts` 등) |
| `searchInWorkspace` | 워크스페이스 텍스트/패턴 검색 |
| `getFileDiagnostics` | 파일 진단(lint/type 등) |
| `getSkillFileContent` | skill `SKILL.md`(및 확장 path) 본문 로드 |

### 2.2 파일 변경 — suggest vs write (`@theia/ai-ide`)

상수: `packages/ai-ide/src/common/file-changeset-function-ids.ts`

| Tool ID | 적용 방식 |
|---------|-----------|
| `suggestFileContent` | 전체 내용 **제안**(changeset, 사용자 수락) |
| `writeFileContent` | 전체 내용 **즉시 디스크 적용** |
| `suggestFileReplacements` | 구간 치환 **제안** (V2, 기본) |
| `writeFileReplacements` | 구간 치환 **즉시 적용** (V2, 기본) |
| `getProposedFileState` | 제안 중인 파일 상태 조회 |
| `clearFileChanges` | 해당 파일 대기 중 변경 초기화 |

레거시/단순 변형(프롬프트에서 보통 안 씀, Registry에는 등록됨):

| Tool ID | 비고 |
|---------|------|
| `suggestFileReplacements_Simple` / `writeFileReplacements_Simple` | V1 replacer, deprecated |
| `simpleSuggestFileReplacements` / `simpleWriteFileReplacements` | 단일 치환 헬퍼 변형 |

### 2.3 Task / Launch (`@theia/ai-ide`)

| Tool ID | 역할 |
|---------|------|
| `listTasks` | 워크스페이스 태스크 목록 |
| `runTask` | 태스크 실행(완료까지 블로킹) |
| `listLaunchConfigurations` | launch.json 설정·실행 상태 |
| `runLaunchConfiguration` | 런치 시작(백그라운드) |
| `stopLaunchConfiguration` | 실행 중 런치 중지 |

### 2.4 Chat context (`@theia/ai-ide`)

상수: `packages/ai-ide/src/common/context-functions.ts`

| Tool ID | 역할 |
|---------|------|
| `context_addFile` | 세션 컨텍스트에 파일 북마크 |
| `context_ListChatContext` | 채팅 컨텍스트 목록 |
| `context_ResolveChatContext` | 컨텍스트 항목 resolve |

프롬프트에서는 보통 `context_addFile`을 “updateContextFiles” 의미로 안내한다.

### 2.5 Task context (계획 문서) (`@theia/ai-ide`)

상수: `packages/ai-ide/src/common/task-context-function-ids.ts`

| Tool ID | 역할 |
|---------|------|
| `createTaskContext` | 구현/리뷰 계획 생성(에디터 오픈) |
| `getTaskContext` | 계획 읽기 |
| `editTaskContext` | 구간 편집 |
| `rewriteTaskContext` | 전체 재작성(폴백) |
| `listTaskContexts` | 세션의 계획 목록 |

### 2.6 협업·상태·기타 IDE (`@theia/ai-ide`)

| Tool ID | 역할 |
|---------|------|
| `todoWrite` | 사용자에게 보이는 todo 목록 |
| `userInteraction` | 다단계 위저드/선택 UI |
| `suggestTerminalCommand` | 터미널에 실행할 명령 **제안** |
| `getGitChanges` | git 변경/diff (Commit Message 등; PredefinedShell 계열) |

### 2.7 App Tester 브라우저 (`@theia/ai-ide`)

상수: `packages/ai-ide/src/common/app-tester-chat-functions.ts`

| Tool ID | 역할 |
|---------|------|
| `launchBrowser` | 테스트용 브라우저 기동 |
| `isBrowserRunning` | 브라우저 연결 상태 |
| `closeBrowser` | 브라우저 종료 |
| `queryDom` | DOM 질의(명시 요청 시) |

AppTester는 여기에 더해 MCP(Chrome DevTools 등) tool fragment를 `{{prompt:mcp_…_tools}}`로 붙인다.

### 2.8 Agent 위임 (`@theia/ai-chat`)

| Tool ID | 역할 |
|---------|------|
| `delegateToAgent` | 다른 ChatAgent에 서브태스크 위임 (`agentId` + `prompt`) |

상수: `packages/ai-core/src/common/tool-constants.ts` → `AGENT_DELEGATION_FUNCTION_ID`.

### 2.9 Shell (`@theia/ai-terminal`)

| Tool ID | 역할 |
|---------|------|
| `shellExecute` | 임의 셸 명령 실행(사용자 승인·allow/deny 목록). 출력 trunc |

Coder 등은 기본 프롬프트에 직접 넣지 않고 `{{capability:shell-execution default off}}`로 옵션화한다.  
capability fragment(`shell-execution`)가 켜지면 그 안에 `~{shellExecute}`(+ task 우선 안내)가 들어간다.

`PredefinedShellTool` 기반 도구(예: `getGitChanges`)는 고정 명령만 실행하며 `shellExecute` allow-list와 별개다.

### 2.10 MCP (`@theia/ai-mcp`)

서버 연결 시 동적으로:

- Tool ID: `mcp_<serverName>_<toolName>`
- Prompt fragment: `mcp_<serverName>_tools` (내부에서 `~{mcp_…}` 또는 deferred `~{?…}` 나열)
- 에이전트 노출: 프롬프트에 `{{prompt:mcp_<server>_tools}}` 또는 Capabilities UI에서 MCP function 선택

---

## 3. 대표 에이전트별 제공 도구

아래는 **기본 system prompt에 `~{…}`로 박혀 있는** 도구다.  
Capabilities/UI에서 사용자가 켠 shell·MCP·skills·추가 functions는 세션마다 더해질 수 있다.

### 3.1 Coder (`coder-agent` / `coder-replace-prompt-template.ts`)

Mode에 따라 **편집 API만** 갈린다.

| 영역 | Edit Mode | Agent Mode / Agent Mode (Next) |
|------|-----------|--------------------------------|
| 탐색 | `getWorkspaceFileList`, `getFileContent`, `findFilesByPattern`, `searchInWorkspace`, `context_addFile` | 동일 |
| 편집 | `getProposedFileState`, `suggestFileReplacements`, `suggestFileContent`, `clearFileChanges` | `writeFileReplacements`, `writeFileContent` |
| 검증 | `getFileDiagnostics`, `listTasks`, `runTask` | 동일 + launch (`list`/`run`/`stop`LaunchConfiguration) |
| 진행 | — | `todoWrite` |
| 계획 | — | Agent Mode Next: `getTaskContext`, `delegateToAgent`(Architect/Explore) |
| Capability (default off) | — | `shell-execution`, `github`, `apptester` (+ Next: `code-review-mode`, `memory`) |

Coder 기본 프롬프트는 `{{skills}}` / `getSkillFileContent`를 **넣지 않는다**. Skill이 필요하면 Explore처럼 추가하거나 Capabilities의skills를 쓴다.

### 3.2 Architect (`architect-prompt-template.ts`)

| Mode | 도구 요지 |
|------|-----------|
| Plan | 탐색 4종 + task context CRUD(`create`/`get`/`edit`/`rewrite`/`list`) |
| Simple | 탐색 위주(계획 도구 최소화) |
| Plan (Next) | 파일 직접 읽기 **없음** → `delegateToAgent`→Explore + task context CRUD |

### 3.3 Explore (`explore-prompt-template.ts`)

- Primary: `getWorkspaceDirectoryStructure`, `getWorkspaceFileList`, `getFileContent`, `searchInWorkspace`, `findFilesByPattern`
- Task: `listTaskContexts`, `getTaskContext`, `listTasks`, `runTask`
- Skills: `{{skills}}`, `~{getSkillFileContent}`
- MCP: `{{prompt:mcp_github_tools}}`
- `shellExecute`는 “Primary로 안 될 때만” 언급(탐색에 cat/grep/find/ls **금지**)

### 3.4 CreateSkill

탐색(`getWorkspaceFileList`, `getFileContent`, `findFilesByPattern`, `context_addFile`) +  
mode에 따라 suggest* 또는 write* 로 skill 파일 작성.

### 3.5 ProjectInfo

탐색 + `context_addFile` + suggest* (`suggestFileContent` / `suggestFileReplacements` / `getProposedFileState` / `clearFileChanges`)로 project-info 템플릿 갱신 제안.

### 3.6 AppTester

브라우저 4종 + launch 3종 + `getFileContent` + (Next 등) `getTaskContext`/`editTaskContext` + **필수 MCP** `{{prompt:mcp_…_tools}}`.

### 3.7 CodeReviewer / PRReview

- CodeReviewer: `getTaskContext`, `editTaskContext`, `getFileContent`, `getFileDiagnostics`, `searchInWorkspace`, `{{prompt:mcp_github_tools}}`
- PRReview: 위 + task context 전체 + `userInteraction` (+ capability로 checkout/github/exploration 등)

### 3.8 Universal / Orchestrator / Command / GitHub / Commit Message

| Agent | 도구 |
|-------|------|
| Universal | 기본 템플릿에 `~{…}` **없음** (컨텍스트 변수만). Capabilities로 도구를 붙일 수 있음 |
| Orchestrator | 도구 없음. `{{availableChatAgents}}` 보고 agent id JSON만 반환 |
| Command | 도구 없음(명령 제안 중심) |
| GitHub | 프롬프트에 직접 `~{…}` 없음 — MCP/capability 경로 |
| Commit Message | 채팅 agent 아님. `getGitChanges` 등 SCM 경로 |

---

## 4. 에이전트에 도구를 제공하는 방법 (실무)

### 4.1 권장: system prompt에 `~{toolId}`

```typescript
// prompt template 예
template: `
## Tools
- ~{${FILE_CONTENT_FUNCTION_ID}} — read a file
- ~{${WRITE_FILE_REPLACEMENTS_ID}} — apply targeted edits
`
```

- ID 상수를 쓰면 오타·리네임에 안전하다 (`workspace-functions.ts` 등).
- 본문에 도구를 **나열하는 이유**: resolve 후 본문에는 id만 남고, 실제 schema/description은 LM tools 배열로 간다. 사용 규칙·우선순위는 프롬프트 문장으로 가르친다.
- **등록만 하고 `~{…}`를 안 쓰면** 그 agent의 LM 호출에 tool이 실리지 않는다.

### 4.2 Capability로 옵션 도구

```text
{{capability:shell-execution default off}}
```

- fragment `shell-execution`이 켜질 때만 그 안의 `~{shellExecute}` 등이 합쳐진다.
- 사용자/세션 override: AI Configuration / chat capability 토글.

### 4.3 MCP fragment

```text
{{prompt:mcp_github_tools}}
```

서버가 떠 있고 fragment가 등록되어 있어야 한다. AppTester·Explore·CodeReviewer 패턴.

### 4.4 Skills

```text
{{skills}}
~{getSkillFileContent}
```

목록은 변수, 본문 로드는 tool. 상세: [shipping-ai-skills.md](./shipping-ai-skills.md).

### 4.5 사용자 메시지 / Capabilities UI

- 채팅에 `~toolId`를 치면 해당 요청에 tool이 추가된다.
- Generic Capabilities(functions / mcpFunctions / skills / agentDelegation)로 세션·agent 설정에 선택 항목을 넣으면, 내부 prompt fragment가 `~{…}`를 생성해 동일 경로로 합쳐진다.

### 4.6 새 tool을 플랫폼에 추가할 때

1. `ToolProvider` 구현 (`getTool(): ToolRequest` — `id`/`handler`/`parameters`/`description`).
2. 해당 패키지 frontend module에서 `bindToolProvider(MyTool, bind)`.
3. **쓸 agent의 prompt(또는 capability/MCP fragment)에 `~{myToolId}` 추가.**
4. (선택) Configuration 문서용으로 agent `functions` 배열에 id 나열.
5. 필요 시 chat UI용 `ChatResponsePartRenderer` 추가 (todo/userInteraction/shell 등).

제품 전용 tool이면 `theia-extensions/<name>/`에 두고 applications dependency로 번들 (AGENTS.md 제품 코드 규칙).

### 4.7 Custom Agent (`agent.md`)

파일 기반 agent도 본문에 `~{getFileContent}` 등을 쓰면 동일하게 resolve된다.  
Registry에 없는 id는 본문에 `~{…}`가 그대로 남고 tool 목록에는 안 실린다.

---

## 5. 설계 패턴 요약

| 패턴 | 언제 |
|------|------|
| suggest* vs write* | Edit Mode(사용자 리뷰) vs Agent Mode(즉시 적용) |
| 탐색 전용 vs 편집 | Explore/Architect(Next)는 사실 수집; Coder가 수정 |
| `delegateToAgent` | 계획(Architect) / 사실(Explore) 분리, 히스토리 미공유 → 위임 프롬프트에 맥락 명시 |
| task over shell | `listTasks`/`runTask` 우선; shell은 capability·예외 경로 |
| launch vs runTask | 장시간 서버는 launch; 빌드/테스트는 task |
| Read before edit | Coder: `getFileContent` 없이 수정 금지 |

---

## 6. 소스 인덱스

| 주제 | 경로 (`../theia/`) |
|------|-------------------|
| Registry / bindToolProvider | `packages/ai-core/src/common/tool-invocation-registry.ts` |
| `~{…}` 파싱 | `packages/ai-core/src/common/prompt-service-util.ts` |
| Prompt resolve + function map | `packages/ai-core/src/common/prompt-service.ts` |
| Chat에 tools 전달 | `packages/ai-chat/src/common/chat-agents.ts` |
| IDE tool bind 목록 | `packages/ai-ide/src/browser/frontend-module.ts` |
| Workspace / task / launch IDs | `packages/ai-ide/src/common/workspace-functions.ts` |
| File change IDs | `packages/ai-ide/src/common/file-changeset-function-ids.ts` |
| Coder prompts | `packages/ai-ide/src/common/coder-replace-prompt-template.ts` |
| Architect / Explore prompts | `packages/ai-ide/src/browser/architect-prompt-template.ts`, `explore-prompt-template.ts` |
| Shell tool + capability | `packages/ai-terminal/.../shell-execution-tool.ts`, `ai-ide/.../shell-execution-capability-contribution.ts` |
| Delegation | `packages/ai-chat/src/browser/agent-delegation-tool.ts` |
| MCP 등록 | `packages/ai-mcp/src/browser/mcp-frontend-service.ts` |
