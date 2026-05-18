---
allowed-tools: Bash(git:*), Bash(git flow:*), Bash(echo:*)
description: git-flow feature 브랜치를 생성한다 (develop 최신화 + git flow feature start)
argument-hint: "<feature-id-or-name>"
---

## 공통 규칙 (반드시 준수)

- 모든 git 조작은 가능한 한 `git flow` 명령을 우선 사용한다.
- `--no-verify`, `--force`, `--force-with-lease` 사용 금지. 강제 푸시가 필요해 보이는 상황이라도 절대 단독 결정하지 말고 사용자에게 묻는다.
- 머지 기준 브랜치(`develop`/`main`) 갱신은 항상 `git pull --ff-only`만 사용한다.
- 충돌 발생 시 자동 해결 금지. 즉시 중단 + 원상복구 + 사용자에게 보고.
- 한 커밋 = 하나의 논리적 변경. Conventional Commits + Feature ID 필수.

## 인자

`$ARGUMENTS` — feature 브랜치 식별자.
- 순수 숫자(`^[0-9]+$`) → Feature ID로 간주하고 그대로 브랜치 이름으로 사용한다.
- 인자가 없거나 비숫자 → 입력을 kebab-case로 정규화하여 사용 (공백/언더스코어 → `-`, 소문자화, 영숫자·하이픈 외 제거).
- 인자가 없으면 사용자에게 한 번 짧게 묻고 (`AskUserQuestion` 가능) 답을 받아 사용한다.

정규화 후 최종 브랜치 이름은 `feature/<id-or-slug>` 형태가 된다 (`git flow feature start <slug>` 가 자동으로 `feature/` 접두사를 붙임).

## 컨텍스트 수집 (반드시 먼저 실행)

다음 명령을 병렬로 실행해서 현재 상태를 확인한 뒤 진행한다:

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git remote -v
```

조건:
- git 저장소가 아니면 즉시 중단하고 사용자에게 보고.
- working tree가 더럽다면(staged/unstaged 변경 존재) 진행하지 않고 사용자에게 어떻게 처리할지 묻는다 (stash / 커밋 / 중단).
- `git flow` 가 초기화되어 있지 않으면 (`.git/config` 에 `[gitflow "branch"]` 섹션 없음) 사용자에게 알리고, 동의를 받은 후 `git flow init -d` 를 실행한다.

## 실행 절차

1. 인자 파싱 → `SLUG` 결정. 숫자면 그대로, 아니면 kebab-case.
2. `develop` 최신화:
   ```bash
   git checkout develop
   git pull --ff-only origin develop
   ```
   ff-only 가 실패하면 즉시 중단하고 보고. 자동 머지/리베이스 시도 금지.
3. feature 브랜치 생성:
   ```bash
   git flow feature start "$SLUG"
   ```
4. 생성 후 확인:
   ```bash
   git branch --show-current
   ```
   결과가 `feature/$SLUG` 가 맞는지 검증.
5. 세션 메모로 다음 정보를 사용자에게 명시적으로 출력 (이후 `/feature-commit` 가 브랜치 이름에서 다시 파싱해 사용):
   - Feature 브랜치: `feature/$SLUG`
   - Feature ID: `$SLUG` (숫자면 그대로 ID, slug면 그 자체)
   - 베이스 SHA: `git merge-base HEAD develop` 결과

## 출력 형식

```
✓ feature/<slug> 생성 완료
  base: <develop-sha>
  next: /feature-commit 으로 변경사항을 커밋하세요
```

## 금지 사항

- `git push` 를 이 단계에서 절대 호출하지 않는다 (브랜치 생성 직후엔 푸시할 변경이 없음).
- `git flow feature start` 외의 우회 명령(`git checkout -b ...`)을 쓰지 않는다.
