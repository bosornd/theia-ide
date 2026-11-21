# Theia AI: Structured JSON Output → Chat UI 분리·Side Effect 처리

조사 기준: 형제 저장소 `../theia/` **v1.77.0** (`@theia/ai-core`, `@theia/ai-chat`, `@theia/ai-ide`, `@theia/ai-openai`, `@theia/ai-terminal` 등).  
관련: [adding-ai-agents.md](./adding-ai-agents.md).

**목표 시나리오:** LLM이 `{ answer, session?, commit?, … }` JSON을 내고, 채팅 UI에는 `answer`만 보이며 나머지 필드는 제품 코드가 처리(세션 제목 변경, git commit 등).

**이 문서의 쓰임:** Theia 1.77 제약과 선택지를 정리한 **설계 전 참고**다. prompt-only JSON 경로를 택할 경우 §8을, history에 `compaction`을 둘 경우 [`local-message-compaction.md` §9](./local-message-compaction.md)를 함께 본다.

### 용어 (`response` vs `answer`)

| 이름 | 층 | 의미 |
|------|-----|------|
| `LanguageModelResponse` | LM | `Text` / `Stream` / `Parsed` 합타입. Parsed는 `parsed` + **`content`** 문자열 |
| `request.response` | Chat | `ChatResponseModel` (상태·취소·토큰 등 메타) |
| `request.response.response` | Chat UI 본문 | `ChatResponse` → `content: ChatResponseContent[]`에 Markdown 등을 `addContent(s)` |
| `response_format` | LM 요청 | schema 강제 API 필드 (OpenAI 용어와 동일) |
| **`answer`** | **제품 schema 제안** | Theia 1급 타입이 **아님**. 옵션 2에서 UI에 넣을 본문 필드로 쓰는 예시 이름 |

프레임워크는 사용자 본문을 `answer`라고 부르지 않는다. UI에 붙일 때는 항상 `request.response.response.addContent(...)`다.

---

## 한눈에 보기

| 질문 | 답 |
|------|----|
| Schema 강제 API | `LanguageModelRequest.response_format` → `{ type: 'json_schema', json_schema: { name, schema, strict? } }` |
| 파싱 결과 타입 | `LanguageModelParsedResponse` (`parsed` + `content`) |
| 채팅 기본 경로 | **`response_format`을 넘기지 않음**. `parsed`도 쓰지 않고 `content` 문자열을 Markdown으로 넣음 → **JSON 전체가 UI에 노출될 수 있음** |
| UI에 본문만 보이게 | `addContentsToResponse` 오버라이드에서 `parsed.answer`(제품 필드)만 `MarkdownChatResponseContent`로 추가 |
| Side effect | 같은 오버라이드에서 필드 처리 **또는** (권장) **Tool**로 rename/commit 등 수행 |
| 세션 제목 | 이미 `ChatSessionNamingAgent` + `ChatService.renameSession` 존재. JSON `session` 필드와 **역할 중복 주의** |
| 스트리밍 + schema | OpenAI structured 경로는 **비스트리밍·Chat Completions `parse`만**. 공식 모델 기본 **Response API**에서는 schema **미연결**. 채팅에 점진 JSON 필드 분리 API 없음 |
| 옵션 3 설계 포인트 | 스트림 **완료 후** trailing text만 파싱. usage 토큰을 세그먼트 경계로 쓰지 말 것 (§8) |

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

- `getTextOfResponse` — parsed면 `content` 문자열; Text면 `text`; Stream/`LanguageModelMonitoredStreamResponse`면 parts 합침
- `getJsonOfResponse` / `getJsonOfText` — fenced JSON 또는 `{`/`[` 시작 텍스트 파싱 (프롬프트만으로 JSON을 받을 때 폴백)

**별도 `structuredOutput` / `outputSchema` 이름은 없다.** 모두 `response_format`이다.

`LanguageModelRequest`에는 1.77에서 `deferredToolIds`, `serverTools`, `reasoning`, `compaction` 등이 추가됐지만, **채팅 `sendLlmRequest`는 여전히 `response_format`을 넣지 않는다.**

