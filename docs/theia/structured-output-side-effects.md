# Theia AI: Structured JSON Output → Chat UI 분리·Side Effect 처리

조사 기준: 형제 저장소 `../theia/` (`@theia/ai-core`, `@theia/ai-chat`, `@theia/ai-ide`, `@theia/ai-openai`, `@theia/ai-vercel-ai`, `@theia/ai-terminal` 등).  
관련: [adding-ai-agents.md](./adding-ai-agents.md).

**목표 시나리오:** LLM이 `{ response, session?, commit?, … }` JSON을 내고, 채팅 UI에는 `response`만 보이며 나머지 필드는 제품 코드가 처리(세션 제목 변경, git commit 등).

---

## 한눈에 보기

| 질문 | 답 |
|------|----|
| Schema 강제 API | `LanguageModelRequest.response_format` → `{ type: 'json_schema', json_schema: { name, schema, strict? } }` |
| 파싱 결과 타입 | `LanguageModelParsedResponse` (`parsed` + `content`) |
| 채팅 기본 경로 | **`response_format`을 넘기지 않음**. `parsed`도 쓰지 않고 `content` 문자열을 Markdown으로 넣음 → **JSON 전체가 UI에 노출될 수 있음** |
| UI에 본문만 보이게 | `addContentsToResponse` 오버라이드에서 `parsed.response`만 `MarkdownChatResponseContent`로 추가 |
| Side effect | 같은 오버라이드에서 필드 처리 **또는** (권장) **Tool**로 rename/commit 등 수행 |
| 세션 제목 | 이미 `ChatSessionNamingAgent` + `ChatService.renameSession` 존재. JSON `session` 필드와 **역할 중복 주의** |
| 스트리밍 + schema | OpenAI/Vercel structured 경로는 **비스트리밍**. 채팅에 점진 JSON 필드 분리 API 없음 |

---

## 1. Structured output API (`@theia/ai-core`)

```typescript
// packages/ai-core/src/common/language-model.ts
response_format?:
  | { type: 'text' }
  | { type: 'json_object' }          // 타입만 있고 프로바이더 구현 거의 없음
  | ResponseFormatJsonSchema;       // type: 'json_schema'

interface LanguageModelParsedResponse {
  parsed: unknown;
  content: string;
  usage?: UsageResponsePart;
}
```

헬퍼 (`language-model-util.ts`):

- `getTextOfResponse` — parsed면 `content` 문자열
- `getJsonOfResponse` / `getJsonOfText` — fenced JSON 또는 `{`/`[` 시작 텍스트 파싱 (프롬프트만으로 JSON을 받을 때 폴백)

**별도 `structuredOutput` / `outputSchema` 이름은 없다.** 모두 `response_format`이다.

### 프로바이더 지원

| 프로바이더 | `json_schema` | 비고 |
|-----------|---------------|------|
| OpenAI (`ai-openai`) | ✅ `chat.completions.parse` (`supportsStructuredOutput`) | **비스트리밍**. tools+structured **미구현**(TODO). Response API 경로와 schema 연결 약함 |
| Vercel AI | ✅ `generateObject` | allowlist / `supportsStructuredOutput` |
| Ollama | ⚠️ `format: 'json'`만 | **요청 schema 본문은 전달 안 됨** |
| Anthropic / Google / HF / Copilot | ❌ | 네이티브 패키지에 `response_format` 처리 없음 |

---

## 2. 채팅 파이프라인이 structured를 “무시”하는 지점

### `sendLlmRequest` — format 미전달

`AbstractChatAgent.sendLlmRequest` (`ai-chat/.../chat-agents.ts`)는 messages/tools/settings만 넣고 **`response_format`을 넣지 않는다.**  
채팅 agent에서 schema를 쓰려면 **`sendLlmRequest` 오버라이드**가 필요하다.

### `addContentsToResponse` — `parsed` 미사용

`AbstractStreamParsingChatAgent`:

```typescript
if (isLanguageModelParsedResponse(languageModelResponse)) {
    const contents = this.parseContents(languageModelResponse.content, request);
    request.response.response.addContents(contents);
    // ← parsed 객체는 여기서 소비되지 않음
}
```

즉 LM이 schema를 지켜도, 기본 chat agent는 **JSON 문자열 전체를 Markdown**으로 보여 줄 수 있다.  
“`response`만 UI”는 **반드시 agent 쪽에서 가로채야** 한다.

### UI / 히스토리 계약 (`ChatResponseContent`)

- `asString()`이 `undefined` → 다음 LLM 턴 히스토리에서 **제외**
- `InformationalChatResponseContent` → `asString()` undefined, 렌더러는 **null**(사용자에게 안 보임). Orchestrator가 위임 안내에 사용
- Markdown / Text → 사용자 본문
- ToolCall → 별도 part renderer

