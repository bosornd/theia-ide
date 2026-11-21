# Cocrates Theia SCM commit adapter 조사

## 조사 결과

Theia 1.77의 공통 SCM 계층에는 제품이 직접 호출할 `commit(message)` 메서드는 없다. 대신 현재 선택된 `ScmRepository`의 provider가 `acceptInputCommand?: ScmCommand`를 제공한다.

- `ScmService.selectedRepository`로 선택된 repository를 얻는다.
- `ScmRepository.input.value`에 commit message를 설정할 수 있다.
- `ScmRepository.provider.acceptInputCommand`에서 provider가 등록한 commit 명령과 인자를 얻는다.
- Theia 공통 `scm.acceptInput` 명령은 provider의 `acceptInputCommand.command`를 `CommandService.executeCommand`로 실행한다.
- 따라서 Option B의 제품 adapter는 SCM repository input을 채운 뒤 provider command를 실행하는 방식으로 구현할 수 있다.

## 구현 경계

1. `ScmService.selectedRepository`가 있는지 확인한다.
2. provider의 `acceptInputCommand`와 command id가 있는지 확인한다.
3. commit message를 길이·공백·허용 문자 정책으로 검증한다.
4. provider groups/resources를 확인해 commit 대상 변경이 하나라도 있는지 확인한다. staged 여부는 거부 조건으로 사용하지 않으며, provider가 commit 대상으로 노출하는 변경을 모두 처리한다.
5. `repository.input.value = commitMessage`로 commit 입력을 설정한다.
6. `CommandService.executeCommand(command.command, ...(command.arguments ?? []))`를 호출한다.
7. command 실패·provider capability 없음·repository 없음·commit 대상 변경 없음·중복 처리 상태를 `answer`와 분리된 side effect 결과로 기록한다.

## 한계와 확인 항목

- `acceptInputCommand`는 provider별 command id와 동작에 의존하므로 모든 SCM provider의 commit semantics를 공통 API가 보장하지 않는다.
- 공통 SCM 모델에는 staged/unstaged를 제품이 동일하게 판정할 수 있는 명시적인 commit API가 없으므로 staged 구분은 수행하지 않고 provider command semantics에 맡긴다.
- 현재 선택 repository 기준 동작은 Theia SCM UI의 기본 경계와 일치하지만, JSON 응답이 의도한 workspace와 선택 repository가 다를 때의 정책은 구현 검증에서 명시해야 한다.
- 이 한계가 R-007~R-008을 충족하지 못하게 하면 shell/git으로 자동 전환하지 않고 실패 상태 또는 후속 ADR로 올린다.

## 근거

- 형제 Theia v1.77 소스 `packages/scm/src/browser/scm-provider.ts`
- 형제 Theia v1.77 소스 `packages/scm/src/browser/scm-repository.ts`
- 형제 Theia v1.77 소스 `packages/scm/src/browser/scm-service.ts`
- 형제 Theia v1.77 소스 `packages/scm/src/browser/scm-contribution.ts`의 `acceptInput()`
