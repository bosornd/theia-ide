# Theia AI: Local / Client-Side Message Compaction — 요소 기술 조사

조사 기준: 형제 저장소 `../theia/` (특히 `@theia/ai-chat`, `@theia/ai-chat-ui`, `@theia/ai-core`, `@theia/ai-ide`).  
관련: [`message-compaction.md`](./message-compaction.md) (이미 구현된 **server-side** compaction), upstream [#17324](https://github.com/eclipse-theia/theia/issues/17324) (`/compact`), [#17636](https://github.com/eclipse-theia/theia/issues/17636), 실험 브랜치 `feat/budget-aware-chat` (커밋 `803c1ad6a`).

**목적:** 프로바이더에 의존하지 않는 **로컬(클라이언트) compaction**을 검토·구현할 때, 최종 LLM request가 어떻게 만들어지는지, history가 어디에 끼어드는지, 확장에서 무엇을 고칠 수 있는지, 필요한 요소 기술을 정리한다.

---

## 한눈에 보기

| 질문 | 답 |
|------|-----|
| 로컬 compaction이 master에 있나? | **일반 ChatAgent용 in-session compact는 없음** (`isStale` / #17324). Server-side는 병합됨. **Claude Code `/compact`는 제공**되나 CC 런타임에 위임 — [`message-compaction.md` §1.3](./message-compaction.md) |
| 오늘 쓸 수 있는 유사 기능 | (1) Claude Code `/compact` (2) `ChatSessionSummaryAgent` + Task Context → **외부 Summary** → **새 세션** `#taskContext` (Theia hierarchy를 자르지 않음) |
| History → request 변환 핵심 | `AbstractChatAgent.getMessages(session)` → `LanguageModelMessage[]` → `sendLlmRequest` → `LanguageModelService.sendRequest` |
| 확장에서 가장 직접적인 훅 | Agent 서브클래스에서 `getMessages` / `invoke` override, 또는 core에 `isStale`+`insertSummary` 도입 후 서비스 기여 |
| Server-side와 관계 | 공존 전제. Local은 Gemini·Chat Completions·임계치 이전 수동 compact·프로바이더 비지원 모델용 |

---

## 1. 최종 request는 어떻게 구성되는가

### 1.1 호출 스택 (한 턴)

```text
ChatService.sendRequest(sessionId, ChatRequest)
  ├─ ChatRequestParser.parseChatRequest(...)     // @agent, #var, ~tool 파싱
  ├─ session.model.addRequest(parsed, agentId)   // hierarchy에 턴 추가
  └─ agent.invoke(MutableChatRequestModel)
        AbstractChatAgent.invoke:
          1. getLanguageModelForRequest(...)
          2. getSystemMessageDescription(context)  // prompt + variables + tools
          3. getMessages(request.session)          // ★ 전체 세션 history
          4. appendExternalFileChangeNotice(...)   // 읽은 파일 변경 알림 (trailing user msg)
          5. messages.unshift(systemMsg)           // system을 맨 앞
          6. tools 조립 (chat tools + prompt tools + additionalToolRequests)
          7. sendLlmRequest → LanguageModelService.sendRequest(UserRequest)
          8. addContentsToResponse / recordTokenUsage / onResponseComplete
```

핵심 파일: `packages/ai-chat/src/common/chat-service.ts`, `packages/ai-chat/src/common/chat-agents.ts`.

### 1.2 `UserRequest` / `LanguageModelRequest`에 실리는 것

`sendLlmRequest`가 조립하는 객체 (`chat-agents.ts`):

| 필드 | 출처 |
|------|------|
| `messages` | `getMessages` + system unshift + file-change notice |
| `tools` / `deferredToolIds` | request·system prompt·agent `additionalToolRequests` |
| `serverTools` | 요청의 vendor별 selection ∩ 모델이 선언한 server tools |
| `settings` | agent `getLlmSettings()` ⊕ session provider settings |
| `reasoning` | `session.settings.commonSettings.reasoning` |
| `compaction` | `session.settings.commonSettings.compaction` (**server-side only**) |
| `agentId` / `sessionId` / `requestId` / `cancellationToken` | 메타 |
| `clientSettings` | preference requestSettings 병합 (frontend LM service). `keepThinking` / `keepToolCalls`로 메시지 필터 |

`LanguageModelServiceImpl.sendRequest`는 위 request를 받은 뒤 `clientSettings`에 따라 thinking/tool 메시지를 걸러내고 `languageModel.request(...)`로 넘긴다. **history를 요약·자르는 단계는 없다.**

### 1.3 System prompt vs history

- System: `getSystemMessageDescription` → `PromptService.getResolvedPromptFragment(systemPromptId, …, ChatSessionContext)`.
- History: 별도 `getMessages`. 둘이 합쳐진 뒤에야 프로바이더별 변환(OpenAI/Anthropic/…)이 일어난다.
- Context variable(`#file`, `#taskContext` 등)은 **파서/프롬프트 해석** 단계에서 풀리며, `getMessages`가 만드는 user/ai 턴과는 다른 축이다. Task Context 요약이 “새 세션에 주입”되는 방식이 바로 이 축이다.

---

## 2. Message history는 어떻게 포함되는가

### 2.1 `getMessages` 알고리즘 (현재 master)

`AbstractChatAgent.getMessages(model)` (`chat-agents.ts`):

1. `model.getRequests()` — hierarchy의 **active branch** 위 완료된 턴들 (대안 브랜치 제외).
2. 각 request에 대해:
   - **User:** `request.message.parts`의 `promptText` 연결 → `TextMessage` (`actor: 'user'`).
   - **Images:** context variables + inline image parts → `ImageMessage`.
   - **AI response** (complete이거나 `includeResponseInProgress`):
     - `Error` / `Informational` content **제외**.
     - signature 없는 Thinking **제외** (취소된 스트림 방어).
     - `toLanguageModelMessage()` 있으면 그 결과 (tool_use/result, compaction, thinking, …).
     - 없으면 `asString()` / `asDisplayString()` → AI `TextMessage`.
     - `asString() === undefined`면 제외 (의도적 LLM 제외 신호).
3. flat list로 반환. **stale/요약 노드 개념 없음** → 세션이 길수록 매 요청에 전체 transcript가 실림.

Server-side compaction 마커(`CompactionChatResponseContent`)도 여기서 `toLanguageModelMessage()` → `CompactionMessage`로 포함되고, **프로바이더 백엔드**가 replay/prefix-drop을 한다 (로컬이 자르지 않음). 상세는 [`message-compaction.md`](./message-compaction.md).

### 2.2 세션 모델 구조

- `MutableChatModel` + `ChatRequestHierarchy`: 요청이 트리/브랜치. `getRequests()` = active path.
- 각 턴: `MutableChatRequestModel` (user message + `response` content parts).
- 직렬화: `toSerializable` / `ChatContentDeserializerRegistry`로 복원. 커스텀 content kind를 넣으려면 deserializer contribution 필수.

### 2.3 이미 있는 “요약” — 세션을 안 자른다

| 메커니즘 | 동작 | History 영향 |
|----------|------|----------------|
| `ChatSessionSummaryAgent` | purpose `chat-session-summary`, alias `default/summarize` | 임시 `MutableChatModel`에 원본 턴을 복사해 invoke → **별도 Summary 문서** |
| `TaskContextService.summarize` | 위 agent로 요약 생성·저장 | 원본 세션 `getRequests()` **불변** |
| `AI_CHAT_SUMMARIZE_CURRENT_SESSION` | Task Context 요약 | UI에 요약 파일; 같은 세션 prompt는 그대로 |
| “Initiate session with task context” | 새 세션 + `#taskContext` | **새** 세션은 짧은 history + variable; 옛 세션은 그대로 |

즉 현재 “Summarize Current Session”은 **local compaction이 아니라 task-context handoff**다. [#17324](https://github.com/eclipse-theia/theia/issues/17324)도 이 점(같은 세션에 남아야 함 / summarize와 `/compact` 통합)을 지적한다.

---

## 3. Upstream이 그린 Local Compaction 설계 (미병합)

Server-side 이슈(#17636)가 명시한 보완축: *manual `/compact` + client-side summarization은 chat-model 리팩터(`isStale`, summary nodes)가 필요하고, 닫힌 실험(#16742 / `feat/budget-aware-chat`)과 공존 예정.*

### 3.1 모델 변경 제안

| 요소 | 역할 |
|------|------|
| `ChatRequest.kind: 'user' \| 'summary'` | summary 턴 구분 |
| `ChatRequestModel.isStale` | 요약에 흡수된 과거 턴. **`getMessages`에서 skip** |
| `MutableChatModel.insertSummary(callback, 'beforeLast'\|'end')` | 요약 agent 호출 → stale 마킹 → `SummaryChatResponseContent` 삽입 → (옵션) 마지막 user turn 재배치 |
| `SummaryChatResponseContent` (`kind: 'summary'`) | UI용 접힌 요약 노드; `toLanguageModelMessage`로 요약 텍스트를 history에 남김 |

`getMessages` 변경 요지 (실험 브랜치):

```ts
if (request.isStale === true) {
    return []; // 내용이 summary node로 대체됨
}
```

요약 콜백 **중**에는 아직 stale이 아니어야 요약 agent가 전체 history를 본다. stale 마킹은 콜백 성공 후.

### 3.2 오케스트레이션

- `ChatSessionSummarizationService.triggerSummarization(sessionId, skipReorder)`  
  → `model.insertSummary` + `ChatSessionSummaryAgent.invoke`.
- `ChatSessionTokenTracker` + threshold event → 자동 트리거.
- (실험) `ChatLanguageModelServiceImpl`: preference `experimentalBudgetAwareToolLoop` 시 tool loop를 외부에서 돌리고, mid-turn에 summarization (`singleRoundTrip`).

### 3.3 일반 chat `/compact` (#17324) — Claude Code `/compact`와 별개

**“master에 `/compact`가 없다”는 뜻이 아니다.** 이름은 같지만 대상이 다르다.

| | Claude Code `/compact` | #17324 일반 chat `/compact` |
|--|------------------------|------------------------------|
| master | **있음** (`@theia/ai-claude-code`) | **없음** |
| 누가 쓰나 | ClaudeCode agent만 | Coder 등 패널 ChatAgent 일반 |
| 누가 요약하나 | Claude Code 런타임 | Theia (`ChatSessionSummaryAgent` 등) |
| 무엇을 줄이나 | CC `resume` 세션 history | `MutableChatModel` + `getMessages` (`isStale`) |

Claude Code 쪽 처리: [`message-compaction.md` §1.3](./message-compaction.md).

#17324가 요구하는 것(미구현):

- 수동 slash command (일반 chat).
- LLM으로 history 요약 → **같은** 세션에 compact summary 삽입.
- 이전 메시지 stale → 이후 prompt에서 제외.
- Summary / stale 시각 구분.
- Server-side와 역할 분리: 비지원 프로바이더, 임계치 이전 강제, 작업 경계에서 수동 compact.

이 절의 “아직 없음”은 **#17324 설계(§3.1 chat-model 변경 포함)** 를 가리킨다. 구현 시 `isStale` / `insertSummary`가 사실상 전제다.

---

## 4. 확장에서 수정·변경할 수 있는 지점

제품 코드 위치는 `theia-extensions/` 또는 upstream PR. Agent 조립은 [`adding-ai-agents.md`](./adding-ai-agents.md) 참고.

### 4.1 Request / History 축 (로컬 compaction의 핵심)

| 훅 | 방법 | 효과 | 한계 |
|----|------|------|------|
| `getMessages` override | 커스텀 `ChatAgent`가 `AbstractChatAgent` 상속 후 override | 전송 history를 자유롭게 자르기/치환 | **모든** 대상 agent에 적용하려면 각각 서브클래스하거나 공통 base 재바인딩 |
| `invoke` override | Coder처럼 `super.invoke` 전후 처리 | compact 선실행 후 본 요청 | 공통 파이프라인에 끼우기 어려움 |
| `sendLlmRequest` override | messages/tools/settings 최종 조정 | 요청 직전 한 번 더 가공 | history 영속화와 어긋날 수 있음 (UI엔 full, wire엔 truncated) |
| `LanguageModelService` 재바인딩 | `ChatLanguageModelServiceImpl` 패턴 | 전 agent budget/tool-loop | core DI; theia-ide만의 패치면 유지비 큼 |
| Session settings | `commonSettings` 확장 | 세션별 local-compact on/off·threshold | 타입/`session-settings-dialog` 확장 필요 |

**권장 방향 (upstream 정렬):** `isStale` + `insertSummary`를 **chat-model**에 두고, `getMessages`가 stale을 skip하게 한 뒤, 확장/서비스는 “언제 compact할지”만 담당. Agent마다 history를 다르게 자르면 일관성이 깨진다.

### 4.2 요약 생성 축 (이미 확장 가능)

| 훅 | 방법 |
|----|------|
| `ChatSessionSummaryAgent` | prompt variant / `default/summarize` alias / 별도 agent id |
| `TaskContextService.summarize(session, promptId?, agent?)` | prompt·agent 주입 가능 |
| `PromptService` / prompt customization | `chat-session-summary-system` 템플릿 커스터마이즈 |
| CommandContribution | `/compact` 또는 “Compact Session” 커맨드 → 서비스 호출 |

### 4.3 Content / UI / 영속화

| Contribution | 용도 |
|--------------|------|
| `ChatContentDeserializerContribution` | `kind: 'summary'` (또는 제품 전용 kind) 복원 |
| `ChatResponsePartRenderer` | 접힌 요약·stale 스타일 |
| `ResponseContentMatcherProvider` | 스트림 텍스트에서 커스텀 블록 파싱 (요약 agent가 structured면) |
| Session store | `isStale` / `kind` 직렬화 필드 (model 변경과 세트) |

### 4.4 가벼운 대안 (core 변경 없이 제품에서)

1. **Handoff 패턴 (현재):** summarize → 새 세션 + `#taskContext`. UX는 “세션 교체”.
2. **Agent-local truncation:** §4.5.
3. **Informational + asString undefined:** UI에만 보이는 안내를 넣고 LLM에는 안 보냄. 과거 턴을 빼지는 못함.
4. **Server-side 우선 + local fallback:** 지원 모델은 기존 preference; 비지원만 agent/service에서 truncate.

Claude Code 패키지의 `/compact` slash는 **Claude Code 런타임**으로 전달되는 명령 등록이지, Theia chat-model local compaction이 아니다. 처리 흐름은 [`message-compaction.md` §1.3](./message-compaction.md).

### 4.5 Agent-local truncation — 구체 절차

**정의:** chat-model/`isStale`을 건드리지 않고, **특정 ChatAgent**가 LLM에 넘기는 `LanguageModelMessage[]`만 줄이는 request shaping. UI·세션 저장에는 전체 턴이 남는다.

**적용 가능 범위:** `AbstractChatAgent` / `AbstractStreamParsingChatAgent` / `AbstractModeAwareChatAgent` 계열 (Coder, Architect, …).  
**제외:** `ClaudeCodeChatAgent` — `getMessages`를 쓰지 않고 SDK `query`+`resume`만 사용 (§위 Claude Code 경로).

#### 4.5.1 끼울 위치 (권장: `getMessages` override)

`AbstractChatAgent.invoke` 순서:

```text
getMessages(session)          ← ★ 여기서 truncate
appendExternalFileChangeNotice
messages.unshift(systemMsg)   ← system은 truncate 이후에 붙음 (보존됨)
sendLlmRequest(messages, …)
```

`getMessages`는 `protected`이므로 서브클래스에서 override 가능. system prompt·file-change notice·tools는 기존 경로 그대로다.

```ts
// 스케치 — 제품 확장에서 CoderAgent 등을 상속할 때
protected override async getMessages(
    model: ChatModel,
    includeResponseInProgress = false
): Promise<LanguageModelMessage[]> {
    const full = await super.getMessages(model, includeResponseInProgress);
    return this.shapeHistory(full, model);
}
```

대안: `invoke` 전체를 감싸 `super` 전에 손대지 말고, `sendLlmRequest` override로 messages를 가공하는 방법도 있으나, history 정책은 `getMessages`에 모으는 편이 테스트·재사용에 유리하다.

#### 4.5.2 단위: message 슬라이스가 아니라 **request(턴) 단위**

`getMessages`는 `model.getRequests().flatMap(...)`으로 **턴 → 여러 LanguageModelMessage**를 만든다 (user text, images, thinking, tool_use/result, compaction, ai text…).

잘못된 방식: `full.slice(-N)`으로 메시지 N개만 자르기 → 턴 한가운데·`tool_use`/`tool_result` 짝이 깨져 프로바이더가 거부하기 쉽다.

권장:

1. `const requests = model.getRequests()` (active branch).
2. 유지할 턴: `requests.slice(-keepLastTurns)` (예: 최근 6~12 request). 현재 진행 중인 마지막 request는 항상 포함.
3. 유지 턴만 다시 `super.getMessages`와 동일한 규칙으로 직렬화하거나, `super.getMessages` 결과에서 **턴 경계 인덱스**를 매핑해 슬라이스.
   - 가장 단순한 PoC: 임시로 “최근 K request만 있는 뷰”를 만들지 말고, `super.getMessages` 후 **앞에서부터 drop할 request 개수에 대응하는 message 구간**을 계산하기보다, **request 리스트를 자른 뒤 그 subset에 대해 직렬화 헬퍼를 호출**하는 쪽이 안전하다.
   - 실용 PoC: subclass에서 `getMessages`를 복사하지 않고,

```ts
protected override async getMessages(model: ChatModel, includeResponseInProgress = false) {
    const requests = model.getRequests();
    if (requests.length <= this.keepLastTurns) {
        return super.getMessages(model, includeResponseInProgress);
    }
    // MutableChatModel에 "window" API는 없으므로:
    // - 옵션 A: dropped prefix를 무시하고 recent requests만 직렬화하는 private helper 복제
    // - 옵션 B: super.getMessages 전체 수신 후, request request들의 promptText/응답을
    //           마커로 쓰지 말고 "턴 개수" 기준으로 message 그룹을 재구성
    return this.serializeRequests(requests.slice(-this.keepLastTurns), includeResponseInProgress);
}
```

`serializeRequests`는 사실상 `AbstractChatAgent.getMessages`의 flatMap 본문과 동일 규칙이어야 한다 (Error/Informational 제외, signature 없는 Thinking 제외, `toLanguageModelMessage` 우선 등). upstream 변경 없이 PoC할 때는 **해당 블록을 헬퍼로 복사**하거나, 제품에서 `AbstractChatAgent`를 감싼 base agent를 한 곳에 둔다.

#### 4.5.3 잘라낸 prefix를 어떻게 할지 (3단계)

| 단계 | 동작 | 난이도 | 효과 |
|------|------|--------|------|
| **T0 Truncate-only** | 오래된 턴 drop, 대체 메시지 없음 | 낮음 | 토큰↓, 초기 요구사항 망각 |
| **T1 Static stub** | drop 구간에 user/system 한 줄 삽입: “Earlier turns omitted (N requests).” | 낮음 | 모델이 잘림을 인지 |
| **T2 Summary stub** | drop된 턴만 모아 summarize LLM 1회 → 요약 `TextMessage`를 recent window **앞**에 삽입 | 중 | handoff에 가까운 품질, 매 턴 비용↑ |

T2 시 재사용 후보:

- `ChatSessionSummaryAgent` + `default/summarize` (세션 전체를 넣는 `TaskContextService.getLlmSummary`와 달리, **dropped requests만** 담은 임시 `MutableChatModel`을 만들어 invoke).
- 또는 `LanguageModelService.sendRequest`로 짧은 summarize 전용 request (agent UI에 안 남김).
- 캐시: `session.id` + “마지막 compact된 request id”를 키로 요약 문자열을 세션 `addData`/weak map에 보관해, 매 턴 재요약하지 않기.

T2를 매 토큰 threshold마다 돌리면 Task Context summarize와 비용 특성이 비슷해진다. PoC는 T0/T1로 시작하고, 품질이 필요할 때만 T2.

#### 4.5.4 반드시 지킬 제약

1. **Tool pair 무결성** — 턴 단위 절단이면 보통 안전. message 단위 절단 금지. 자른 window의 첫 메시지가 `tool_result`이면 한 턴 더 앞으로 확장하거나 해당 고아 result를 drop.
2. **Thinking signature** — 기존 `getMessages`가 signature 없는 thinking을 이미 제거. truncate 헬퍼도 동일 필터 유지 (Anthropic 거부 방지).
3. **Server-side compaction marker** — window 밖에 남은 `CompactionMessage`는 재전송되지 않음. OpenAI prefix-drop 전제와 충돌할 수 있으므로, **server-side compaction이 이미 돈 세션**에서는 agent-local truncate를 끄거나, window에 최신 compaction 마커가 있으면 그걸 window 시작으로 삼기.
4. **이미지** — 최근 턴에만 큰 base64가 있어도 토큰·페이로드가 큼. 필요 시 오래된 이미지 턴만 추가 drop.
5. **UI 불일치** — 사용자는 전체 transcript를 보지만 모델은 일부만 봄. PoC라도 stub(T1) 또는 Informational content로 “요청에 최근 N턴만 포함”을 알리는 편이 디버깅에 유리 (`asString() === undefined`면 LLM에는 안 감).
6. **멀티 agent 세션** — 같은 `ChatModel`을 Coder→Architect가 이어받으면, truncate 정책이 agent마다 달라져 컨텍스트가 출렁인다. 세션 pin agent 기준으로만 켜기.
7. **Claude Code** — 이 패턴 적용 대상 아님.

#### 4.5.5 트리거

| 트리거 | 구현 아이디어 |
|--------|----------------|
| 고정 N턴 | `keepLastTurns` 상수/preference |
| 대략 토큰 | `request.response`에 쌓인 `tokenUsageEntries` 합 또는 `TokenUsageService`; threshold 넘으면 N을 줄이거나 T2 1회 |
| `maxInputTokens` | `languageModel.maxInputTokens`의 일정 비율 (정확 추정기 없음 → usage 기반이 현실적) |
| 수동 | 커맨드가 세션 data에 `forceKeepLastTurns` 기록 → 다음 `getMessages`가 반영 |

자동 임계치는 server-side compaction preference와 **겹치지 않게** (지원 모델은 server-side에 맡기고, Gemini/Chat Completions 등만 local truncate).

#### 4.5.6 제품(`theia-ide`)에 넣는 방법

1. `theia-extensions/<name>/`에 agent subclass (예: `TruncatingCoderAgent extends CoderAgent`).
2. Frontend module:

```ts
bind(TruncatingCoderAgent).toSelf().inSingletonScope();
rebind(CoderAgent).toService(TruncatingCoderAgent);
// ChatAgent / Agent 바인딩은 ai-ide가 CoderAgent 심볼로 toService 하므로
// rebind(CoderAgent)만으로 ChatAgent 쪽도 새 구현을 보게 되는지 확인.
// 안 되면 ChatAgentService 등록 경로를 따라 Agent·ChatAgent도 재바인딩.
```

`@theia/ai-ide`는 `bind(CoderAgent).toSelf()` 후 `bind(ChatAgent).toService(CoderAgent)` 패턴이다. 제품 모듈 로드 순서가 ai-ide **이후**여야 `rebind`가 먹는다.

3. applications `package.json` dependencies에 해당 확장 추가.
4. **upstream 미수정 PoC**면 Coder만; 전 agent 공통화는 shared `TruncatingChatAgentMixin` / base class를 확장 패키지에 둔다.
5. 장기적으로는 #17324 chat-model 경로로 옮기고 이 override는 제거하는 전제.

#### 4.5.7 최소 PoC 체크리스트

- [ ] Coder(또는 단일 agent) subclass + `getMessages` override
- [ ] 턴 단위 `keepLastTurns` (T0)
- [ ] tool_use/result·thinking 필터가 super와 동일
- [ ] 긴 세션에서 API 4xx(context length / tool pairing) 사라지는지
- [ ] UI에는 옛 턴이 그대로인지 (의도 확인)
- [ ] server-side compaction on인 Anthropic/OpenAI 세션과 동시 사용 시 이상 없는지
- [ ] Claude Code 세션에는 영향 없음

#### 4.5.8 #17324 / handoff와의 관계

| | Agent-local truncation | Task Context handoff | #17324 in-session compact |
|--|------------------------|----------------------|---------------------------|
| UI history | 유지 | 새 세션 | 유지 + stale 표시 |
| Wire history | 매 요청마다 축소 | 새 세션은 짧음 | 영구 축소 (stale) |
| 요약 | 선택(T2) | 파일로 저장 | 세션 summary node |
| core 변경 | 불필요 | 불필요 | 필요 |

Agent-local truncation은 **검증·임시 완화**용이다. 제품 기본 동작으로 굳히기 전에 UI/wire 불일치와 agent별 정책 분산 비용을 감수할지 정한다.

---

## 5. Local Compaction에 필요한 요소 기술

구현을 분해하면 아래 레이어가 필요하다. 괄호는 현재 상태.

### A. History 정책 (chat-model) — **미구현 / 필수**

- Stale(또는 equivalent) 플래그와 직렬화.
- Summary node 삽입 API (`insertSummary` 수준).
- `getMessages`가 stale을 제외하고 summary content를 포함.
- (선택) 브랜치/대안 요청과의 stale 전파 규칙.

### B. 요약 엔진 — **부분 구현**

- `ChatSessionSummaryAgent` + `CHAT_SESSION_SUMMARY_PROMPT` + `default/summarize` alias → **재사용 가능**.
- Compact 전용 prompt (작업 상태·미완 tool·URI 보존) 분리 권장.
- 실패 시 rollback (실험 브랜치의 summary request 제거 패턴).

### C. 트리거 — **미구현 (이슈/실험만)**

| 트리거 | 설명 |
|--------|------|
| Manual `/compact` | #17324 |
| Token threshold | 누적 input ≈ `maxInputTokens` 비율; `TokenUsageService` / session indicator는 server-side 이후 일부 존재 |
| Mid-tool-loop | `singleRoundTrip` + 외부 tool loop (실험 브랜치; Anthropic 중심) |
| Preference | global / per-session; server-side compaction 설정과 **직교**하게 둘 것 |

토큰 추정: 응답 usage 누적이 현실적. 요청 **전** 정확한 token count API는 모델·프로바이더마다 다르며 master에 generic estimator는 없다. `LanguageModelMetaData.maxInputTokens`는 threshold 분모로 쓸 수 있다.

### D. Request 조립 연동 — **작은 변경**

- `getMessages` stale skip만으로 대부분 충족.
- System prompt는 유지 (요약에 “system도 고려”하라고 이미 summary prompt에 명시).
- Tool call 연속성: mid-turn compact 시 pending tool_use/result를 요약 밖으로 보존해야 함 (실험 브랜치 `skipReorder` / `beforeLast`).

### E. UI / UX

- Summary 마커 (server-side `CompactionPartRenderer`와 구분: 로컬은 보통 **읽기 가능한** 요약).
- Stale 메시지 접기·흐리게.
- Token indicator에 “locally compacted” 카운트 (선택).
- Server-side와 동시 활성 시 이중 압축·혼란 방지 카피/설정.

### F. 프로바이더 중립성

- Local compact 결과는 일반 `TextMessage`(또는 summary content → text)여야 함. Opaque `CompactionMessage`는 특정 프로바이더 전용.
- 모델 전환 후에도 유효해야 함 (server-side opaque marker의 약점 보완).

### G. 제품(`theia-ide`) 배치

| 선택 | 위치 |
|------|------|
| Upstream 기여 | `../theia/` `@theia/ai-chat` (+ ui) — **권장** (이슈 #17324와 정렬) |
| 제품 전용 PoC | `theia-extensions/`에 agent wrapper + command; history 정책은 약함 |
| 설정만 | server-side on + summarize handoff 문서화 |

---

## 6. 구현 옵션 비교

```text
옵션 1 — Upstream local compact (isStale + /compact)
  장점: 전 agent·전 프로바이더, 세션 유지, #17324와 일치
  단점: chat-model/UI/deser 변경 범위 큼

옵션 2 — 실험 브랜치 재도입 (budget-aware + auto)
  장점: 자동·tool-loop까지 설계됨
  단점: Anthropic/`singleRoundTrip` 의존, hard-coded 200k; server-side와 재조정 필요

옵션 3 — Agent getMessages truncation (제품 확장)
  장점: 빠른 PoC, core PR 불필요
  단점: agent마다 중복, 세션 UI/저장과 wire history 불일치, 재사용성 낮음

옵션 4 — Task Context handoff만 강화
  장점: 이미 동작
  단점: 같은 세션 compaction이 아님; 토큰 절감은 “새 세션”에서만
```

**실무적으로:** 장기 SSOT는 옵션 1. PoC는 옵션 3으로 wire 절감만 검증 후, 옵션 1로 모델 변경을 upstream에 올리는 순서가 안전하다. Server-side와는 설정·트리거를 분리해 공존시킨다.

---

## 7. 권장 조사/구현 순서 (로컬 compact 착수 시)

1. `AbstractChatAgent.invoke` / `getMessages` / `sendLlmRequest`를 한 턴 breakpoint로 확인 (이 문서 §1–2).
2. `TaskContextService.getLlmSummary`로 요약 agent 입출력을 재사용해 품질 baseline 확보.
3. `#17324` + `feat/budget-aware-chat`의 `insertSummary`/`isStale` 패치를 스펙 초안으로 채택할지 ADR.
4. chat-model 변경 → `getMessages` skip → deserializer/renderer → `/compact` command → (이후) threshold 자동화.
5. Server-side와 동시 사용 시나리오 검증 (OpenAI Responses + local `/compact`).

---

## 8. 주요 소스 맵

| 관심사 | 경로 |
|--------|------|
| Request 진입 | `../theia/packages/ai-chat/src/common/chat-service.ts` |
| invoke / getMessages / sendLlmRequest | `../theia/packages/ai-chat/src/common/chat-agents.ts` |
| Session·hierarchy·content | `../theia/packages/ai-chat/src/common/chat-model.ts` |
| LM send + clientSettings filter | `../theia/packages/ai-core/src/common/language-model-service.ts` |
| Summary agent / prompt | `../theia/packages/ai-chat/src/common/chat-session-summary-agent*.ts` |
| Task context summarize | `../theia/packages/ai-chat/src/browser/task-context-service.ts` |
| Summarize UI command | `../theia/packages/ai-chat-ui/src/browser/ai-chat-ui-contribution.ts` |
| Server-side compaction | [`message-compaction.md`](./message-compaction.md) |
| 실험 local summarization | `git show 803c1ad6a` / branch `origin/feat/budget-aware-chat` |
| Agent 확장 가이드 | [`adding-ai-agents.md`](./adding-ai-agents.md) |
