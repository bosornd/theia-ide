# 제품 브랜딩(이름) 변경 가이드

이 저장소의 “Theia IDE” 정체성을 다른 제품명으로 바꾸려면 **런타임 설정**, **패키징 메타**, **UI/리소스**, **런처·업데이터 하드코드**를 함께 맞춰야 한다. Theia 프레임워크(`@theia/*`)를 고칠 필요는 없고, 제품 조립·확장 쪽만 수정한다.

참고 패턴: `applications/electron-next`는 stable과 코드를 공유하면서 `package.json`의 `theia` 블록 값만으로 이름·설정 폴더·URI 스킴·브랜딩을 갈라친다. 커스텀 제품도 같은 방식으로 “설정으로 분기”하는 것이 이 저장소의 의도다.

---

## 1. 정체성 맵 (무엇을 바꿀지)

재브랜딩 전에 아래 값을 먼저 정한다. 예시는 가상의 `Acme IDE`다.

| 역할 | Theia IDE (stable) | Theia IDE Next | 예시 (커스텀) |
|------|--------------------|----------------|---------------|
| 표시 이름 (창 제목·메뉴·Welcome 일부) | `Theia IDE` | `Theia IDE Next` | `Acme IDE` |
| npm/`productName` (공백 가능) | `Theia IDE` | `Theia IDE Next` | `Acme IDE` |
| 실행 파일·아티팩트명 (`electron-builder` `productName`, 공백 없음) | `TheiaIDE` | `TheiaIDENext` | `AcmeIDE` |
| 사용자 설정 폴더 | `.theia-ide` | `.theia-ide-next` | `.acme-ide` |
| OS 앱 ID (`appId`) | `eclipse.theia` | `eclipse.theia.next` | `com.acme.ide` |
| Windows Jump List ID (`appUserModelId`) | `eclipse.theia` | `eclipse.theia.next` | `com.acme.ide` (**`appId`와 동일해야 함**) |
| URI / CLI 스킴 | `theia` (기본값) | `theia-next` | `acme` |
| 브랜딩 변형 | (없음 = stable) | `next` | 필요 시 새 variant 또는 stable만 사용 |

**이름이 두 종류다.** UI·CLI 도움말은 공백 있는 표시명(`Theia IDE`)을 쓰고, 설치본 실행 파일·`.app`·AppImage·deb 아티팩트는 `electron-builder.yml`의 공백 없는 `productName`(`TheiaIDE`)을 쓴다. `executableName`을 따로 두지 않으면 후자가 실행 파일명이 된다.

---

## 2. 수정 축 세 곳

```text
applications/{browser,electron,electron-next}/
  package.json          → theia.frontend/backend 설정 (런타임 정체성)
  electron-builder.yml  → appId, 실행파일명, 아이콘, publish, mac protocols
  resources/            → splash, 플랫폼 아이콘, LICENSE, preload
  scripts/              → AppImage config 폴더 하드코드 등

theia-extensions/product/
  → Welcome/About 카피·로고·CSS·Help 링크·Linux 창 아이콘 contribution

theia-extensions/launcher/ · updater/
  → CLI/desktop·sudo 표시명, 업데이트 UI 문구·피드 URL (하드코드 구간)
```

browser는 `product`만, Electron(stable/next)은 `product` + `launcher` + `updater`를 포함한다.

---

## 3. 앱 `package.json`의 `theia` 블록

각 타깃의 기준 파일:

- `applications/browser/package.json`
- `applications/electron/package.json`
- `applications/electron-next/package.json`

### 바꿔야 할 필드

| 필드 | 위치 | 효과 |
|------|------|------|
| `productName` | 패키지 루트 | CLI `--help`/`--version` 등 스크립트 표시명 |
| `theia.frontend.config.applicationName` | frontend | 창 제목, HTML `<title>`, 런처 다이얼로그, `.desktop`의 `Name=` |
| `theia.backend.config.configurationFolder` | backend | 사용자 데이터 루트 (`~/.theia-ide` 등) |
| `theia.frontend.config.electron.appUserModelId` | Electron만 | Windows 알림/Jump List. **`electron-builder` `appId`와 일치** |
| `theia.frontend.config.electron.uriScheme` | Electron | OS 프로토콜 핸들러, CLI `/usr/local/bin/<scheme>` |
| `theia.frontend.config.electron.splashScreenOptions.content` | Electron | 스플래시 SVG 경로 |
| `theia.frontend.config.brandingVariant` | (next 패턴) | `data-theia-branding` → product CSS/카피 분기 |
| `theia.frontend.config.availableUpdateChannels` + `preferences["updates.channel"]` | Electron | 업데이트 채널 |

