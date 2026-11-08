# getSkillFileContent에 상대 path 파라미터 추가

조사 기준: 형제 저장소 `../theia/` (`@theia/ai-ide` `skill-file-functions.ts`, `@theia/ai-core` Skill 모델).  
관련: [shipping-ai-skills.md](./shipping-ai-skills.md), Agent Skills spec ([agentskills.io/specification](https://agentskills.io/specification)).

**목표 API (제품 기대):**

```text
getSkillFileContent(skillName, path = "SKILL.md")
```

| 호출 | 결과 |
|------|------|
| `getSkillFileContent("sdd")` | `…/sdd/SKILL.md` 본문 |
| `getSkillFileContent("sdd", "SKILL.md")` / `"./SKILL.md"` | 동일 |
| `getSkillFileContent("sdd", "workflow/01-prd.md")` | `…/sdd/workflow/01-prd.md` 원문 |

---

## 한눈에 보기

| 질문 | 답 |
|------|----|
| 지금 동작 | `skillName`만 받음. 항상 `Skill.location`(절대경로 **SKILL.md**)을 읽고 frontmatter를 벗긴 **본문**만 반환 |
| 왜 부족한가 | Agent Skills·이 제품 `sdd`는 `workflow/*.md` 등 보조 파일을 스킬 루트 상대경로로 참조. tool이 SKILL.md만 열어 progressive disclosure가 막힘 |
| 어디를 고치나 | 제품 **DI `rebind(GetSkillFileContent)`**(upstream 소스 불필요, 전역·기존 호환 상위집합) / 또는 upstream / `patches/` |
| 핵심 설계 | optional `path`(기본 `SKILL.md`) → 스킬 **디렉터리** 기준 resolve → `..`/절대경로 거부 → SKILL.md만 `parseSkillFile`, 그 외는 raw |

---

## 1. As-is

### 구현 위치

| 경로 (`../theia/`) | 역할 |
|--------------------|------|
| `packages/ai-ide/src/browser/skill-file-functions.ts` | `GetSkillFileContent` ToolProvider |
| `packages/ai-ide/src/common/workspace-functions.ts` | `GET_SKILL_FILE_CONTENT_FUNCTION_ID = 'getSkillFileContent'` |
| `packages/ai-ide/src/browser/frontend-module.ts` | `bindToolProvider(GetSkillFileContent, bind)` |
| `packages/ai-core/src/common/skill.ts` | `Skill.location` = SKILL.md 절대경로; `SKILL_FILE_NAME`; `parseSkillFile` |
| `packages/ai-core/src/browser/skills-variable-contribution.ts` | `{{skill:name}}`도 동일하게 location만 읽고 parse |

등록된 tool schema (요약):

```typescript
parameters: {
  type: 'object',
  properties: {
    skillName: { type: 'string', description: '…' }
  },
  required: ['skillName']
}
```

handler 요약:

1. `JSON.parse` → `skillName`
2. `skillService.getSkill(skillName)` (qualifiedName 우선, 없으면 plain name)
3. `FileService.read(URI.fromFilePath(skill.location))`
4. `parseSkillFile(…).content` 반환 (성공 시 **plain string**; 실패 시 `JSON.stringify({ error })`)

전용 unit test는 없음 (`skill-file-functions` spec 부재).

### Skill 모델이 “폴더”가 아니라 “SKILL.md 파일”을 가리킴

```typescript
export interface Skill {
    /** Absolute file path to the SKILL.md file */
    location: string;
    qualifiedName: string;
    // …name, description, …
}
```

스킬 **루트 디렉터리**는 `URI.fromFilePath(skill.location).parent`로만 유도할 수 있다.  
`{{skills}}` XML의 `<location>`도 SKILL.md 절대경로다.

### 제품 쪽 소비

- Cocrates 프롬프트: `~{getSkillFileContent}` + `{{skills}}` (`theia-extensions/cocrates-agent/.../cocrates-prompt-template.ts`)
- 내장 예: `.agents/skills/sdd/SKILL.md`가 `workflow/01-prd.md` 등을 **스킬 루트 상대**로 링크. 현재 tool로는 그 파일을 못 연다.

### 이미 보조 파일을 다루는 다른 경로

| 경로 | 보조 파일 | getSkillFileContent |
|------|-----------|---------------------|
| Registry install (`ai-registry`) | 폴더 전체 download·hash (`relativePath`) | 미사용 |
| Agent Skills spec | `scripts/`, `references/`, `assets/` 허용, 상대경로 참조 | — |
| `getFileContent` (workspace) | 워크스페이스(또는 `allowedExternalPaths`) 파일 | 스킬이 workspace 밖(plugin/user config)이면 불가·우회에 부적합 |

즉 **설치·해시는 멀티파일**, **에이전트 로드 tool은 싱글파일**인 비대칭 상태다.

---

## 2. 목표 동작 (정규화 규칙)

JSON tool args 기준으로 제안:

```json
{ "skillName": "sdd" }
{ "skillName": "sdd", "path": "SKILL.md" }
{ "skillName": "sdd", "path": "./SKILL.md" }
{ "skillName": "sdd", "path": "workflow/01-prd.md" }
```

### path 정규화

1. `path` 생략 · `""` · `undefined` → `"SKILL.md"`
2. 구분자는 `/`로 통일 (`Path.normalizePathSeparator`)
3. 선행 `./` 제거 (`./SKILL.md` → `SKILL.md`)
4. **거부:** 절대경로(`/`·`C:\`·`file://`), 세그먼트 `.`만으로 구성된 쓸모없는 경로 외의 **`..`**, `\`가 섞인 세그먼트(설치 쪽과 동일 정책 참고: `skill-install-backend-service` unsafe path 검사)
5. resolve 후 `skillDir.isEqualOrParent(target)` 재확인 (normalize로 `a/../../etc` 같은 우회 차단)

### 반환 내용

| path (정규화 후) | 반환 |
|------------------|------|
| `SKILL.md` (대소문자: 상수 `SKILL_FILE_NAME`와 동일 비교 권장) | 현행과 같이 `parseSkillFile` → **body only** (호환) |
| 그 외 파일 | **raw** 전체 문자열 (frontmatter strip 하지 않음) |
| 없음 / 디렉터리 / 읽기 실패 | 현행처럼 `{ "error": "…" }` JSON 문자열 |

파라미터 이름은 기존 호환을 위해 **`skillName` 유지** + optional **`path`**. (프롬프트/모델용 설명에 “skill folder relative path”를 명시.)

---

## 3. 구현 스케치

로직은 동일하다. 제품에서는 `theia-extensions/cocrates-agent`(또는 product)에 동등 클래스를 두고 §4.C로 rebind하고, upstream에 기여할 때는 `../theia/packages/ai-ide/src/browser/skill-file-functions.ts`를 고친다.

```typescript
// getTool() parameters.properties 에 추가
path: {
    type: 'string',
    description:
        'Optional path relative to the skill directory (default: SKILL.md). ' +
        'Examples: "SKILL.md", "./SKILL.md", "workflow/01-prd.md". ' +
        'Must stay inside the skill folder; ".." and absolute paths are rejected.'
}
// required는 그대로 ['skillName']

private async getSkillFileContent(arg_string: string): Promise<string> {
    const args = JSON.parse(arg_string);
    const skillName: string = args.skillName;
    const relativePath = this.normalizeSkillRelativePath(args.path); // default SKILL.md

    const skill = this.skillService.getSkill(skillName);
    if (!skill) {
        return JSON.stringify({ error: `Skill not found: ${skillName}` });
    }

    try {
        const skillFileUri = URI.fromFilePath(skill.location);
        const skillDir = skillFileUri.parent;
        const targetUri = skillDir.resolve(relativePath).normalizePath();
        if (!skillDir.isEqualOrParent(targetUri)) {
            return JSON.stringify({ error: `Path escapes skill directory: ${relativePath}` });
        }

        const fileContent = await this.fileService.read(targetUri);
        if (relativePath === SKILL_FILE_NAME || targetUri.path.base === SKILL_FILE_NAME) {
            return parseSkillFile(fileContent.value).content;
        }
        return fileContent.value;
    } catch (error) {
        return JSON.stringify({ error: `Failed to load skill content: ${error}` });
    }
}
```

`normalizeSkillRelativePath`는 `WorkspaceFunctionScope.hasParentSegment` / install 쪽 세그먼트 검사와 같은 수준으로 두면 된다.

### description 문구

tool description에 “listed skill의 **폴더 안 상대 파일**도 path로 읽는다. SKILL.md를 먼저 로드한 뒤, 그 안의 상대 참조를 같은 tool로 이어서 로드하라”를 넣으면 sdd/workflow progressive load가 모델에 전달된다.

### 테스트

`packages/ai-ide`에 spec 추가 권장:

- default / `SKILL.md` / `./SKILL.md` → 동일 body
- `workflow/01-prd.md` → raw
- `../other/SKILL.md`, `/etc/passwd`, `foo/../../x` → error
- 없는 스킬·없는 파일 → error
- qualifiedName (`qualifier:name`)으로도 path 동작

---

## 4. 어디에 고칠지 (옵션)

### A. 제품 DI `rebind` (권장 — upstream 소스 불필요)

path 확장은 기존 `skillName`만 넘기는 호출을 **그대로 포함**하는 상위집합이므로, registry의 `getSkillFileContent`를 **전역 교체**해도 문제 없다. Explore·Capabilities도 같은 강화 tool을 쓴다.

```typescript
// cocrates-frontend-module.ts (ai-ide 모듈 로드 이후)
export default new ContainerModule((bind, _unbind, _isBound, rebind) => {
    // …기존 Cocrates 바인딩…
    bind(CocratesGetSkillFileContent).toSelf().inSingletonScope();
    rebind(GetSkillFileContent).toService(CocratesGetSkillFileContent);
});
```

- `GetSkillFileContent`는 `@theia/ai-ide`의 클래스. `bindToolProvider`가 `ToolProvider → GetSkillFileContent`이므로 클래스 rebind만으로 registry가 강화 `getTool()`을 본다 (`docs/theia/local-message-compaction.md` Coder rebind와 동일 패턴).
- **id/`name`은 기존 `getSkillFileContent` 유지** → 프롬프트의 `~{getSkillFileContent}` 변경 불필요.
- 같은 id로 `bindToolProvider`만 추가하면 registry가 **덮어쓰지 않고 warn**만 하므로, 반드시 `rebind` (또는 unregister 후 재등록).
- 모듈 순서가 `@theia/ai-ide` **이후**여야 한다.

### B. Upstream Theia

- `../theia/packages/ai-ide/.../skill-file-functions.ts`에 §3을 반영해 기여.
- 제품 rebind/패치 없이도 전 어답터가 이득. 로컬 검증은 `yarn build:local-theia`.

### C. `patches/` + patch-package

- published `@theia/ai-ide`만 급히 고칠 때. 버전 bump마다 패치 충돌 비용.

### D. tool 변경 없이 `getFileContent` 우회 (비권장)

- workspace `.agents/skills/...`만 가능.
- plugin/config 티어·절대경로 스킬은 `allowedExternalPaths` 없이는 막힘.
- Agent Skills “스킬 tool로 on-demand load” 모델과도 어긋남.

---

## 5. 부가로 손댈 수 있는 표면

필수는 아니고, path 확장과 같은 스프린트에서 검토하면 좋은 항목.

| 표면 | 비고 |
|------|------|
| `{{skill:name}}` (`skills-variable-contribution`) | 여전히 SKILL.md 전용으로 둬도 됨. 인라인 변수에 path 문법은 복잡도↑ |
| Explore / generic capabilities 프롬프트 | “path로 보조 파일” 한 줄 안내 |
| Cocrates 시스템 프롬프트 | skills 섹션에 workflow 등 상대 path 예시 |
| `ai-core` README Skills Usage | “skill name only” → “optional relative path” |
| Open Skill UI | 에디터로 SKILL.md만 여는 기존 UX와 별개; tool 범위와 무관 |

---

## 6. 권장 작업 순서

1. `cocrates-agent`에 §2–3 동작의 `CocratesGetSkillFileContent` 구현 (`id`/`name` = `getSkillFileContent`).
2. frontend 모듈에서 `rebind(GetSkillFileContent).toService(…)` (§4.A).
3. (선택) Cocrates 프롬프트 skills 안내에 `path` 예시 한 줄.
4. `@Cocrates`(및 Explore 등)로 `sdd` / `workflow/01-prd.md`·기본 SKILL.md 스모크.
5. 이후 upstream 기여가 필요하면 동일 로직을 Theia PR로 옮기고 제품 rebind 제거.

제품 Spec이 보조 파일 로드를 요구하면 `built-in-ai-skills` / `cocrates-agent` verification에 한 줄 추가.

---

## 핵심 파일 맵

| 경로 | 역할 |
|------|------|
| `theia-extensions/cocrates-agent/` (강화 ToolProvider + `rebind`) | **제품 수정 본진** |
| `../theia/packages/ai-ide/src/browser/skill-file-functions.ts` | upstream as-is / 기여 시 동등 수정 |
| `../theia/packages/ai-core/src/common/skill.ts` | `SKILL_FILE_NAME`, `parseSkillFile`, `Skill.location` |
| `../theia/packages/ai-registry/.../skill-install-backend-service.ts` | 상대 path 안전 검사 참고 |
| `../theia/packages/ai-ide/src/browser/workspace-functions.ts` | path normalize / parent-segment 참고 |
| `.agents/skills/sdd/` | 멀티파일 스킬 대표 소비자 |
| `theia-extensions/cocrates-agent/.../cocrates-prompt-template.ts` | (선택) path 안내 문구 |
