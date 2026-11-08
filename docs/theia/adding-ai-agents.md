# Theia AI: 새 Agent 추가 방법

조사 기준: 형제 저장소 `../theia/` (`@theia/ai-core`, `@theia/ai-chat`, `@theia/ai-ide` 등) 및 이 제품 모노레포 `theia-ide/`의 조립 방식.  
프레임워크 버전 참고: applications가 끌어오는 `@theia/ai-*` (예: 1.76.0).

---

## 한눈에 보기

| 질문 | Theia에서의 답 |
|------|----------------|
| Agent 유형 | `Agent`(비채팅 포함) / `ChatAgent`(채팅) / 파일 기반 Custom Agent / 외부 패키지(ClaudeCode·Codex 등) |
| Agent / Edit 모드 | 글로벌 모드가 아니라 **agent별 `ChatMode`** (= prompt variant ID). Coder에 Edit Mode / Agent Mode가 대표 예 |
| 상속 | Mode 없음 → `AbstractStreamParsingChatAgent` / Mode 있음 → `AbstractModeAwareChatAgent` |
| System Prompt | `PromptVariantSet` + `{{variables}}` / `~{tools}` / `{{capability:…}}`. 변수 목록은 §4.3, Coder 구조는 §7 |
| Skills / Tools / Settings | `SKILL.md`, `ToolProvider`/`~{toolId}`, preference `ai-features.*` / `AgentSettings` |
| Default agent | preference `ai-features.chat.defaultChatAgent` → (optional) DI `DefaultChatAgentId` → `FallbackChatAgentId` |

**theia-ide 제품 코드는 ChatAgent를 거의 커스텀하지 않는다.** `@theia/ai-ide` 등을 의존성으로 조립하고, `theia-ide-product-ext`는 AI Registry tool 이름만 브랜딩한다.

---

## 1. Agent 유형

### 1.1 코어 인터페이스

- **`Agent`** (`packages/ai-core/src/common/agent.ts`)  
  `id`, `name`, `description`, `prompts`, `languageModelRequirements`, `variables`, `functions` 등.  
  채팅이 아닌 SCM 스파클용 등에도 쓰일 수 있다 (예: CommitMessageAgent는 `Agent`만).
- **`ChatAgent`** (`packages/ai-chat/src/common/chat-agents.ts`)  
  `Agent` 확장. `locations`, `modes?`, `invoke(...)`.
- **등록**  
  - `bind(Agent).toService(MyAgent)` → `AgentService.registerAgent` (built-in prompt 등록)  
  - `bind(ChatAgent).toService(MyAgent)` → `ChatAgentService` contribution  
  **둘 다** bind하는 것이 채팅 agent 표준 패턴이다.

### 1.2 `@theia/ai-ide`에 내장된 ChatAgent

`packages/ai-ide/src/browser/frontend-module.ts`에서 이중 bind:

| Agent | 역할 요약 |
|-------|-----------|
| Architect | 설계·계획 (Plan/Simple 등 mode) |
| Coder | 코딩·파일 변경 (Edit / Agent Mode) |
| ProjectInfo | 프로젝트 정보 |
| CreateSkill | Skill 작성 |
| Orchestrator | 다른 chat agent로 위임 |
| Universal | 범용 |
| AppTester / GitHub / Command | 특수 도메인 |
| Explore / CodeReviewer / PRReview | 탐색·리뷰 |

별도 패키지: `@theia/ai-claude-code`, `@theia/ai-codex` (theia-ide applications가 dependency로 포함).

### 1.3 파일 기반 Custom Agent

코드 확장 없이 워크스페이스에 정의:

- 기본 경로: `.agents/agents/<id>/agent.md` (`.agents/` 선호)
- 호환: `.prompts/agents/`
- 런타임: `CustomChatAgent` + `CustomAgentFactory`가 `ChatAgentService` / `AgentService`에 등록
- 레거시 `customAgents.yml` → per-agent `agent.md` 마이그레이션 (`doc/Migration.md`)

### 1.4 theia-ide에서의 위치

