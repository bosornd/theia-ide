# Theia AI: Message Compaction

조사 기준: 형제 저장소 `../theia/` (`@theia/ai-core`, `@theia/ai-chat`, `@theia/ai-chat-ui`, `@theia/ai-openai`, `@theia/ai-anthropic`, `@theia/ai-claude-code`).  
관련 업스트림: [#17636](https://github.com/eclipse-theia/theia/issues/17636), [#17746](https://github.com/eclipse-theia/theia/pull/17746), [#17795](https://github.com/eclipse-theia/theia/pull/17795), [#17324](https://github.com/eclipse-theia/theia/issues/17324) (일반 chat `/compact`).  
외부 참고: [OpenAI Compaction guide](https://developers.openai.com/api/docs/guides/compaction), [Claude Code `/compact`](https://code.claude.com/docs/en/commands).  
로컬 compact 요소 기술: [`local-message-compaction.md`](./local-message-compaction.md).

---

## 한눈에 보기

| 질문 | 답 |
|------|-----|
| 무엇인가 | 긴 채팅의 context/토큰을 줄이는 압축. Theia에는 **(A) 프로바이더 server-side compaction**과 **(B) Claude Code `/compact` 전달**이 있다 |
| (A) Server-side | OpenAI Responses / Anthropic: 요청에 compaction 지시 → opaque 마커 수신·세션 저장·재전송 → UI “Conversation compacted” |
| (B) `/compact` | **Claude Code agent 전용** slash command. Theia는 명령을 등록·프롬프트로 넘기고, **실제 요약·history 치환은 Claude Code 런타임**이 수행 |
| 일반 ChatAgent(Coder 등) `/compact` | **없음** ([#17324](https://github.com/eclipse-theia/theia/issues/17324) 미구현). chat-model `isStale` + in-session summary도 master에 없음 |
| 인접 기능 | `ChatSessionSummaryAgent` / “Summarize Current Session” → Task Context **파일** 요약 (같은 세션 history를 자르지 않음) |
| 기본값 (A) | `ai-features.chat.serverSideCompaction` = **true** (지원 모델만 실질 적용) |
| 제품(`theia-ide`) | preference/렌더러 커스텀 없음. Claude Code 패키지를 dependency로 넣으면 `/compact` 자동 등록 |

토큰 낭비·context overflow를 줄이는 수단이 (A)/(B)다. “client-side가 완전 공백”은 아니며, **일반 Theia chat 파이프라인의 in-session local summarization**만 비어 있다.

---

## 1. 개념

### 1.1 세 갈래 (혼동 주의)

| 경로 | 누가 요약하나 | 무엇을 줄이나 | master 상태 |
|------|---------------|---------------|-------------|
| **Server-side compaction** | OpenAI / Anthropic API | 다음 LLM 요청의 컨텍스트 (opaque marker) | **구현됨** (§2–§4) |
| **Claude Code `/compact`** | Claude Code 프로세스 | Claude Code **세션** history (`resume` id) | **제공됨** — Theia는 위임 (§1.3) |
| **Theia chat-model local compact** | IDE + summarize 모델 | `getMessages`가 보는 chat hierarchy (`isStale` 등) | **미구현** (#17324, 실험 브랜치만) |

- **Server-side:** 프로바이더가 threshold를 넘으면 압축하고 opaque `compaction` 아이템을 돌려준다. Theia는 마커를 저장·재전송한다.
- **Claude Code `/compact`:** 사용자가 slash로 수동 트리거(또는 Claude Code 측 자동 compact). Theia `MutableChatModel`의 `getMessages` 경로와 무관하다 — Claude Code agent는 transcript를 다시 보내지 않고 `resume: claudeSessionId`로 이어간다.
- **미구현 local compact:** 일반 agent가 같은 세션에서 과거 턴을 stale 처리하고 summary node로 치환하는 것. Task Context 요약은 **새 세션 handoff**용이라 이 칸에 넣지 않는다.

### 1.2 Opaque marker (server-side)

`CompactionMessage` (`packages/ai-core/src/common/language-model.ts`):

```ts
{
  actor: 'ai',
  type: 'compaction',
  provider: 'openai-responses' | 'anthropic' | ...,
  data: unknown,      // 암호화 blob 등 — 타 프로바이더가 해석하지 않음
  summary?: string    // Anthropic은 human-readable summary를 노출할 수 있음
}
```

`provider`가 일치할 때만 해당 백엔드가 `data`를 재전송한다. 다른 프로바이더로 모델을 바꾸면 마커는 skip되고, 그 주변 실제 히스토리(또는 OpenAI의 경우 prefix drop 후 이후 턴만)로 이어진다.

### 1.3 `/compact` 명령 처리 (Claude Code)

**제공 여부:** 예 — 다만 **`@ClaudeCode` agent 범위**에서만. Coder/Architect 등 일반 ChatAgent 입력창의 범용 `/compact`는 없다.

#### 등록

`ClaudeCodeSlashCommandsContribution` (`packages/ai-claude-code/src/browser/claude-code-slash-commands-contribution.ts`):

- `PromptService.addBuiltInPromptFragment`로 static slash 등록.
- `commandName: 'compact'`, `template: '/compact'`, `commandAgents: [CLAUDE_CHAT_AGENT_ID]`.
- description: “Compact conversation with optional focus instructions”.
- `/clear`, `/config`, `/memory`, `/resume` 등과 함께 등록. 워크스페이스 `.claude/commands/*.md`는 동적 명령으로 추가.

채팅 입력의 `/` 자동완성은 `PromptService.getCommands(agentId)` / `isKnownCommand`를 본다. 파서는 `/name`을 `#prompt:name|args` 변수 파트로 바꾼다 (`chat-request-parser.ts`: commands are syntactic sugar for `#prompt:…`).

#### invoke 분기

`ClaudeCodeChatAgent.invoke` (`claude-code-chat-agent.ts`):

```text
request.request.text 에서 /word 스캔
        │
        ├─ localCommands 에 있으면  (clear → AI_CHAT_HOME,
        │   config / memory / resume → Theia CommandService)
        │   → IDE 커맨드만 실행하고 LLM 호출 없이 complete
        │
        └─ 그 외 (/compact 포함)
              → claudeCode.send({
                   prompt: request.request.text,   // 예: "/compact focus on auth"
                   options: { resume: claudeSessionId, ... }
                 })
              → Claude Code 런타임이 /compact 수행
              → 스트림(요약 결과·stdout 등)을 chat response에 표시
```

**중요:** `/compact`는 `localCommands`에 **없다**. Theia가 chat hierarchy를 요약하지 않고, 사용자 텍스트를 Claude Code에 그대로 넘긴다. Claude Code는 자체 세션 history를 요약·치환한다 (lossy; 인자로 focus 지시 가능). 임계치 근접 시 Claude Code 쪽 **자동 compact**도 별도로 존재한다.

#### Theia 세션과의 관계

- Claude Code agent는 `AbstractChatAgent.getMessages`로 전체 Theia transcript를 재구성하지 않는다. 이전 턴은 `resume` 세션 id로 이어진다.
- 따라서 `/compact` 후 **Claude Code가 보는 컨텍스트는 줄어들지만**, Theia UI의 과거 말풍선은 그대로 남을 수 있다 (표시용 history ≠ wire history).
- 다른 agent로 전환하면 Claude Code 세션과 단절된다는 경고(`differentAgentRequestWarning`)가 뜬다.

#### 일반 chat `/compact` (#17324)와의 차이

| | Claude Code `/compact` (현재) | #17324 제안 (미구현) |
|--|-------------------------------|----------------------|
| 대상 agent | ClaudeCode만 | 패널 chat 일반 |
| 요약 실행 주체 | Claude Code 런타임 | Theia + `ChatSessionSummaryAgent` 등 |
| History 모델 | Claude session id | `MutableChatModel` + `isStale` / summary node |
| UI | CC 응답 스트림 | 접힌 summary + stale 구분 |

---

## 2. Server-side 활성화 계층 (broad → specific)

우선순위는 **세션 > 프로바이더 override > 글로벌**. 모델 **capability**가 없으면 전부 무시.

```text
LanguageModel.serverSideCompactionSupport  (capability gate)
        │
        ▼
resolveServerSideCompaction(capability, enabledByDefault, request.compaction)
        │
        ├─ request.compaction.enabled 가 있으면 → 그 값
        └─ 없으면 → enabledByDefault
                    (= resolveCompactionDefault(global, perProviderOverride))
```

| 계층 | Preference / UI | 값 |
|------|-----------------|-----|
| 글로벌 | `ai-features.chat.serverSideCompaction` | `boolean`, default `true` |
| 글로벌 threshold | `ai-features.chat.serverSideCompactionTokenThreshold` | `integer` ≥ **50_000**, unset면 프로바이더 기본 |
| OpenAI override | `ai-features.openAiOfficial.serverSideCompaction` | `default` \| `enabled` \| `disabled` |
| OpenAI threshold | `ai-features.openAiOfficial.serverSideCompactionTokenThreshold` | ≥ 50_000 |
| Anthropic override | (동형 preference in `@theia/ai-anthropic`) | 동일 enum |
| 세션 | Chat session settings dialog → Server-Side Compaction | `enabled` / `disabled` / follow default + optional token threshold |

헬퍼: `resolveCompactionDefault`, `resolveCompactionTokenThresholdDefault`, `resolveCompactionTokenThreshold`, `resolveServerSideCompaction` — 모두 `language-model.ts`.

최소 threshold `SERVER_SIDE_COMPACTION_TOKEN_THRESHOLD_MINIMUM = 50_000`은 Anthropic 제약에 맞춘 공통 하한이다.

---

## 3. Server-side 런타임 흐름

```text
Chat session (commonSettings.compaction)
        │
        ▼
AbstractChatAgent → LanguageModelRequest.compaction
        │
        ├─ OpenAI (Response API + capability)
        │     context_management: [{ type: 'compaction', compact_threshold? }]
        │
        └─ Anthropic (capability)
              betas: ['compact-2026-01-12']
              context_management.edits: [{ type: 'compact_20260112', trigger? }]
        │
        ▼
Stream: CompactionResponsePart
        │
        ▼
CompactionChatResponseContent (persisted with session)
        │
        ├─ UI: CompactionPartRenderer (“Conversation compacted”)
        ├─ Token tooltip: “compacted N×”
        └─ Next request: toLanguageModelMessage() → CompactionMessage replay
```

### 3.1 OpenAI (`@theia/ai-openai`)

- **전제:** `useResponseApi` / `serverSideCompactionSupport`. Chat Completions 경로에서는 compaction 지시자를 붙이지 않으며, `CompactionMessage`는 drop.
- **요청:** `applyResponseApiCompaction`이 `context_management: [{ type: 'compaction', compact_threshold? }]` 부착.
- **스트림:** `response.output_item.done` + `item.type === 'compaction'` → `{ provider: 'openai-responses', data: { id, encrypted_content } }`.
- **재전송 (핵심 토큰 절감):** `processMessages`가 **가장 최근** `openai-responses` compaction 마커를 찾고, **그 이전 메시지를 전부 버린 뒤** 마커 + 이후 메시지만 input으로 보낸다. 이전 compaction 마커는 subsumed.
- OpenAI 문서의 standalone `/responses/compact` 엔드포인트는 Theia가 쓰지 않는다. create 요청의 server-side compaction만 사용.

### 3.2 Anthropic (`@theia/ai-anthropic`)

- **요청:** `applyCompactionParams` → beta `compact-2026-01-12` + `context_management.edits`.
- **스트림:** compaction content block + delta → `{ provider: 'anthropic', data: { content, encrypted_content }, summary: content }`.
- **재전송:** OpenAI와 달리 **주변 실제 메시지를 유지**한 채 anthropic compaction 블록을 content로 삽입. compaction이 꺼져 있거나 타 프로바이더 마커면 블록만 빼고 히스토리는 유지.

### 3.3 기타 프로바이더

- **Ollama:** `CompactionMessage`를 표현할 방법이 없어 drop (`ollama-language-model.ts` 주석).
- Capability 없는 모델: 설정 UI에 값이 있어도 요청이 바뀌지 않음.

---

## 4. Server-side UI / 영속화

| 위치 | 역할 |
|------|------|
| `packages/ai-chat-ui/.../session-settings-dialog.tsx` | 세션별 enable + token threshold |
| `packages/ai-chat-ui/.../compaction-part-renderer.tsx` | 인라인 “Conversation compacted”; `summary` 있으면 `<details>` |
| `packages/ai-chat-ui/.../chat-token-usage-indicator-util.ts` | 세션 내 compaction 파트 개수 → tooltip “compacted N×” |
| `packages/ai-chat/.../chat-model.ts` | `CompactionChatResponseContent` + `toLanguageModelMessage` / serialize |
| `packages/ai-chat/.../chat-content-deserializer.ts` | 세션 복원 시 `kind: 'compaction'` |

세션을 저장·리로드해도 마커와 토큰 usage가 함께 복원된다 (PR #17746 검증 시나리오).  
`/compact`(Claude Code) 결과는 일반 Markdown/tool 응답으로 보이며, 위 `kind: 'compaction'` 마커와는 다른 축이다.

---

## 5. 제품(`theia-ide`) 관점

- `applications/*`는 `@theia/ai-*`를 의존성으로 조립할 뿐, server-side compaction preference나 렌더러를 오버라이드하지 않는다.
- Claude Code를 포함한 타깃에서는 `@theia/ai-claude-code`가 `/compact` slash를 등록한다. 동작 확인: Claude Code agent에서 `/compact` (또는 `/compact <focus>`) → 세션이 이어지며 컨텍스트가 줄어드는지.
- Server-side 확인: Responses API를 켠 OpenAI 또는 Anthropic 지원 모델 + 글로벌 compaction on → 긴 대화 후 마커·tooltip·세션 재개.
- OpenAI server-side는 **Response API**가 켜져 있어야 한다. Chat Completions만 쓰면 설정이 무시된다.

---

## 6. 설계 시사점 / 한계

1. **토큰 절감 효과는 경로마다 다르다.** Server-side는 프로바이더·재전송 전략(OpenAI prefix drop vs Anthropic 마커+주변 유지)에 의존. Claude Code `/compact`는 CC 세션만 줄인다.
2. **품질 트레이드오프:** opaque(server-side)든 CC 요약이든 lossy. 임계치·수동 compact 시점을 너무 공격적으로 잡으면 세부 지시·tool 결과가 빠질 수 있다.
3. **프로바이더 전환 (server-side):** `openai-responses` 마커만 있는 채 Anthropic으로 바꾸면 마커는 skip. prefix drop 이후면 복구 어려울 수 있다.
4. **“client-side 완전 미구현”은 과장이다.** Claude Code `/compact`와 Task Context 요약은 있다. 빠진 것은 **일반 ChatAgent용 in-session local summarization** (`isStale` / #17324). 상세는 [`local-message-compaction.md`](./local-message-compaction.md).
5. **커스텀 OpenAI-compatible 엔드포인트**가 Responses + `context_management`를 지원하지 않으면 server-side 플래그가 무시되거나 실패할 수 있다.

---

## 7. 주요 소스 맵

| 관심사 | 경로 |
|--------|------|
| 타입·resolve 헬퍼·최소 threshold | `../theia/packages/ai-core/src/common/language-model.ts` |
| 글로벌 preference | `../theia/packages/ai-core/src/common/ai-core-preferences.ts` |
| Request에 settings 전달 | `../theia/packages/ai-chat/src/common/chat-agents.ts` |
| Content 모델 | `../theia/packages/ai-chat/src/common/chat-model.ts` |
| OpenAI 요청/리플레이 | `../theia/packages/ai-openai/src/node/openai-language-model.ts`, `openai-response-api-utils.ts` |
| Anthropic 요청/스트림 | `../theia/packages/ai-anthropic/src/node/anthropic-language-model.ts` |
| 세션 UI / 마커 렌더 / 토큰 표시 | `../theia/packages/ai-chat-ui/src/browser/...` |
| `/compact` slash 등록 | `../theia/packages/ai-claude-code/src/browser/claude-code-slash-commands-contribution.ts` |
| `/compact` 전달·localCommands | `../theia/packages/ai-claude-code/src/browser/claude-code-chat-agent.ts` |
| slash → `#prompt:` 파싱 | `../theia/packages/ai-chat/src/common/chat-request-parser.ts` |