### stable Electron 예시 (현재)

```json
"productName": "Theia IDE",
"theia": {
  "frontend": {
    "config": {
      "applicationName": "Theia IDE",
      "electron": {
        "appUserModelId": "eclipse.theia",
        "splashScreenOptions": {
          "content": "resources/TheiaIDESplash.svg"
        }
      }
    }
  },
  "backend": {
    "config": {
      "configurationFolder": ".theia-ide"
    }
  }
}
```

stable에는 `uriScheme`이 없다. Theia 기본값은 `theia`다 (`../theia`의 `application-props` 기본). 커스텀 스킴을 쓰려면 **명시적으로** `electron.uriScheme`을 넣고, 아래 mac `protocols`·런처와도 맞춘다.

### next가 보여 주는 분기 예시

```json
"applicationName": "Theia IDE Next",
"brandingVariant": "next",
"availableUpdateChannels": ["next"],
"preferences": { "updates.channel": "next" },
"electron": {
  "uriScheme": "theia-next",
  "appUserModelId": "eclipse.theia.next",
  "splashScreenOptions": { "content": "resources/TheiaIDENextSplash.svg" }
},
"configurationFolder": ".theia-ide-next"
```

빌드 후 `src-gen/`에 값이 박힌다. `src-gen/`은 생성물이므로 직접 편집하지 말고 `package.json`을 고친 뒤 앱을 다시 빌드한다.

---

## 4. `electron-builder.yml` (실행 파일·appId·아이콘·프로토콜)

기준 파일:

- `applications/electron/electron-builder.yml`
- `applications/electron-next/electron-builder.yml`

### 필수

```yaml
appId: eclipse.theia          # → com.acme.ide 등. appUserModelId와 동일
productName: TheiaIDE         # → AcmeIDE. 실행 파일·아티팩트 basename
```

`executableName`을 쓰지 않으면 `productName`이 곧 실행 파일명이다 (`TheiaIDE.exe`, `TheiaIDE.app`, AppImage 등). E2E(`applications/electron/test/app.spec.js`)도 이 `productName`으로 바이너리 경로를 잡는다.

### 아이콘·스플래시 경로

| 항목 | stable 경로 |
|------|-------------|
| Windows | `resources/icons/WindowsLauncherIcons/TheiaIDE.ico` (+ NSIS installer/uninstaller 아이콘) |
| macOS | `resources/icons/MacLauncherIcons/icon.icns` |
| Linux 패키지 | `resources/icons/LinuxLauncherIcons` |
| Linux 창 아이콘(번들 포함) | `resources/icons/WindowIcon/512-512.png` (`files`에 명시) |
| NSIS 사이드바 | `resources/icons/InstallerSidebarImage/*.bmp` |
| 스플래시 | `resources/TheiaIDESplash.svg` (`files`와 `splashScreenOptions` 둘 다) |

파일명을 바꾸면 `electron-builder.yml`의 `icon` / `files` / `nsis.*` 경로와 `package.json` splash `content`를 같이 고친다. 파일 내용만 교체하고 경로를 유지해도 된다.

### macOS 프로토콜

```yaml
mac:
  protocols:
    - name: theia
      schemes:
        - theia
```

`uriScheme`을 바꾸면 여기 `name`/`schemes`도 동일하게 맞춘다.

### 게시·업데이트 URL

`publish.url`(및 updater 확장의 피드 상수)이 실제 배포 위치와 일치해야 자동 업데이트가 동작한다. 자체 호스팅이면 `electron-builder.yml`과 `theia-extensions/updater`의 채널 URL을 함께 교체한다.

---

## 5. AppImage main 스크립트의 설정 폴더 하드코드

`configurationFolder`만 바꿔서는 AppImage 플러그인 복사 경로가 따라가지 않는다. 진입 스크립트에 **폴더명이 다시 하드코드**되어 있다.

| 타깃 | 파일 | 기본값 |
|------|------|--------|
| stable | `applications/electron/scripts/theia-electron-main.js` | `~/.theia-ide` |
| next | `applications/electron-next/scripts/theia-electron-main.js` | `~/.theia-ide-next` |

```js
const configDir = process.env.THEIA_CONFIG_DIR || path.join(os.homedir(), '.theia-ide');
```