- Agent **구현**은 upstream `../theia/` (`@theia/ai-*`).
- 제품은 `applications/{browser,electron,electron-next}`가 해당 패키지를 의존성으로 끌어와 Theia CLI로 번들.
- 제품 고유 확장이 필요하면 `theia-extensions/<name>`에 ChatAgent를 두고 applications에 dependency 추가 (AGENTS.md의 제품 코드 규칙).

---

## 2. Agent Mode / Edit Mode

**Cursor식 글로벌 Agent/Edit 스위치가 아니다.**  
agent가 선언한 `modes: ChatMode[]`와 prompt **variant**가 모드다.

### Coder (대표)

| Mode 표시명 | Variant / template ID | 동작 요지 |
|-------------|----------------------|-----------|
| Edit Mode | `coder-system-edit` | 변경을 suggestion/changeset로 제안 |
| Agent Mode | `coder-system-agent-mode` | 파일 즉시 수정 등 자율 실행 |
| Agent Mode (Next) | `coder-system-agent-mode-next` | Next 변형 프롬프트 |

소스: `packages/ai-ide/src/browser/coder-agent.ts`,  
`packages/ai-ide/src/common/coder-replace-prompt-template.ts`,  
기반 클래스 `AbstractModeAwareChatAgent` (`mode-aware-chat-agent.ts`).

규칙:

- `modeDefinitions`의 각 `id` = prompt variant ID.
- 요청의 `modeId`가 있으면 그 variant로 system message resolve.
- Agent Mode 최초 사용 시 `AgentModeConfirmationService` 확인 후 preference `ai-features.agentMode.enabled` 등으로 기록.

Architect / CreateSkill도 같은 패턴으로 각자 Plan Mode 등 variant를 둔다.  
UI: `ai-chat-ui`의 mode selector가 `agent.modes`를 표시.

샘플: `examples/api-samples/src/browser/chat/mode-chat-agent-contribution.ts`.

---

## 3. 클래스 상속: `AbstractStreamParsingChatAgent` vs `AbstractModeAwareChatAgent`

맞다. 채팅 agent는 보통 아래 둘 중 하나를 상속한다.

| 기반 클래스 | 언제 |
|-------------|------|
| **`AbstractStreamParsingChatAgent`** | Mode 구분 없음. Explore, CustomChatAgent, ProjectInfo 등 |
| **`AbstractModeAwareChatAgent`** | Edit/Agent/Plan 등 **mode ↔ prompt variant**가 필요. Coder, Architect, CreateSkill |

계층: `AbstractChatAgent` → `AbstractStreamParsingChatAgent` → (`ai-ide`) `AbstractModeAwareChatAgent`.

Mode 없으면 `AbstractStreamParsingChatAgent`로 충분하다. Mode가 있으면 `modeDefinitions`의 각 `id`를 `prompts[].variants`의 fragment id와 맞추고, `systemPromptId`는 variant **set** id로 둔다 (Coder: set `coder-system`, variants `coder-system-edit` / `coder-system-agent-mode` / …).

참고: 모드 없는 Explore 예 — `packages/ai-ide/src/browser/explore-agent.ts`.

---

## 4. System Prompt / Prompt Template

### 4.1 코드에서 정의

1. `prompts: PromptVariantSet[]` (`id`, `defaultVariant`, `variants?`).
2. `systemPromptId` = 그 set id.
3. `AgentService.registerAgent` → built-in fragment 등록.
4. `invoke` → `getResolvedPromptFragment` → system message.
5. 템플릿의 `~{toolId}` / `~{?deferredToolId}` → LLM tools.

Mode-aware는 **선택한 variant id**로 resolve한다.

### 4.2 템플릿 문법 (핵심)

| 문법 | 의미 |
|------|------|
| `{{variableName}}` | AI variable 치환 |
| `{{variableName:arg}}` | 인자가 있는 variable (예: `{{prompt:project-info}}`, `{{skill:my-skill}}`) |
| `{{capability:fragment-id default on\|off}}` | 사용자가 켜/끌 수 있는 prompt fragment 조건부 삽입 |
| `~{toolId}` | tool을 agent에 노출 (본문에 설명 문구로도 쓰임) |
| `{{!-- ... --}}` | 주석 (모델에 안 감) |