글로벌 “response transformer”는 **없다.** 확장점은 agent 오버라이드·content matcher·part renderer다.

---

## 3. Upstream에 이미 있는 패턴

### A. Terminal Agent — `response_format` + `parsed` (채팅 UI 밖)

`packages/ai-terminal/.../ai-terminal-agent.ts`:

1. `response_format: { type: 'json_schema', json_schema: { schema: … } }`
2. `isLanguageModelParsedResponse` → `result.parsed` 사용
3. 실패 시 `getJsonOfResponse` 폴백

**채팅이 아닌 Agent**에서 structured를 쓰는 공식 레퍼런스.

### B. Orchestrator — 텍스트 JSON 파싱 + UI 숨김 + side effect

`packages/ai-ide/src/common/orchestrator-chat-agent.ts`의 `addContentsToResponse`:

1. `getTextOfResponse` → `getJsonOfText`로 agent id 배열 추출
2. UI에는 `InformationalChatResponseContent`(사실상 숨김)만 추가
3. `overrideAgentId`로 **다른 chat agent에 위임** (side effect)

프롬프트 JSON에 의존하며 **`response_format`은 쓰지 않는다.**

### C. CommandChatAgent — `AbstractTextToModelParsingChatAgent`

전체 응답 텍스트를 파싱해 `Markdown` + `CommandChatResponseContent`로 나눔. Schema 강제 없음.

### D. Tools — side effect의 주류

예: `todoWrite`, file changeset(`changeSet.setTitle`), git/shell tools.  
사용자에게 보일 설명은 **assistant 텍스트**, 동작은 **tool call**. Coder/Universal과 가장 잘 맞음.

### E. 세션 제목 (이미 분리됨)

`ChatService.updateSessionMetadata` (`chat-service.ts`):

1. 첫 요청 `displayText`로 임시 제목
2. 응답 complete 후 **`ChatSessionNamingAgent`**가 별도 LLM으로 이름 생성
3. 수동/API: `ChatService.renameSession(sessionId, title)`

모델 JSON의 `session` 필드를 쓰는 내장 경로는 **없다.**

### F. Commit — 채팅 밖 Agent

`CommitMessageAgent`는 `Agent`만 구현하고 SCM runner가 호출. 채팅에 JSON `commit` 필드를 꽂는 구조가 아님.

---

## 4. 목표 시나리오에 대한 접근법

### 옵션 1 — Tools로 side effect + 자유 텍스트 응답 (권장, Theia 관용)

```text
User → ChatAgent (스트리밍 가능)
     ├─ tool: renameSession / updateSessionTitle
     ├─ tool: gitCommit (또는 기존 SCM 연동)
     └─ assistant markdown = 사용자에게 보일 response
```

- Chat UI는 본문 + (원하면) tool part만 표시
- OpenAI structured+tools 미구현 제약과 **무관**
- Anthropic 등 structured 미지원 모델과도 동작
- 세션 제목은 기존 NamingAgent와 **둘 중 하나만** 쓰거나, tool 호출 시 NamingAgent 결과를 덮어쓰지 않도록 조건 필요

**단점:** “한 번의 JSON blob”이 아니라 text + tool calls 순서에 의존.

### 옵션 2 — `response_format` + `addContentsToResponse` 오버라이드 (원하신 “단일 JSON”에 가장 직접)

제품 `ChatAgent`에서:

1. **`sendLlmRequest` 오버라이드** — 요청에 예:

   ```typescript
   response_format: {
     type: 'json_schema',
     json_schema: {
       name: 'agent_reply',
       strict: true,
       schema: {
         type: 'object',
         properties: {
           response: { type: 'string' },
           session: { type: 'string' },
           commit: { type: 'object', /* … */ }
         },
         required: ['response'],
         additionalProperties: false
       }
     }
   }
   ```

2. **`addContentsToResponse` 오버라이드** — Orchestrator + Terminal 조합:

   ```typescript
   if (isLanguageModelParsedResponse(lmResponse)) {
     const data = lmResponse.parsed as { response: string; session?: string; commit?: … };
     request.response.response.addContent(
       new MarkdownChatResponseContentImpl(data.response)
     );
     if (data.session) {
       await this.chatService.renameSession(request.session.id, data.session);
     }
     if (data.commit) {
       // SCM / git API 호출 — UI content로 넣지 않음
     }
     return;
   }
   // 폴백: getJsonOfResponse / getJsonOfText
   ```

3. (선택) 중간 안내용으로만 `InformationalChatResponseContent` 또는 Progress.

**조건·비용:**

