---
allowed-tools: Bash(git:*), Bash(gh:*), Bash(command:*)
description: PR 머지된 feature 브랜치를 develop 으로 복귀하고 로컬/원격에서 안전하게 삭제 (git flow feature finish 의 PR 기반 대응)
argument-hint: "[feature-id]"
---

## 공통 규칙 (반드시 준수)

- `--no-verify`, `--force` 사용 금지.
- 로컬 브랜치 삭제는 항상 `git branch -d` (소문자) 만 사용. `-D` 로 강제 삭제 금지 — 미머지 커밋이 남아 있으면 즉시 중단/보고.
- 머지 기준 브랜치 갱신은 `git pull --ff-only` 만.
- **PR 이 머지되지 않은 브랜치는 절대 삭제하지 않는다.** PR 이 open/closed(unmerged) 상태이면 중단하고 사용자에게 보고.
- 원격 브랜치가 이미 사라진 경우(GitHub auto-delete 등) `git push origin --delete` 의 실패는 정상으로 간주하고 진행한다.
- 본 명령은 PR 기반 워크플로우에서 `git flow feature finish` 의 마무리 단계(develop 복귀 + feature 브랜치 삭제)에 해당한다. develop 머지 자체는 PR 머지로 이미 처리되어 있어야 한다.

## 인자

`$ARGUMENTS` — 선택적 Feature ID.
- 비어 있지 않으면 그대로 `feature/<id>` 형태의 대상 브랜치 이름으로 사용.
- 비어 있으면 다음 우선순위로 결정:
  1. 현재 브랜치가 `feature/<x>` 패턴이면 그 브랜치를 대상으로 사용.
  2. 그 외의 경우 "내가 author 이고 머지된 PR" 후보를 조회한 뒤 사용자에게 선택을 받는다.

## 컨텍스트 수집 (병렬)

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git remote -v
command -v gh
```

검증:
- git 저장소가 아니면 즉시 중단.
- working tree 가 더러우면 중단 후 사용자에게 안내 (stash / 커밋 / 중단 선택).
- 원격 `origin` 이 존재해야 함.
- `gh` 가 없으면 머지 여부 확인이 불가능하므로 중단하고 사용자에게 알림.

## 대상 결정

### 인자가 있는 경우

```
TARGET_BRANCH = "feature/$ARGUMENTS"
```

### 인자가 없고 현재 브랜치가 feature 인 경우

```
TARGET_BRANCH = $(git branch --show-current)
```

### 인자가 없고 현재 브랜치가 feature 가 아닌 경우

내가 author 인 머지된 PR 후보 조회:

```bash
gh pr list \
  --author @me \
  --state merged \
  --json number,title,headRefName,mergedAt \
  --limit 30
```

- 결과가 비어 있으면 "정리할 대상이 없습니다" 라고 보고하고 종료.
- `headRefName` 이 `feature/` 로 시작하는 항목만 필터링.
- 로컬에 해당 브랜치가 실제로 존재하는 것만 추리기 위해 다음을 추가로 사용:

```bash
git branch --list 'feature/*' --format '%(refname:short)'
```

- 두 목록의 교집합을 후보로 제시. 후보가 0개면 "로컬에 남은 머지된 feature 브랜치가 없습니다" 보고 후 종료.
- 후보 1개면 자동 선택해 사용자에게 확인 1회.
- 후보 2개 이상이면 `AskUserQuestion` 으로 목록을 보여주고 하나만 선택.

## 검증 단계 (대상 브랜치 확정 후 필수)

### 1) 브랜치 존재 확인

```bash
git show-ref --verify --quiet "refs/heads/$TARGET_BRANCH"
```

실패하면 "로컬에 $TARGET_BRANCH 가 없습니다" 라고 보고 후 종료 (원격만 남은 경우는 별도 처리 — 아래 6단계 참고).

### 2) PR 머지 여부 확인

```bash
gh pr list \
  --head "$TARGET_BRANCH" \
  --state all \
  --json number,state,mergedAt,author \
  --limit 5
```

판정:
- 결과가 비어 있음 → PR 자체가 없음. "$TARGET_BRANCH 와 연결된 PR 을 찾을 수 없습니다. /feature-pr 이 완료되지 않았거나 다른 브랜치로 PR 이 생성되었을 수 있습니다." 보고 후 종료.
- 가장 최근 PR 의 `state` 가 `MERGED` 가 아니면 → 중단:
  ```
  ✗ feature/<id> 의 PR(#<num>) 이 아직 머지되지 않았습니다 (state: <state>).
    finish 는 PR 머지 이후에만 수행할 수 있습니다.
  ```
- `state == MERGED` 이면 머지 SHA / 머지 시각을 사용자에게 출력하고 계속.

### 3) 현재 브랜치가 대상이면 develop 으로 이동

```bash
CURRENT=$(git branch --show-current)
if [ "$CURRENT" = "$TARGET_BRANCH" ]; then
  git checkout develop
fi
```

### 4) develop 최신화

```bash
git pull --ff-only origin develop
```

ff-only 실패 시 즉시 중단 + 보고. develop 이 충분히 최신화되어야 `git branch -d` 가 "이미 머지됨" 으로 판단할 수 있다.

### 5) 로컬 브랜치 삭제

```bash
git branch -d "$TARGET_BRANCH"
```

- 성공 → 6단계 진행.
- 실패(미머지 커밋 존재) → **절대 `-D` 로 재시도하지 않는다.** 다음과 같이 보고하고 종료:
  ```
  ✗ $TARGET_BRANCH 에 develop 으로 머지되지 않은 커밋이 남아 있습니다.
    PR 은 머지되었지만 로컬 브랜치에 추가 커밋이 있을 수 있습니다.
    수동 확인 후 git branch -D 로 강제 삭제하거나 push 후 다시 시도해주세요.

    참고:
      git log develop..$TARGET_BRANCH
  ```

### 6) 원격 브랜치 삭제

원격에 같은 브랜치가 남아 있는지 확인:

```bash
git ls-remote --heads origin "$TARGET_BRANCH"
```

- 결과가 비어 있음 → GitHub auto-delete 등으로 이미 제거됨. 건너뜀.
- 결과가 존재 → 삭제:
  ```bash
  git push origin --delete "$TARGET_BRANCH"
  ```
  실패 시 stderr 를 그대로 사용자에게 보여주고 종료 (강제 삭제 재시도 금지).

### 7) 원격 추적 정리

```bash
git fetch --prune origin
```

## 출력 형식

성공:
```
✓ feature/<id> finish 완료
  PR:     #<num> merged at <mergedAt>
  local:  deleted
  remote: deleted (or already gone)
  pruned: origin tracking refs
```

중단(머지 안 됨):
```
✗ 중단됨: feature/<id> 의 PR 이 아직 머지되지 않음
  PR #<num> state=<state>
  PR 머지 후 다시 시도해주세요.
```

중단(미머지 커밋):
```
✗ 중단됨: 로컬에 develop 으로 머지되지 않은 커밋 존재
  diff:   git log develop..feature/<id>
  강제 삭제가 필요하면 사용자가 직접 git branch -D 를 실행해주세요.
```

## 금지 사항

- `git branch -D` 자동 실행 금지.
- 원격 강제 푸시/삭제 재시도 금지.
- PR 머지 여부 확인 없이 삭제 금지.
- `--no-verify` 사용 금지.