`package.json`의 `configurationFolder`와 **항상 동기화**한다. (`THEIA_CONFIG_DIR`로 런타임 오버라이드는 가능.)

---

## 6. `theia-extensions/product` (UI 브랜딩)

`applicationName`만 바꿔도 **Welcome/About 큰 제목은 그대로**일 수 있다. 표시 카피와 로고는 product 확장에 있다.

### 동작 요약

1. 앱이 `brandingVariant: "next"`를 주면 `theia-ide-config.ts`의 `applyBranding()`이 `document.body`에 `data-theia-branding="next"`를 붙인다.
2. `style/index.css`가 variant별 로고·아이콘·액센트 색 CSS 변수를 바꾼다.
3. `branding-util.tsx`의 `renderProductName()`은 **`applicationName`을 쓰지 않고** `Eclipse Theia IDE`(+ Next 접미사)를 하드코드한다.

### 손댈 파일

| 파일 | 내용 |
|------|------|
| `src/browser/branding-util.tsx` | 제품 헤드라인, Welcome/About 섹션 문구·다운로드/문서/지원 URL |
| `src/browser/style/index.css` | `--theia-branding-logo` / `-icon` / `-accent` |
| `src/browser/icons/TheiaIDE.png`, `512-512.png` | stable 워드마크·탭 아이콘 |
| `src/browser/icons/TheiaIDE-next.png`, `512-512-next.png` | next 변형 |
| `src/browser/theia-ide-contribution.tsx` | Help 메뉴 URL, 커맨드 id/category |
| `src/browser/theia-ide-getting-started-widget.tsx` | Welcome 레이아웃 |
| `src/browser/theia-ide-about-dialog.tsx` | About 다이얼로그 |
| `src/browser/theia-ide-ai-registry-configuration.ts` | AI 레지스트리 키 (`theia-ide` / `theia-ide-next`) |
| `src/electron-main/icon-contribution.ts` | Linux 창 아이콘 → 앱 `resources/icons/WindowIcon/512-512.png` (번들 상대 경로) |

커스텀 제품이 next 채널을 쓰지 않으면 `brandingVariant` 없이 stable 경로(기본 CSS·아이콘)만 교체해도 된다. 두 외형이 필요하면 next처럼 variant + CSS 변수 + 아이콘 세트를 추가한다.

브라우저 favicon: `applications/browser/ico/favicon.ico`.

---

## 7. `launcher` / `updater` 하드코드

설정으로 안 빠지는 문자열·URL이 있다.

### launcher (`theia-extensions/launcher`)

| 위치 | 내용 |
|------|------|
| `create-launcher-contribution.ts` | 커맨드 category `Theia IDE` (다이얼로그 본문은 `applicationName`/`uriScheme` 사용) |
| `launcher-endpoint.ts` | sudo 프롬프트 이름: `uriScheme === 'theia-next' ? 'Theia IDE Next' : 'Theia IDE'` |
| `desktopfile-endpoint.ts` | `.desktop` `Name=`, `MimeType=x-scheme-handler/${uriScheme}`, `StartupWMClass` = applicationName을 kebab-case |
| `launcher-util.ts` | 스토리지 하위 경로 `globalStorage/theia-ide-launcher/` |
| CLI 설치 경로 | `/usr/local/bin/${uriScheme}` |

### updater (`theia-extensions/updater`)

| 위치 | 내용 |
|------|------|
| `theia-updater-frontend-contribution.ts` | 진행 문구 `Theia IDE Update` |
| `theia-updater-impl.ts` | stable/preview/next 피드 URL 상수 |
| `theia-updater.ts` | 채널 타입 `stable` \| `preview` \| `next` |

자체 업데이트 서버를 쓰면 피드 상수와 `electron-builder.yml` `publish`를 한 세트로 맞춘다.

---

## 8. 리소스 교체 체크리스트

Electron stable 기준 유지보수 자산:

```text
applications/electron/resources/
  TheiaIDESplash.svg
  icons/WindowsLauncherIcons/TheiaIDE.ico
  icons/MacLauncherIcons/icon.icns
  icons/LinuxLauncherIcons/512x512.png
  icons/WindowIcon/512-512.png
  icons/InstallerSidebarImage/*.bmp
  LICENSE
  preload.html          # CSS 키프레임명 theia-ide-spinner 등은 동작에 무관(선택 정리)

applications/browser/ico/favicon.ico

theia-extensions/product/src/browser/icons/
  TheiaIDE.png, 512-512.png
  (+ next 변형이 있으면 *-next.png)
```