### 프로바이더 지원

| 프로바이더 | `json_schema` | 비고 |
|-----------|---------------|------|
| OpenAI (`ai-openai`) | ⚠️ Chat Completions만 | `supportsStructuredOutput` + **`useResponseApi === false`** 일 때 `chat.completions.parse`. tools+structured **미구현**(TODO). **공식 모델 기본 `useResponseApi: true`** → Response API 경로로 가며 **`response_format` 미전달·미연결** (실패 시 Completions 폴백은 가능하나 schema 경로를 보장하지 않음) |
| Ollama | ⚠️ `format: 'json'`만 | **요청 schema 본문은 전달 안 됨** (파싱 후 `LanguageModelParsedResponse` 반환) |
| Anthropic / Google / HF / Copilot | ❌ | 네이티브 패키지에 `response_format` 처리 없음 |
| ~~Vercel AI~~ | — | **`@theia/ai-vercel-ai`는 1.77에서 제거됨** (`generateObject` 경로 없음) |

---

## 2. 채팅 파이프라인이 structured를 “무시”하는 지점

### `sendLlmRequest` — format 미전달

`AbstractChatAgent.sendLlmRequest` (`ai-chat/.../chat-agents.ts`)는 messages / tools / `deferredToolIds` / `serverTools` / settings / `reasoning` / `compaction` 등을 넣지만 **`response_format`은 넣지 않는다.**  
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
“제품 필드 `answer`만 UI”는 **반드시 agent 쪽에서 가로채야** 한다.

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
다만 공식 OpenAI 모델이 Response API를 쓰면 schema가 적용되지 않을 수 있어, 실무에서는 **텍스트 JSON + `getJsonOfResponse` 폴백**에 더 의존할 수 있다.

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
사용자에게 보일 설명은 **assistant 마크다운 본문**, 동작은 **tool call**. Coder/Universal과 가장 잘 맞음.

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

### 옵션 1 — Tools로 side effect + 자유 텍스트 본문 (권장, Theia 관용)

```text
User → ChatAgent (스트리밍 가능)
     ├─ tool: renameSession / updateSessionTitle
     ├─ tool: gitCommit (또는 기존 SCM 연동)
     └─ assistant markdown = 사용자에게 보일 본문
         (프레임워크 이름: ChatResponse content — 제품 schema의 `answer`와 무관)
```

