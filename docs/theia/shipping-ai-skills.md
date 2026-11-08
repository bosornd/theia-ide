# Theia AI: IDE에 Skill 포함·배포하기

조사 기준: 형제 저장소 `../theia/` (`@theia/ai-core` 1.76.0, `@theia/ai-registry`) 및 이 제품의 Electron/browser 조립.  
관련: [adding-ai-agents.md](./adding-ai-agents.md) §5 Skills, [get-skill-file-content-path.md](./get-skill-file-content-path.md) (`getSkillFileContent`에 스킬 폴더 상대 `path` 확장), upstream issue [#17869](https://github.com/eclipse-theia/theia/issues/17869).

---

## 한눈에 보기

| 질문 | 답 |
|------|----|
| Skill이란 | `SKILL.md`(+ YAML frontmatter)가 있는 **디렉터리**. 에이전트가 `{{skills}}` / `getSkillFileContent`로 소비 |
| 공식 “builtin 번들” API | **아직 없음** (#17869 open). 대신 파일시스템 탐색 + `SkillDirectoryContribution` |
| 제품에 넣으려면 | (권장) `extraResources`로 번들 → `SkillDirectoryContribution` 또는 `skillDirectories`로 스캔 / (대안) 최초 기동 시 config dir에 복사 / (대안) Agent Plugin으로 배포 |
| 개발 중 | 워크스페이스 `.agents/skills/` 또는 preference로 로컬 폴더 지정 |

---

## 1. Skill이 어디서 발견되는가

`DefaultSkillService` (`packages/ai-core/src/browser/skill-service.ts`)가 아래 순서로 디렉터리를 합친다. **같은 이름(또는 qualifiedName)이면 먼저 나온 쪽이 이긴다.**

| 우선순위 | 티어 | 경로 |
|----------|------|------|
| 1 (높음) | `workspace` | `<workspace>/.prompts/skills/`, `<workspace>/.agents/skills/` |
| 2 | `configured` | preference `ai-features.skills.skillDirectories` (틸드 `~` 확장) |
| 3 | `default` | `<productConfigDir>/skills/`, `~/.agents/skills/` |
| 4 (낮음) | `plugin` | `SkillDirectoryContribution`이 반환한 루트 (예: Agent Plugin의 `…/skills`) |

### 이 제품에서의 config dir

`applications/*/package.json` → `theia.configurationFolder`:

- stable / browser: **`.theia-ide`** → 기본 글로벌 스킬 폴더 = `~/.theia-ide/skills/`
- next: **`.theia-ide-next`** → `~/.theia-ide-next/skills/`

README의 `~/.theia/skills`는 “제품 configuration folder”의 예시일 뿐이다. theia-ide에서는 **`.theia-ide`** 가 기준이다.

Registry에서 **Install**한 스킬은 항상 `~/.agents/skills/<name>/` (backend `SkillInstallBackendService.skillsRoot`). Agent Plugin은 `~/.agents/plugins/<id>/` 아래 `skills/` (`PluginSkillDirectoryContribution`, qualifier로 이름 충돌 회피).

### 폴더·파일 규칙

```text
<skills-root>/
├── my-skill/
│   ├── SKILL.md          # 필수. frontmatter name === 디렉터리명 (kebab-case)
│   └── (선택) 보조 파일
└── another-skill/
    └── SKILL.md
```

`SkillDirectoryContribution` / preference에 넣는 경로는 **스킬 폴더들의 부모**(위 트리의 `<skills-root>`), 개별 `my-skill/`이 아니다.

소비:

- 프롬프트에 `{{skills}}` → 이름·설명 XML
- 본문 → tool `getSkillFileContent` (`~{getSkillFileContent}`)
- `{{skill:name}}` 변수로 특정 스킬 인라인 가능  
Coder 기본 프롬프트에는 `{{skills}}`가 **없다**. Explore 등은 있음. 제품 agent가 스킬을 쓰게 하려면 프롬프트/Capabilities 경로를 맞춰야 한다.

---

## 2. 배포 패키지에 포함하는 방법

### 제약 (왜 “그냥 확장에 넣기”가 애매한가)

- Skill은 **실파일시스템 절대 경로**로 등록된다 (`Skill.location`). `getSkillFileContent`·Open UI가 그 경로를 읽는다.
- Electron은 `asar: true`. asar 안 경로를 FileService가 안정적으로 다루게 하려면 **`extraResources` 또는 `asarUnpack`** 으로 asar 밖에 두는 편이 안전하다 (VS Code 확장 `plugins/`와 동일 패턴).
- Browser/Docker는 백엔드가 보이는 디스크 경로만 스캔한다. 이미지에 파일을 넣고 contribution/preference로 그 경로를 가리켜야 한다.
- Upstream [#17869](https://github.com/eclipse-theia/theia/issues/17869)는 “builtin tier + copy-on-write 커스터마이즈”를 제안 중이며 **미구현**. 지금은 파일 루트 기여로 우회한다.

### 옵션 A — `SkillDirectoryContribution` + 번들 디렉터리 (제품 전용에 가장 맞음)

1. 저장소에 예: `theia-extensions/product/skills/` 또는 `applications/*/resources/skills/`에 스킬 트리 작성.
2. Electron: `electron-builder.yml` `extraResources`에 추가 (예: `to: app/skills`). AppImage 읽기 전용이면 VS Code `builtInPlugins`처럼 사용자 config로 복사하는 헬퍼를 검토.
3. `theia-extensions/`에 contribution 구현:

```typescript
@injectable()
export class ProductSkillDirectoryContribution implements SkillDirectoryContribution {
    async getSkillDirectories(): Promise<SkillDirectoryEntry[]> {
        return [{
            path: /* 패키징·dev 모두에서 해석 가능한 절대 경로 */,
            tier: 'plugin',
            qualifier: 'theia-ide' // 선택: 이름이 `theia-ide:my-skill`로 노출
        }];
    }
}
// frontend 모듈:
bind(ProductSkillDirectoryContribution).toSelf().inSingletonScope();
bind(SkillDirectoryContribution).toService(ProductSkillDirectoryContribution);
```

4. applications dependency에 해당 확장 포함 후 빌드·패키징.

장점: 사용자 `~/.agents/skills`와 분리, 업데이트 시 번들만 갈면 됨(복사 전략을 쓰면 버전 메타 필요).  
단점: 경로 해석(dev / 패키지 / AppImage / browser)을 제품이 책임져야 함. `plugin` 티어라 workspace·user 스킬에 가려질 수 있음(의도된 동작).

### 옵션 B — `extraResources` + preference `ai-features.skills.skillDirectories`

번들 경로를 preference **default** 또는 제품 설정으로 넣는다.  
절대 경로가 환경마다 달라서, 보통은 **런타임에 경로를 정해 contribution이나 동적 preference로 넣는 A가 낫다.**

### 옵션 C — 최초 기동 시 `~/.theia-ide/skills/`에 시드 복사

`default` 티어에 자동 포함되므로 contribution 없이 동작.  
#17869가 지적하듯 **제품 콘텐츠와 사용자 편집이 섞이고**, 앱 업데이트 시 덮어쓰기/보존 정책이 필요하다. 단순 PoC용.

### 옵션 D — Agent Plugin으로 배포

구조 예:

```text
my-plugin/
├── plugin.json
└── skills/
    └── deploy/
        └── SKILL.md
```

설치 위치: `~/.agents/plugins/<filenamified-id>/`. `PluginSkillDirectoryContribution`이 `skills/`를 `plugin` 티어로 등록(qualifier 부여).  
제품 “내장”보다는 **레지스트리·수동 설치** 모델. 미리 풀어 두고 link/install 하는 방식으로는 가능.

### 옵션 E — AI Registry 스킬 Install만 안내

배포 패키지에 파일을 넣지 않고, 사용자가 Extensions에서 설치 → `~/.agents/skills`. 제품 번들과는 별개.

### theia-ide에서 손댈 위치 (구현 시)

| 목적 | 경로 |
|------|------|
| contribution / 시드 로직 | `theia-extensions/product` (또는 전용 확장) |
| Electron 번들 | `applications/electron|electron-next/electron-builder.yml` `extraResources`, 필요 시 `scripts/theia-electron-main.js`·`appimage-helpers.js` |
| Browser/Docker | 이미지에 스킬 트리 복사 + 동일 contribution/경로 |
| Agent가 스킬을 쓰게 | agent 프롬프트에 `{{skills}}` + `~{getSkillFileContent}` (또는 Capabilities) |

---

## 3. 스킬 폴더 설정 방법

### Preference

- 키: `ai-features.skills.skillDirectories`
- 타입: `string[]` (추가 루트만; workspace·config·`~/.agents/skills`는 항상 포함)
- UI: AI Configuration / Settings

예:

```json
{
  "ai-features.skills.skillDirectories": [
    "~/shared-skills",
    "/opt/company/theia-skills"
  ]
}
```

존재하지 않는 configured 경로는 경고 로그 후 스킵.

### 코드 contribution

`SkillDirectoryContribution` (`ai-core`):

- `getSkillDirectories(): MaybePromise<SkillDirectoryEntry[]>`
- 선택 `onDidChange`로 재스캔
- `tier: 'plugin'`, 선택 `qualifier`

등록은 `@theia/ai-registry`의 `PluginSkillDirectoryContribution`과 동일하게 `bind(SkillDirectoryContribution).toService(...)`.

### 우선순위·섀도잉

워크스페이스에 같은 `name`의 스킬이 있으면 제품 번들(`plugin` 티어)은 보이지 않는다.  
플러그인끼리 같은 이름은 `qualifier:name`으로 구분된다.

---

## 4. 개발 중에는 어떻게?

| 방법 | 언제 |
|------|------|
| `<repo>/.agents/skills/<name>/SKILL.md` | 프로젝트·워크스페이스 스킬. 별도 설정 없음. **가장 간단** |
| `.prompts/skills/` | 동일 workspace 티어; 같은 이름이면 `.prompts`가 `.agents`보다 앞 |
| Settings에 `skillDirectories` → 모노레포 `skills/` 절대경로 | 제품 번들과 같은 트리를 개발 중에도 스캔 |
| `~/.theia-ide/skills/`에 심링크/복사 | 글로벌 스킬처럼 동작 |
| CreateSkill agent | SKILL.md 작성 보조 (`@theia/ai-ide`) |
| `yarn electron start` / `yarn browser start` | contribution·extraResources 없이 workspace/preference만으로도 검증 가능 |

개발 체크리스트:

1. AI enable (`ai-features.AiEnable.enableAI`).
2. AI Configuration → Skills(또는 Variables의 skills)에 스킬이 보이는지.
3. 테스트 agent 프롬프트에 `{{skills}}`가 있는지 (Coder만으로는 목록이 안 들어갈 수 있음).
4. 채팅에서 스킬 사용 유도 → `getSkillFileContent` 호출 여부.

패키징 검증: `yarn electron package:preview` 후 설치본에서 번들 경로·AppImage 복사·asar 밖 배치를 확인.

---

## 5. 권장 방향 (theia-ide)

1. **단기 / 개발:** 워크스페이스 `.agents/skills` + (필요 시) preference로 공유 폴더.
2. **제품 내장 배포:** 스킬 트리를 **asar 밖 `extraResources`** 로 넣고, `theia-extensions`의 **`SkillDirectoryContribution`** 으로 절대 경로 기여. AppImage는 plugins와 같은 **버전 메타 기반 사용자 디렉터리 복사**를 검토.
3. **사용자 추가 설치:** Registry Skills / Agent Plugin 기존 경로 유지.
4. Upstream builtin tier(#17869)가 들어오면 contribution·시드 복사를 그쪽 API로 이전 검토.

---

## 핵심 파일 맵

| 경로 (`../theia/`) | 역할 |
|--------------------|------|
| `packages/ai-core/README.md` | Skills 개요 |
| `packages/ai-core/src/common/skill.ts` | 파싱, `combineSkillDirectories`, qualifier |
| `packages/ai-core/src/browser/skill-service.ts` | 스캔, `SkillDirectoryContribution` |
| `packages/ai-core/src/common/ai-core-preferences.ts` | `ai-features.skills.skillDirectories` |
| `packages/ai-registry/.../plugin-skill-directory-contribution.ts` | Agent Plugin → skills 루트 |
| `packages/ai-registry/.../skill-install-backend-service.ts` | `~/.agents/skills` 설치 |
| issue #17869 | 공식 builtin 스킬 제안 (open) |

| 경로 (theia-ide) | 역할 |
|------------------|------|
| `applications/*/package.json` `configurationFolder` | `~/.theia-ide(-next)/skills` |
| `applications/electron*/electron-builder.yml` | `extraResources` 패턴 (plugins 참고) |
| `applications/electron*/scripts/theia-electron-main.js` | AppImage built-in 복사 패턴 |