next 아이콘은 루트 `scripts/generate-next-icons.js`가 stable 블루(`#00ADEE`)를 퍼플로 리컬러해 생성한다. 브랜드 색이 다르면 스크립트 색상 맵 또는 수동 교체를 검토한다.

---

## 9. 권장 작업 순서

1. **정체성 표 확정** — 표시명, 실행파일명, config folder, appId/AUMID, uriScheme, 업데이트 피드.
2. **`applications/*/package.json` `theia` 블록** — browser → electron → (필요 시) electron-next.
3. **`electron-builder.yml`** — `appId`, `productName`, 아이콘 경로, mac `protocols`, `publish`, `files`의 splash/WindowIcon.
4. **리소스 파일 교체** — splash, Win/Mac/Linux/WindowIcon, NSIS, favicon, product PNG.
5. **`theia-extensions/product`** — `branding-util.tsx` 헤드라인·카피·URL, CSS/아이콘, Help contribution.
6. **launcher / updater** — sudo·category·Update 문구, 피드 URL, storage id.
7. **`theia-electron-main.js`** — AppImage용 `~/.…`를 `configurationFolder`와 동기화.
8. **서명·공증** — `applications/electron/scripts/sign-directory.ts`의 bundle id `eclipse.theia` 등.
9. **(선택) npm 패키지 rename** — `theia-ide-*-app` / `theia-ide-*-ext`를 바꾸면 루트 `package.json`의 yarn scope(`theia-ide*app`, `theia-ide-next*` 등), 워크스페이스 의존성, Docker/CI 아티팩트명까지 연쇄 수정. **제품 표시명만 바꿀 때는 패키지명을 유지해도 된다.**
10. **검증** — 아래 절.

CI·Docker·README의 `theia-ide` 문자열은 런타임 정체성과 무관한 경우가 많다. 배포 파이프라인까지 바꿀 때만 손댄다.

---

## 10. 검증

| 확인 | 방법 |
|------|------|
| 확장 빌드 | `yarn build:extensions` |
| 앱 빌드·기동 | `yarn electron build && yarn electron start` (또는 browser / electron-next) |
| 창 제목·스플래시 | `applicationName`, splash 자산 |
| Welcome / About | product 카피·로고·액센트 (창 제목과 별개로 확인) |
| 설정 폴더 | 기동 후 `~/.<configurationFolder>` 생성 여부 |
| 패키징 실행 파일명 | `yarn electron package:preview` 후 dist의 `productName` 바이너리 |
| URI / CLI | AppImage에서 런처 설치 → `/usr/local/bin/<uriScheme>`, `.desktop` Name/MimeType |
| Windows AUMID | `appUserModelId` === `appId` |
| 업데이트 | 채널 preference·피드 URL (해당 시) |
| E2E | `yarn electron package:preview && yarn electron test` |

---

## 11. 자주 빠뜨리는 함정

1. **Welcome 제목 ≠ `applicationName`** — `branding-util.tsx`를 안 고치면 UI에 “Eclipse Theia IDE”가 남는다.
2. **`appId` ≠ `appUserModelId`** — Windows Jump List/알림이 깨질 수 있다. Theia 주석도 둘을 동일하게 두라고 한다.
3. **`configurationFolder`만 변경** — AppImage `theia-electron-main.js` 하드코드를 잊으면 플러그인 복사 경로가 예전 폴더를 가리킨다.
4. **stable `uriScheme` 미기재** — 기본 `theia`가 생성된다. 커스텀이면 명시 + mac protocols + desktop mime.
5. **아이콘 파일만 교체하고 yml/`files`/splash `content` 경로 불일치** — 패키징본에 옛 파일명 참조가 남는다.
6. **업데이트 피드와 `publish.url` 불일치** — 설치본이 업데이트를 못 찾는다.
7. **npm 패키지명까지 rename** — 루트 스크립트 scope·의존성 그래프 전체를 함께 고쳐야 한다. 표시명 재브랜딩과는 범위를 분리하는 편이 안전하다.

---

## 관련 문서

- 저장소 구조·타깃 차이: 루트 `AGENTS.md`
- asar / `asarUnpack` / patch-package: `ADOPTER.md`
- 로컬 Theia 프레임워크 연동: `docs/developing-with-local-theia.md`