- 모델이 `supportsStructuredOutput` (주로 OpenAI Chat Completions / Vercel 경로)
- **비스트리밍** (스키마 강제 시)
- **같은 요청에 tools와 병행 불가**(OpenAI TODO)
- Anthropic 등에서는 schema 무시 → 프롬프트 JSON + `getJsonOfText` 폴백 필요

### 옵션 3 — 프롬프트만 JSON + 파싱 (Orchestrator/Command 스타일)

`response_format` 없이 시스템 프롬프트로 JSON을 강제하고 `getJsonOfText`로 분리.  
구현은 쉽지만 schema enforcement 없음, 스트림 중 partial JSON이 마크다운으로 깜빡일 수 있음.

### 옵션 4 — `ResponseContentMatcher` / custom content kind

스트림 텍스트 안의 delimiter(코드펜스 등) 분할용.  
“전체 JSON 숨기고 필드만 처리”에는 옵션 2보다 불리.

---

## 5. 스트리밍

| 경로 | 동작 |
|------|------|
| `json_schema` (OpenAI/Vercel) | 완료 후 한 번에 `LanguageModelParsedResponse` |
| 일반 stream | `parseContents`가 버퍼마다 재실행 — **완성 JSON 필드 분리 API 없음** |
| 채팅에서 옵션 2 | 응답이 끝날 때까지 UI가 비거나, 폴백 텍스트 파싱 시 중간 JSON이 보일 수 있음 |

실시간 타이핑감이 중요하면 **옵션 1(tools + 자유 텍스트 스트림)** 이 맞다.

---

## 6. OOTB 갭 (제품에서 메워야 할 것)

1. 채팅 기본 파이프라인에 `response_format` 없음  
2. `LanguageModelParsedResponse.parsed`를 chat agent가 소비하지 않음  
3. `json_object` 타입 — 프로바이더 구현 부재에 가깝음  
4. Anthropic/Google 등 네이티브 structured 없음  
5. OpenAI structured + tools 동시 불가  
6. 스트리밍 structured / incremental schema validation 없음  
7. 모델 JSON → `session`/`commit` 자동 wiring 없음  
8. `supportsStructuredOutput`이 공통 `LanguageModel` 메타데이터가 아님 (프로바이더별)

---

## 7. 제품(`theia-ide`)에서의 위치 제안

AGENTS.md 기준으로 ChatAgent 커스텀은:

- 구현: `theia-extensions/<새확장>/` (또는 기존 product와 분리한 AI 확장)
- 조립: `applications/{browser,electron,electron-next}` dependencies에 패키지 추가
- Upstream 프레임워크 수정이 필요하면 `../theia/`로 보내고, 불가피한 우회만 `patches/`

**바로 쓸 훅:** `AbstractStreamParsingChatAgent` 서브클래스에서 `sendLlmRequest` + `addContentsToResponse` (+ 필요 시 `onResponseComplete`).

세션 제목만이면 JSON 필드보다 **`ChatService.renameSession`** 또는 NamingAgent 커스터마이즈가 더 얇다.  
Commit만이면 **non-chat `CommitMessageAgent` 패턴** 또는 dedicated tool이 structured 필드보다 Theia와 잘 맞는다.

---

## 핵심 파일

| 주제 | 경로 |
|------|------|
| Request / schema / Parsed | `../theia/packages/ai-core/src/common/language-model.ts` |
| JSON helpers | `../theia/packages/ai-core/src/common/language-model-util.ts` |
| Chat invoke / sendLlmRequest / stream parse | `../theia/packages/ai-chat/src/common/chat-agents.ts` |
| Session rename / naming | `../theia/packages/ai-chat/src/common/chat-service.ts`, `chat-session-naming-service.ts` |
| OpenAI structured | `../theia/packages/ai-openai/src/node/openai-language-model.ts` |
| Terminal 레퍼런스 | `../theia/packages/ai-terminal/src/browser/ai-terminal-agent.ts` |
| Orchestrator JSON+숨김 UI | `../theia/packages/ai-ide/src/common/orchestrator-chat-agent.ts` |
| Commit (non-chat) | `../theia/packages/ai-ide/` → `commit-message-agent` |

---

## 결론

**가능하지만 OOTB “JSON 필드 라우팅”은 없다.**

- **단일 structured JSON + UI에 `response`만:** 커스텀 ChatAgent에서 `response_format` + `addContentsToResponse`로 `parsed`를 직접 소비 (Terminal + Orchestrator 패턴 결합). 모델·비스트리밍·tools 병행 제약을 감수.
- **Theia에 더 맞는 분리:** 사용자 문구는 일반(스트리밍) 응답, `session`/`commit` 등은 **Tool**(또는 기존 NamingAgent / CommitMessageAgent)로 처리.
