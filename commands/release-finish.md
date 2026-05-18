---
allowed-tools: Bash(git:*), Bash(git flow:*), Bash(awk:*), Bash(sed:*), Bash(curl:*), Bash(jq:*), Bash(grep:*), Bash(date:*), Bash(tr:*), Bash(sort:*), Bash(paste:*), Bash(cat:*), Bash(test:*)
description: release 브랜치를 main/develop 에 머지하고 태그 + push + Monday.com Release 보드 기록까지 처리
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

### 5) Monday.com Release 보드에 아이템 등록

**비차단 단계**: 이 단계의 어떤 실패도 release-finish 전체를 실패로 만들지 않는다 (이미 git push 까지 완료된 상태). 실패 시 경고 출력 후 정상 종료.

#### 5-1) 전제 조건 확인

```bash
# 토큰 조회 (env 우선 → 파일 fallback)
MONDAY_TOKEN="${MONDAY_API_TOKEN:-}"
TOKEN_FILE="$HOME/.claude/.gitflow-sph-monday.token"
if [ -z "$MONDAY_TOKEN" ] && [ -f "$TOKEN_FILE" ]; then
  MONDAY_TOKEN=$(tr -d '\r\n' < "$TOKEN_FILE")
fi

MONDAY_STATUS="skip"
MONDAY_DETAIL=""

if [ -z "$MONDAY_TOKEN" ]; then
  MONDAY_DETAIL="토큰 없음 — /monday-token 으로 설정 가능"
elif [ ! -f monday.config.json ]; then
  MONDAY_DETAIL="monday.config.json 없음 — /monday-init 으로 생성 가능"
elif ! command -v curl >/dev/null || ! command -v jq >/dev/null; then
  MONDAY_DETAIL="curl 또는 jq 미설치"
else
  MONDAY_STATUS="run"
fi
```

전제 조건 불충분 시 한 줄 안내 후 6) 단계로:

```bash
[ "$MONDAY_STATUS" = "skip" ] && echo "ℹ monday: skip ($MONDAY_DETAIL)"
```

#### 5-2) CHANGELOG 섹션에서 Feature ID 추출

step 2) 에서 생성한 `/tmp/release-tag-message.txt` 재사용:

```bash
# (feature-id) 형식 추출. SHA(7자 hex) 와 구분하기 위해 8자 이상 순수 숫자만.
FEATURE_IDS=$(grep -oE '\([0-9]{8,}\)' /tmp/release-tag-message.txt 2>/dev/null \
  | tr -d '()' | sort -u | paste -sd, -)
```

비어 있어도 진행 (릴리즈에 Feature ID 없는 케이스 — chore 전용 릴리즈 등).

#### 5-3) 설정 로드

```bash
BOARD_ID=$(jq -r '.release_board_id // empty' monday.config.json)
COL_VERSION=$(jq -r '.columns.version // empty' monday.config.json)
COL_DATE=$(jq -r '.columns.date // empty' monday.config.json)
COL_FIDS=$(jq -r '.columns.feature_ids // empty' monday.config.json)
COL_CHANGELOG=$(jq -r '.columns.changelog // empty' monday.config.json)

# 필수 필드 검증
if [ -z "$BOARD_ID" ] || [ "$COL_VERSION" = "REPLACE_WITH_COLUMN_ID" ]; then
  MONDAY_STATUS="skip"
  MONDAY_DETAIL="monday.config.json 미완성 (placeholder 상태) — /monday-init 재실행 필요"
  echo "ℹ monday: skip ($MONDAY_DETAIL)"
fi
```

#### 5-4) Release 보드에 아이템 생성

```bash
if [ "$MONDAY_STATUS" = "run" ]; then
  TODAY=$(date +%F)
  CHANGELOG_BODY=$(cat /tmp/release-tag-message.txt)

  # column_values 구성: jq 로 동적 키 생성
  COLUMN_VALUES=$(jq -nc \
    --arg v "$VERSION" \
    --arg d "$TODAY" \
    --arg fids "$FEATURE_IDS" \
    --arg cl "$CHANGELOG_BODY" \
    --arg cv "$COL_VERSION" \
    --arg cd "$COL_DATE" \
    --arg cf "$COL_FIDS" \
    --arg cc "$COL_CHANGELOG" \
    '{($cv):$v, ($cd):{date:$d}, ($cf):$fids, ($cc):$cl}')

  # column_values 는 GraphQL JSON 타입이므로 문자열로 직렬화해서 전달
  MUTATION=$(jq -nc \
    --arg b "$BOARD_ID" \
    --arg n "v$VERSION" \
    --arg cv "$COLUMN_VALUES" \
    '{query:"mutation($b:ID!,$n:String!,$cv:JSON!){create_item(board_id:$b,item_name:$n,column_values:$cv){id}}",variables:{b:$b,n:$n,cv:$cv}}')

  RESPONSE=$(curl -sS -X POST https://api.monday.com/v2 \
    -H "Authorization: $MONDAY_TOKEN" \
    -H "Content-Type: application/json" \
    -d "$MUTATION")

  NEW_ITEM_ID=$(echo "$RESPONSE" | jq -r '.data.create_item.id // empty')
  ERROR_MSG=$(echo "$RESPONSE" | jq -r '.errors[0].message // .error_message // empty')

  if [ -n "$NEW_ITEM_ID" ]; then
    MONDAY_STATUS="ok"
    MONDAY_DETAIL="board $BOARD_ID, item $NEW_ITEM_ID"
    echo "✓ monday: Release 아이템 생성 (board $BOARD_ID, item $NEW_ITEM_ID)"
  else
    MONDAY_STATUS="fail"
    MONDAY_DETAIL="${ERROR_MSG:-알 수 없는 오류}"
    echo "⚠ monday: 실패 — $MONDAY_DETAIL (릴리즈는 정상 완료)"
    echo "  응답: $RESPONSE"
  fi
fi
```

### 6) 완료 보고

위에서 수집한 `MONDAY_STATUS` / `MONDAY_DETAIL` 을 출력 형식의 `monday:` 라인에 반영.

## 출력 형식

성공 (Monday 성공):
```
✓ release/<VERSION> finish 완료
  merged into: main, develop (no-ff)
  tagged:      v<VERSION>
  pushed:      origin/main, origin/develop, refs/tags/v<VERSION>
  monday:      Release 아이템 생성 (board <BOARD_ID>, item <NEW_ITEM_ID>)
```

성공 (Monday skip / 실패):
```
✓ release/<VERSION> finish 완료
  merged into: main, develop (no-ff)
  tagged:      v<VERSION>
  pushed:      origin/main, origin/develop, refs/tags/v<VERSION>
  monday:      skip (<사유>)         # 또는: 실패 — <에러 요약>
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