채팅 입력의 `#var` / `~fn` / `@agent`와 별개로, **system prompt 파일/문자열 안에서는 `{{…}}` / `~{…}`** 가 resolve 대상이다.

### 4.3 사용 가능한 Variables (`{{name}}`)

등록은 `AIVariableContribution` → `AIVariableService`. 템플릿에서는 **`name`** 필드를 쓴다 (`id`와 다를 수 있음).

#### 자주 쓰는 것 (agent system prompt)

| `{{…}}` | 패키지 | 설명 |
|---------|--------|------|
| `{{productName}}` | ai-core | 앱 이름 (theia-ide 설정) |
| `{{today}}` | ai-core | 오늘 날짜 (`{{today:Format}}` 가능) |
| `{{skills}}` | ai-core | 발견된 skill 목록(XML: name·description). **본문은 `~{getSkillFileContent}`로** |
| `{{skill:skillName}}` | ai-core | 특정 skill 본문 |
| `{{prompt:fragmentId}}` | ai-core | 다른 prompt fragment를 인라인 (예: `project-info`, `open-editors-hint`) |
| `{{capability:…}}` | ai-core | 옵션 기능 조각 (shell, github, apptester, memory, code-review-mode…) |
| `{{contextFiles}}` | ai-ide | 세션에 첨부/북마크된 파일 목록 |
| `{{changeSetSummary}}` | ai-chat | 이전 제안·적용 변경 요약 |
| `{{taskContextSummary}}` | ai-ide | 세션의 task context(계획) 요약 |
| `{{agents}}` / `{{chatAgents}}` | ai-core / ai-chat | agent 목록 |
| `{{contextSummary}}` / `{{contextDetails}}` | ai-chat | 채팅 컨텍스트 요약·상세 |
| `{{openEditors}}` / `{{_ff}}` | ai-core | 열린 에디터 목록 / 짧은 경로 목록 |
| `{{editorContext}}` | ai-editor | 에디터 컨텍스트 |
| `{{file:uri}}` | ai-core | 지정 URI 파일 내용 |
| `{{memoryDirectory}}` | ai-ide | 워크스페이스별 agent memory 디렉터리 |
| `{{taskContext:id}}` | ai-chat | 특정 task context |
| `{{selected_skills}}` 등 | ai-core | Capabilities 패널에서 사용자가 고른 항목들 |

#### 에디터/워크스페이스 (Theia variable 브리지)

| `{{…}}` | 설명 |
|---------|------|
| `{{currentAbsoluteFilePath}}` | 현재 파일 절대 경로 |
| `{{currentRelativeFilePath}}` / `{{_f}}` | 워크스페이스 상대 경로 |
| `{{currentRelativeDirPath}}` | 현재 파일 디렉터리 |
| `{{currentFileContent}}` | 현재 파일 본문 |
| `{{selectedText}}` | 선택 텍스트 |
| `{{lineNumber}}` | 줄 번호 |
| `{{workspaceFolder}}` | 워크스페이스 폴더 |

목록은 AI Configuration → Variables에서도 볼 수 있다. 확장 패키지가 추가 contribution을 등록하면 더 늘어난다.

**참고:** Coder 기본 프롬프트는 `{{skills}}`를 **쓰지 않는다**. Explore 등은 `{{skills}}`를 넣는다. Skill을 쓰려면 프롬프트에 `{{skills}}`(+ `~{getSkillFileContent}`)를 넣거나 Capabilities/`selected_skills` 경로를 쓰면 된다.

### 4.4 런타임 커스터마이즈

- AI Configuration View, `.prompttemplate`, agent 폴더 prompt.
- Custom agent: `agent.md` 본문 → `{name}_prompt` fragment.
- `{{prompt:project-info}}` → 보통 `.prompts/project-info.prompttemplate` (ProjectInfo agent가 관리).

---

## 5. Skills, Tools, Settings

### Tools

- `ToolProvider.getTool(): ToolRequest` → `bindToolProvider(MyTool, bind)`.
- Registry: `ToolInvocationRegistry`.
- IDE 도구는 `@theia/ai-ide` frontend-module에서 다수 등록 (workspace files, search, file edit, tasks, skill content, git, user interaction 등).
- 프롬프트에서 `~{toolId}`로 참조해야 agent가 해당 tool을 쓴다 (`Agent.functions` 문서화와 맞춤).

