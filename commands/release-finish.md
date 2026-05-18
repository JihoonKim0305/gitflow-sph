---
allowed-tools: Bash(git:*), Bash(git flow:*), Bash(awk:*), Bash(sed:*)
description: release 브랜치를 main/develop 에 머지하고 태그 + push 까지 처리
---

## 공통 규칙 (반드시 준수)

- `--no-verify`, `--force`, `--force-with-lease` 금지.
- `git pull --ff-only` 만 사용.
- 충돌 발생 시 자동 해결 금지. 즉시 `git merge --abort`, 필요 시 `git reset --hard "$PRE_FINISH_SHA"` 로 원상복구하고 사용자에게 보고.
- 커밋 메시지에 `Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>` 트레일러 (필요 시) 포함.

## 컨텍스트 수집 (병렬)

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git remote -v
git branch --list 'release/*'
test -f CHANGELOG.md && head -n 100 CHANGELOG.md
```

검증:
- 현재 또는 존재하는 release 브랜치가 `release/<version>` 형태인지 확인.
- `RELEASE_BRANCH = release/<version>`, `VERSION = <version>` 추출.
- working tree 가 깨끗해야 함.

## 사전 백업

`/release-finish` 시작 시점의 main/develop SHA 를 기록 (원상복구용):

```bash
MAIN_BEFORE=$(git rev-parse origin/main 2>/dev/null || git rev-parse main)
DEV_BEFORE=$(git rev-parse origin/develop 2>/dev/null || git rev-parse develop)
RELEASE_BEFORE=$(git rev-parse "$RELEASE_BRANCH")
```

이 값들을 사용자에게 한 번 출력해 두면, 실패 시 복구 명령을 안내하기 쉬움.

## 실행 절차

### 1) main / develop 최신화

```bash
git checkout main
git pull --ff-only origin main
git checkout develop
git pull --ff-only origin develop
git checkout "$RELEASE_BRANCH"
```

ff-only 실패 시 즉시 중단.

### 2) CHANGELOG 에서 태그 메시지 추출

`CHANGELOG.md` 에서 `## [<VERSION>]` 섹션을 추출:

```bash
awk -v v="$VERSION" '
  BEGIN { in_section = 0 }
  /^## \[/ {
    if (in_section) exit
    if ($0 ~ "^## \\[" v "\\]") { in_section = 1; print; next }
  }
  in_section { print }
' CHANGELOG.md > /tmp/release-tag-message.txt
```

추출된 내용을 태그 메시지로 사용. 비어 있으면 사용자에게 알리고 사용자 입력으로 대체.

### 3) git flow release finish 실행

비대화형 실행을 위해 `-m` 또는 `-f <file>` 옵션 사용:

```bash
GIT_MERGE_AUTOEDIT=no git flow release finish -f /tmp/release-tag-message.txt "$VERSION"
```

`git flow release finish` 의 기본 동작:
1. release 브랜치를 main 에 머지 (no-ff).
2. main 에 `v<VERSION>` 태그 생성 (옵션 `-f <file>` 로 메시지 지정 — 일부 git-flow 구현에서는 `-m "msg"` 사용. 환경에 맞춰 fallback).
3. release 브랜치를 develop 에 머지 (no-ff).
4. release 브랜치 삭제.

`-f` 옵션이 지원되지 않는 git-flow 버전이면 fallback:

```bash
TAG_MSG=$(cat /tmp/release-tag-message.txt)
GIT_MERGE_AUTOEDIT=no git flow release finish -m "$TAG_MSG" "$VERSION"
```

**충돌 발생 시 처리 (필수)**:

git flow release finish 가 머지 충돌로 중단되면:

```bash
# 진행 중인 머지 취소
git merge --abort 2>/dev/null || true

# 모든 브랜치를 원상복구
git checkout main
git reset --hard "$MAIN_BEFORE"
git checkout develop
git reset --hard "$DEV_BEFORE"
git checkout "$RELEASE_BRANCH" 2>/dev/null || git checkout -b "$RELEASE_BRANCH" "$RELEASE_BEFORE"
git reset --hard "$RELEASE_BEFORE"

# 잘못 생성된 태그가 있으면 제거
git tag -d "v$VERSION" 2>/dev/null || true
git tag -d "$VERSION" 2>/dev/null || true
```

그리고 사용자에게:
```
✗ release finish 중 충돌 발생 — 원상복구 완료
  main:     <MAIN_BEFORE>
  develop:  <DEV_BEFORE>
  release:  <RELEASE_BEFORE>
  태그(v<VERSION>): 제거됨
  수동으로 충돌을 해결하시고 다시 시도해주세요.
```

라고 보고하고 종료.

### 4) push (성공 시)

`git flow release finish` 종료 시점에는 일반적으로 `develop` 브랜치에 위치하게 된다. 다음 순서로 push 한다:

```bash
git checkout develop
git push origin develop
git checkout main
git push origin main
git push origin "v$VERSION"
git checkout develop
```

순서 고정:
1. **develop push 우선** — release finish 직후 위치한 브랜치이므로 가장 먼저 원격에 반영.
2. main checkout 후 main push — 릴리즈 머지 결과 반영.
3. 태그(`v<VERSION>`) push — main 푸시와 분리하여 명시적으로 처리.
4. 마지막에 develop 으로 돌아와 작업 환경을 복귀.

각 push 가 거부되면(non-ff 등) 절대 강제 푸시하지 말고 사용자에게 보고 후 중단.

### 5) Monday.com 보드 업데이트 (TODO — 후속 작업)

```bash
# TODO: Monday.com 보드 업데이트
# 이 단계는 후속 작업으로 분리되어 있습니다.
# 사용자가 Monday MCP 연동을 명시적으로 요청하면 다음을 구현:
#   - 릴리즈 버전과 매칭되는 board item 상태를 "Released" 로 변경
#   - 릴리즈 노트(CHANGELOG 섹션) 를 item update 로 게시
#   - 관련 Feature ID 들의 status 전환
#
# 현재는 사용자에게 한 줄로 안내만 출력:
echo "ℹ Monday.com 보드 업데이트는 후속 작업입니다 (현재 미연동)."
```

## 출력 형식

성공:
```
✓ release/<VERSION> finish 완료
  merged into: main, develop (no-ff)
  tagged:      v<VERSION>
  pushed:      origin/main, origin/develop, refs/tags/v<VERSION>
  todo:        Monday.com 보드 업데이트 (후속 작업)
```

중단:
```
✗ 중단됨: <사유>
  복구 SHA  → main: <MAIN_BEFORE>, develop: <DEV_BEFORE>, release: <RELEASE_BEFORE>
  자세한 내용은 위 충돌 로그 참조.
```

## 금지 사항

- 강제 푸시 자동 수행 금지.
- 충돌 자동 해결 금지.
- `--no-verify` 사용 금지.
- 태그 형식을 임의로 변경 금지 (기본 `v<VERSION>`).