- Chat UI는 본문 + (원하면) tool part만 표시
- OpenAI structured+tools 미구현·Response API schema 미연결 제약과 **무관**
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
           answer: { type: 'string' },   // 제품 필드 예시 이름
           session: { type: 'string' },
           commit: { type: 'object', /* … */ }
         },
         required: ['answer'],
         additionalProperties: false
       }
     }
   }
   ```

2. **`addContentsToResponse` 오버라이드** — Orchestrator + Terminal 조합:

   ```typescript
   if (isLanguageModelParsedResponse(lmResponse)) {
     const data = lmResponse.parsed as { answer: string; session?: string; commit?: … };
     request.response.response.addContent(
       new MarkdownChatResponseContentImpl(data.answer)
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

- 모델이 `supportsStructuredOutput`이고, OpenAI면 **Chat Completions 경로**(`useResponseApi: false` 또는 Response API 실패 후 폴백)여야 schema가 실제 적용됨
- **비스트리밍** (스키마 강제 시)
- **같은 요청에 tools와 병행 불가**(OpenAI TODO)
- Anthropic 등에서는 schema 무시 → 프롬프트 JSON + `getJsonOfText` 폴백 필요
- 공식 OpenAI 기본(Response API)에서는 Terminal처럼 **폴백 파싱**을 전제하는 편이 안전

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
| `json_schema` (OpenAI Completions `parse`) | 완료 후 한 번에 `LanguageModelParsedResponse` |
| OpenAI Response API (공식 기본) | structured **미연결** — 일반 stream/text와 동일하게 취급될 수 있음 |
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
6. OpenAI **Response API ↔ `response_format` 미연결** (공식 모델 기본 경로)  
7. 스트리밍 structured / incremental schema validation 없음  
8. 모델 JSON → `session`/`commit` 자동 wiring 없음  
9. `supportsStructuredOutput`이 공통 `LanguageModel` 메타데이터가 아님 (프로바이더별)

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
| OpenAI Completions structured | `../theia/packages/ai-openai/src/node/openai-language-model.ts` (`handleStructuredOutputRequest`) |
| OpenAI Response API (schema 없음) | `../theia/packages/ai-openai/src/node/openai-response-api-utils.ts` |
| OpenAI `useResponseApi` 기본 | `../theia/packages/ai-openai/src/common/openai-preferences.ts` (`USE_RESPONSE_API_PREF`, default `true`) |
| Terminal 레퍼런스 | `../theia/packages/ai-terminal/src/browser/ai-terminal-agent.ts` |
| Orchestrator JSON+숨김 UI | `../theia/packages/ai-ide/src/common/orchestrator-chat-agent.ts` |
| Commit (non-chat) | `../theia/packages/ai-ide/` → `commit-message-agent` |

---

## 결론

**가능하지만 OOTB “JSON 필드 라우팅”은 없다.** (1.76 → 1.77에서도 이 결론은 동일. Vercel 제거·Response API 기본화로 OpenAI structured 실효 경로만 더 좁아짐.)

- **단일 structured JSON + UI에 제품 필드 `answer`만:** 커스텀 ChatAgent에서 `response_format` + `addContentsToResponse`로 `parsed`를 직접 소비 (Terminal + Orchestrator 패턴 결합). Completions 전제·비스트리밍·tools 병행 제약을 감수하고, Response API 모델에서는 폴백 파싱을 준비.
- **Theia에 더 맞는 분리:** 사용자 문구는 일반(스트리밍) **ChatResponse 본문**, `session`/`commit` 등은 **Tool**(또는 기존 NamingAgent / CommitMessageAgent)로 처리.
- **스키마 강제 없이 단일 JSON 계약을 유지하려면:** 옵션 3 + §8의 스트림·세그먼트·side effect 경계를 설계에 명시한다.

---

## 8. 설계 가이드 — 옵션 3 (prompt-only JSON)

`response_format`과 structured tool call에 의존하지 않고, 시스템 프롬프트로 최종 JSON object를 요구한 뒤 **응답 완료 텍스트를 파싱**하는 경로다. 제공자 호환성은 높지만 schema 강제가 없고, Theia 기본 스트림 루프를 그대로 두면 파싱이 조용히 스킵되기 쉽다.

### 8.1 권장 책임 분리

| 책임 | 권장 경계 |
|------|-----------|
| 최종 텍스트에서 JSON 추출·검증 | Agent와 분리된 순수 파서 (`answer: string` 필수, optional 필드는 타입 가드) |
| UI 본문 | 파싱 성공 시 `answer`만 `MarkdownChatResponseContent`. 실패 시 기존 `parseContents` Markdown 폴백 |
| `session` / `commit` 등 side effect | 파서와 분리된 어댑터. 실패해도 `answer` 표시는 유지하고 오류만 별도 수집 |
| `compaction` | 별도 ChatResponse content + `getMessages` shaping ([`local-message-compaction.md` §9](./local-message-compaction.md)) |

ChatAgent의 훅은 `addContentsToResponse`(및 스트림 소비 override)다. 글로벌 response transformer는 없다 (§2).

### 8.2 스트림 설계 (세그먼트 모델)

OpenAI 등 tool loop는 **한 AsyncIterable** 안에 text / tool_call / thinking / usage를 섞어 보낸다. upstream `AbstractStreamParsingChatAgent.addStreamResponse`는 non-text마다 텍스트 버퍼를 비운다. 옵션 3을 붙일 때는 다음을 전제로 설계한다.

```text
스트림 토큰
  ├─ text          → trailing text 버퍼에 누적 (중간 UI는 임시 Markdown 가능)
  ├─ usage         → 토큰 사용량만 기록. 세그먼트 경계로 취급하지 않음 ★
  ├─ tool/thinking → content로 보존. 여기서 trailing text 세그먼트를 리셋
  └─ 스트림 종료
        ├─ trailing text 없음 → structured 파싱 없음 (예: tool로 끝난 중간 턴)
        └─ trailing text 있음 → 그 세그먼트만 JSON 후보
              성공 → 앞선 tool/thinking content 유지 + answer(및 side effect)
              실패 → 일반 Markdown 폴백
```

비스트림(`text` / `content` / Parsed의 문자열)도 **같은 파서·같은 UI/side-effect 경계**로 모은다.

### 8.3 설계에서 미리 막을 위험

| 위험 | 왜 생기는가 | 설계 시 규칙 |
|------|-------------|--------------|
| 유효 JSON인데 UI에 raw JSON이 남고 side effect 미실행 | 최종 텍스트 **뒤**에 `usage`가 온다. “모든 non-text에서 버퍼 clear”를 복사하면 종료 시 후보 문자열이 비어 파싱이 스킵된다 | **usage(및 빈 content 배열)는 세그먼트 경계가 아니다.** content-bearing non-text만 경계를 연다 |
| tool loop 턴에서 최종 JSON이 무시됨 | “스트림에 tool이 한 번이라도 있으면 structured 금지”로 가드하면 같은 스트림의 **마지막 텍스트 세그먼트**까지 버린다 | **trailing text만** 파싱. 중간 tool content는 세그먼트 시작 인덱스 앞에 보존 |
| progress + JSON을 한 번에 `JSON.parse` | 실행 중 일반 텍스트와 최종 object가 한 버퍼에 섞임 | 파싱 후보는 **마지막 세그먼트**. fenced 또는 `{`…`}` 추출 폴백을 명시 |
| side effect 실패 시 본문도 안 보임 | 파싱과 side effect를 한 트랜잭션처럼 묶음 | 파싱 성공 후 `answer`는 항상 UI에 추가. side effect 오류는 독립 기록(예: blockquote) |
| commit이 SCM/권한과 어긋남 | shell/git 프로세스를 ChatAgent에서 직접 실행 | Theia SCM (`input.value` + `acceptInputCommand`) 등 기존 경계 (`docs/theia/scm-commit-adapter.md`) |
| 재시도·재시작 시 중복 commit | 실행 상태 미영속 | 제품 로컬 저장소에 `in-flight` / `succeeded` / `failed`와 요청·repo·message 키 |
| 부분 스트림에서 session/commit 실행 | 미완성 JSON을 구조화 성공으로 취급 | **스트림 완료 후에만** side effect. 부분 파싱 금지 |

### 8.4 UI·스트리밍 UX 트레이드오프

- 완료 전 `parseContents`로 임시 본문을 그리면 partial JSON이 잠깐 보일 수 있다. 완료 후 `answer`로 교체하는 전제를 스펙에 적는다.
- 완료 전 UI를 비우면 깜빡임은 없으나 체감 지연이 커진다. 둘 중 하나를 ADR/스펙에 고른다.
- Theia 스트림 계약에 명시적 completion 파트가 없으면, iterator 종료를 완료로 보게 된다. 조기 종료 감지 필요 여부를 설계 단계에서 결정한다.

### 8.5 설계·검증 체크리스트 (구현 착수 전/직후)

- [ ] stream / text / Parsed 문자열 경로가 **동일 파서·동일 UI 규칙**을 쓰는가.
- [ ] synthetic 스트림 `text → usage`에서 파싱이 수행되는가 (usage가 버퍼를 지우지 않는가).
- [ ] synthetic 스트림 `tool → text(JSON)`에서 tool UI는 유지되고 본문만 `answer`인가.
- [ ] malformed / `answer` 없는 object는 일반 Markdown 폴백인가.
- [ ] UI에 `session` / `commit` / `compaction` raw 필드가 노출되지 않는가.
- [ ] `compaction`을 쓸 경우 [`local-message-compaction.md` §9](./local-message-compaction.md)의 wire 순서를 스펙에 적었는가.
- [ ] commit 멱등·SCM 어댑터·NamingAgent와의 역할 중복을 ADR에 적었는가.