### Skills

- `SKILL.md` + YAML frontmatter (`name`, `description`, …).
- 탐색 경로 예: `.prompts/skills/`, `.agents/skills/`, user dirs, `~/.theia/skills/` (`ai-core` README).
- 프롬프트 변수 `{{skills}}`, 본문 로드 tool `getSkillFileContent`.
- CreateSkill agent가 작성 지원.
- `AgentSettings.genericCapabilitySelections.skills` 등으로 agent/세션별 선택.

### Settings (preference)

| Key | 용도 |
|-----|------|
| `ai-features.AiEnable.enableAI` | AI 마스터 스위치 |
| `ai-features.chat.defaultChatAgent` | @mention 없을 때 기본 chat agent |
| `ai-features.chat.pinChatAgent` | @mention pin |
| `ai-features.agentSettings` | agent별 enable, showInChat, variants, LM, skills… |
| `ai-features.agentMode.enabled` | Coder Agent Mode 확인 관련 |
| `ai-features.orchestrator.excludedAgents` | Orchestrator 위임 제외 |
| `ai-features.skills.skillDirectories` | 추가 skill 디렉터리 |

MCP: `@theia/ai-mcp` / `ai-mcp-ui` (applications dependency에 포함).

---

## 6. Default agent 설정

해석 순서 (대략, `ChatAgentService` / chat service):

1. 사용자 메시지 `@AgentName`
2. Preference `ai-features.chat.defaultChatAgent`
3. DI `DefaultChatAgentId` (optional)
4. DI `FallbackChatAgentId` (optional)

**현재 `@theia/ai-ide` / theia-ide는 `DefaultChatAgentId`·`FallbackChatAgentId`를 bind하지 않는다.**  
README상 Orchestrator/Universal 언급과 달리, 런타임 기본은 **preference(또는 walkthrough의 setDefault)** 에 더 의존한다. Orchestrator 내부 fallback은 코드에서 `'Universal'` 등으로 하드코딩된 부분이 있다.

Adopter가 제품 기본을 고정하려면 product/앱 모듈에서:

```typescript
bind(DefaultChatAgentId).toConstantValue({ id: 'Orchestrator' });
bind(FallbackChatAgentId).toConstantValue({ id: 'Universal' });
```

또는 preference 스키마 default / 설정 UI로 `ai-features.chat.defaultChatAgent`를 제공한다.

Welcome 추천: `default-chat-agent-recommendation-service.ts` (Coder / Architect / Universal 등).

---

## 7. Coder Agent system prompt 구조

소스 SSOT: `../theia/packages/ai-ide/src/common/coder-replace-prompt-template.ts`  
Agent: `coder-agent.ts` — set id `coder-system`, **defaultVariant = Agent Mode**.

| Mode | Fragment ID | 편집 API | 성격 |
|------|-------------|---------|------|
| Edit Mode | `coder-system-edit` | `suggest*` (사용자 리뷰) | 제안만, task는 사용자 확인 후 |
| Agent Mode | `coder-system-agent-mode` | `write*` (즉시 적용) | 자율·끝까지 완수 |
| Agent Mode (Next) | `coder-system-agent-mode-next` | `write*` | Agent Mode 압축·강화 + 위임(Architect/Explore) |

공통 컨텍스트 꼬리(세 mode 모두 유사):

- `{{contextFiles}}` — 첨부/북마크 파일
- `{{changeSetSummary}}` — 이전 변경
- `{{prompt:project-info}}` — 프로젝트 정보 fragment
- `{{prompt:open-editors-hint}}` — 열린 에디터 힌트
- `{{taskContextSummary}}` — task context 요약
- `{{productName}}` (및 Next만 `{{today}}`)

### 7.1 Edit Mode (`coder-system-edit`) — 섹션 요약

1. **Identity** — IDE 통합 assistant; 변경은 **제안**, 사용자가 수락  
2. **Context Retrieval** — `getWorkspaceFileList`, `getFileContent`, `findFilesByPattern`, `searchInWorkspace`, `updateContextFiles`; 경로 확인·단계적 탐색  
3. **Propose Code Changes** — **응답에 diff를 직접 출력하지 말 것**. `getFileContent` → `getProposedChanges` / `suggestFileReplacements` / `suggestFileContent` / `clearFileChanges`. 변경 설명 주석 금지  
4. **Tasks** — `listTasks` / `runTask`; **제안 미수락 시 워크스페이스는 원본**; 실행 전 사용자 확인  
5. **File Validation** — `getFileDiagnostics` (역시 수락 전엔 원본 기준)  
6. **Diagrams** — 작은 Mermaid  
7. **Additional Context** — 위 공통 변수들  
8. **Final Instruction** — 제안 중심; 빌드/린트는 사용자 요청·수락 후에

### 7.2 Agent Mode (`coder-system-agent-mode`) — 섹션 요약

1. **Identity** — autonomous agent; 관련 작업 완료 전까지 yield 금지  
2. **Core Principles** — Autonomy/Persistence, Objectivity, Parallel tool calls, Planning/Reflection  
3. **Code Quality** — over-engineering 금지, 보안(OWASP)  
4. **Tools Reference** — 추측 금지; 탐색/편집/검증/테스트/런치/todo  
   - 편집: **`writeFileReplacements` / `writeFileContent`** (즉시 적용, 리뷰 없음)  
   - **Read Before Edit** 필수 (`getFileContent`)  
5. **`{{capability:… default off}}`** — `shell-execution`, `github`, `apptester`  
6. **Workflow** — Understand → Investigate → Plan/Implement → Validate → Test → Final Review  
7. **Error Recovery / When to Seek Clarification / Communication**  
8. **Context** — 공통 변수  
9. **Final Instruction** — 에러·린트·빌드·테스트·보안까지 끝난 뒤에만 종료  

### 7.3 Agent Mode (Next) — Edit/Agent와의 차이점

- 더 짧은 톤, **최소 변경·재사용 우선**(코드베이스 → stdlib → 기존 의존성 → 신규).  
- **Agent Delegation**: `~{delegateToAgent}` → Architect(계획) / Explore(사실 수집). 히스토리 미공유 → 위임 프롬프트에 목표·질문·파일 명시. Architect의 `taskContextId`는 `getTaskContext`로 읽기.  
- Task Context를 **계획/todo 전에** 확인; 있으면 신뢰하고 구현.  
- 추가 capability: `code-review-mode`, `memory` (+ shell/github/apptester).  
- Output Format: Modified files / Summary / Build·lint·test evidence(실행 안 한 검증을 PASS로 보고 금지).  
- Follow-up·가설 기반 디버그·dead-end 감지·범위 확대 시 재위임 규칙이 더 구체적.

### 7.4 새 agent가 Coder를 참조할 때

| 빌릴 것 | 비고 |
|---------|------|
| Identity + 역할 경계 | “제안만” vs “즉시 수정”을 mode/도구로 명확히 |
| Tool 목록·사용 규칙 | `~{id}`로 노출; Read-before-edit, 검색 전략 표 |
| Context 블록 | `contextFiles` / `project-info` / `changeSetSummary` / `taskContextSummary` |
| Capability 슬롯 | 옵션 기능은 `{{capability:…}}` |
| `{{skills}}` | Coder 기본엔 없음 → 필요하면 Explore처럼 추가 |
| Mode가 필요하면 | `AbstractModeAwareChatAgent` + variant id = mode id |
| Mode 없으면 | `AbstractStreamParsingChatAgent` + 단일 `defaultVariant` |

도구 id 상수: `workspace-functions.ts`, `file-changeset-function-ids.ts` 등 (`getFileContent`, `suggestFileReplacements`, `writeFileReplacements`, …).

상세 원문·전체 문구는 위 `.ts` 파일을 연다. 이 절은 구조·차이·참조 포인트만 정리한다.

---

## 8. theia-ide에 새 Agent를 추가하는 방법

### A. 코드 기반 (권장: `theia-extensions/`)

1. 패키지 생성 예: `theia-extensions/my-agent/` (`theia-ide-my-agent-ext`).
2. 클래스: **Mode 없음 → `AbstractStreamParsingChatAgent`**, Mode 있음 → `AbstractModeAwareChatAgent`.
3. prompt template 모듈에서 system prompt 문자열(및 variants) 정의 — Coder의 `coder-replace-prompt-template.ts` 패턴.
5. DI 모듈:

```typescript
bind(MyChatAgent).toSelf().inSingletonScope();
bind(Agent).toService(MyChatAgent);
bind(ChatAgent).toService(MyChatAgent);
// 필요 시 bindToolProvider(...); bind(DefaultChatAgentId)...
```

6. `package.json`의 `theiaExtensions`에 frontend 모듈 경로.
7. `applications/browser|electron|electron-next/package.json` dependencies에 패키지 추가 (workspace/lerna 등록).
8. `yarn build:extensions` → `yarn browser|electron build` (또는 `build:dev`).
9. AI enable 후 Chat에서 `@MyAgent` 스모크. default면 preference 또는 DI 설정.

모드가 필요하면 `modeDefinitions`의 id와 `prompts.variants` id를 일치시킨다.

### B. 파일 기반 (확장 없이)

워크스페이스에 `.agents/agents/<id>/agent.md` 작성.  
도구·커스텀 invoke가 복잡하면 A로.

### C. Upstream에 기여

프레임워크 공통 agent는 `../theia/packages/ai-ide` (또는 전용 패키지)에 추가 후 theia-ide는 버전만 올린다. 제품 전용 동작만 `theia-extensions/`에 둔다.

### D. 제품이 지금 하는 일

`theia-extensions/product/.../theia-ide-ai-registry-configuration.ts` — AI Registry `getToolName()`만 `theia-ide` / `theia-ide-next`로 재바인딩. ChatAgent 클래스·프롬프트·default DI 커스텀 없음.

---

## 핵심 파일 맵 (`../theia/`)

| 경로 | 역할 |
|------|------|
| `packages/ai-core/src/common/agent.ts` | `Agent` |
| `packages/ai-core/src/common/agent-service.ts` | 등록·enable |
| `packages/ai-core/src/common/prompt-service.ts` | fragment / custom agents API |
| `packages/ai-core/src/common/skill.ts` | Skill |
| `packages/ai-chat/src/common/chat-agents.ts` | `ChatAgent`, `AbstractChatAgent` |
| `packages/ai-chat/src/common/chat-agent-service.ts` | default/fallback, 목록 |
| `packages/ai-chat/src/common/ai-chat-preferences.ts` | `defaultChatAgent` 등 |
| `packages/ai-chat/src/common/custom-chat-agent.ts` | 파일 기반 agent 런타임 |
| `packages/ai-ide/src/browser/frontend-module.ts` | 내장 agent·tool DI 허브 |
| `packages/ai-ide/src/browser/coder-agent.ts` | Edit/Agent Mode 예 |
| `packages/ai-ide/src/common/coder-replace-prompt-template.ts` | Coder system prompt 원문 |
| `packages/ai-ide/src/browser/mode-aware-chat-agent.ts` | mode ↔ variant |
| `packages/ai-ide/src/browser/explore-agent.ts` | Mode 없는 `AbstractStreamParsingChatAgent` 예 |
| `packages/ai-core/src/browser/*-variable-contribution.ts` | `{{skills}}`, `{{prompt:…}}` 등 |
| `doc/Migration.md` | `.agents/agents/<id>/agent.md` 레이아웃 |
| `examples/api-samples/src/browser/chat/*` | adopter 샘플 |

theia-ide:

| 경로 | 역할 |
|------|------|
| `applications/*/package.json` | `@theia/ai-*` 조립 |
| `theia-extensions/product/.../theia-ide-ai-registry-configuration.ts` | Registry tool 이름 브랜딩 |

---

## 관련 문서

- [shipping-ai-skills.md](./shipping-ai-skills.md) — 제품에 Skill 번들·폴더 설정·개발 중 사용
- upstream: `packages/ai-ide/README.md`, `packages/ai-core/README.md` (skills), `packages/ai-chat/README.md`, `doc/Migration.md` (custom agents 폴더)
