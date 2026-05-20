---
allowed-tools: Bash(git:*), Bash(command:*)
description: feature 브랜치를 develop 위로 rebase 후 fast-forward 머지하고 로컬/원격에서 삭제
argument-hint: "[feature-id]"
---

## 공통 규칙 (반드시 준수)

- `--no-verify`, `--force` 사용 금지.
- 로컬 브랜치 삭제는 항상 `git branch -d` (소문자) 만 사용. `-D` 로 강제 삭제 금지 — 머지되지 않은 커밋이 남아 있으면 즉시 중단/보고.
- 머지 기준 브랜치 갱신은 `git pull --ff-only` 만.
- **머지는 항상 rebase 후 fast-forward** — 머지 커밋을 만들지 않고 develop 을 선형으로 유지한다.
- 원격 브랜치가 이미 사라진 경우(GitHub auto-delete 등) `git push origin --delete` 의 실패는 정상으로 간주하고 진행한다.
- rebase 도중 충돌 발생 시 자동 해결하지 않고 `git rebase --abort` 후 원상복구 + 사용자 보고.
- 본 명령은 feature 를 develop 위로 rebase 후 fast-forward 머지 → feature 브랜치 로컬/원격 삭제 → develop 으로 복귀한다.

## 인자

`$ARGUMENTS` — 선택적 Feature ID.
- 비어 있지 않으면 `feature/<id>` 형태의 대상 브랜치 이름으로 사용.
- 비어 있으면 다음 우선순위로 결정:
  1. 현재 브랜치가 `feature/<x>` 패턴이면 그 브랜치를 대상으로 사용.
  2. 그 외의 경우 로컬의 `feature/*` 브랜치 목록을 보여주고 `AskUserQuestion` 으로 선택.

## 컨텍스트 수집 (병렬)

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git remote -v
git show-ref --verify --quiet refs/heads/develop
```

검증:
- git 저장소가 아니면 즉시 중단.
- working tree 가 더러우면 중단 후 사용자에게 안내 (stash / 커밋 / 중단 선택).
- `develop` 브랜치가 로컬에 없으면 중단 ("git flow init -d 를 먼저 실행해주세요").
- 원격 `origin` 이 없으면 원격 관련 단계(push develop / 원격 브랜치 삭제)는 자동 skip 하고 사용자에게 알림.

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

로컬 feature 브랜치 후보 조회:

```bash
git branch --list 'feature/*' --format '%(refname:short)'
```

- 결과가 비어 있으면 "정리할 feature 브랜치가 없습니다" 보고 후 종료.
- 후보 1개면 자동 선택해 사용자에게 확인 1회.
- 후보 2개 이상이면 `AskUserQuestion` 으로 목록 제시 후 하나만 선택.

## 검증 단계 (대상 브랜치 확정 후 필수)

### 1) 브랜치 존재 확인

```bash
git show-ref --verify --quiet "refs/heads/$TARGET_BRANCH"
```

실패 시 "로컬에 $TARGET_BRANCH 가 없습니다" 보고 후 종료.

### 2) develop 으로 이동

```bash
CURRENT=$(git branch --show-current)
if [ "$CURRENT" != "develop" ]; then
  git checkout develop
fi
```

### 3) develop 최신화 (원격이 있을 때만)

```bash
git pull --ff-only origin develop
```

ff-only 실패 시 즉시 중단 + 보고. develop 이 비-fast-forward 상태면 사용자가 직접 정리 후 재시도해야 한다.

### 4) feature 를 develop 위로 rebase 후 fast-forward 머지

머지 커밋을 만들지 않고 선형으로 develop 에 얹는다. develop 이 앞서 있어도 rebase 로 feature 를 develop 끝에 올려 fast-forward 가 가능하게 만든다.

#### 4-1) feature 를 develop 위로 rebase

```bash
git checkout "$TARGET_BRANCH"
git rebase develop
```

- 성공 → 4-2 진행.
- 충돌 발생 → 자동 해결 금지. 즉시 복구 후 보고:
  ```bash
  git rebase --abort
  ```
  보고:
  ```
  ✗ 중단됨: $TARGET_BRANCH 를 develop 위로 rebase 중 충돌 발생
    git rebase --abort 로 원래 상태로 복구했습니다.
    수동으로 충돌 해결 후 다시 시도해주세요.

    참고:
      git checkout $TARGET_BRANCH
      git rebase develop
      # 충돌 해결 후
      /feature-finish $TARGET_BRANCH
  ```

#### 4-2) develop 으로 fast-forward 머지

```bash
git checkout develop
git merge --ff-only "$TARGET_BRANCH"
```

- 성공 → 5단계 진행.
- 실패(4-1 rebase 가 성공했다면 이론상 발생하지 않음) → develop 은 변경되지 않은 상태. 보고 후 종료:
  ```
  ✗ 중단됨: develop ← $TARGET_BRANCH fast-forward 머지 실패
    develop 은 변경되지 않았습니다. 상태 확인 후 다시 시도해주세요.
  ```

### 5) develop push (원격이 있을 때만)

```bash
git push origin develop
```

- 성공 → 6단계 진행.
- 실패(원격이 더 앞서 있어 non-ff 등) → 절대 force push 하지 않는다. 다음과 같이 보고하고 종료:
  ```
  ✗ 중단됨: develop push 실패
    로컬 develop 은 머지 완료 상태이며, feature 브랜치는 아직 삭제되지 않았습니다.
    원격 develop 을 동기화한 뒤 다시 시도해주세요.
  ```
  (이때 로컬 머지는 유지되며, 사용자가 push 만 해도 finish 가 완료되는 상태가 된다.)

### 6) 로컬 feature 브랜치 삭제

```bash
git branch -d "$TARGET_BRANCH"
```

- 성공 → 7단계 진행.
- 실패(develop 으로 머지되지 않은 커밋 존재) → **절대 `-D` 로 재시도하지 않는다.** 보고:
  ```
  ✗ $TARGET_BRANCH 에 develop 으로 머지되지 않은 커밋이 남아 있습니다.
    이론상 4단계 머지가 성공했다면 발생하지 않아야 하는 상태입니다.
    수동 확인 후 처리해주세요.

    참고:
      git log develop..$TARGET_BRANCH
  ```

### 7) 원격 feature 브랜치 삭제 (원격이 있을 때만)

원격에 같은 브랜치가 남아 있는지 확인:

```bash
git ls-remote --heads origin "$TARGET_BRANCH"
```

- 결과가 비어 있음 → 이미 제거됨. 건너뜀.
- 결과가 존재 → 삭제:
  ```bash
  git push origin --delete "$TARGET_BRANCH"
  ```
  실패 시 stderr 를 그대로 사용자에게 보여주고 종료 (강제 삭제 재시도 금지).

### 8) 원격 추적 정리

```bash
git fetch --prune origin
```

## 출력 형식

성공:
```
✓ feature/<id> finish 완료
  rebase: feature/<id> onto develop
  merge:  develop ← feature/<id>  (fast-forward, 머지 커밋 없음)
  push:   origin/develop updated
  local:  feature/<id> deleted
  remote: feature/<id> deleted (or already gone)
  pruned: origin tracking refs
  HEAD:   develop
```

중단(충돌):
```
✗ 중단됨: feature/<id> → develop rebase 충돌
  rebase --abort 로 원래 상태로 복구됨.
  충돌 해결 후 다시 시도해주세요.
```

중단(미머지 커밋):
```
✗ 중단됨: 로컬 feature 브랜치에 develop 으로 머지되지 않은 커밋 존재
  diff: git log develop..feature/<id>
```

중단(push 실패):
```
✗ 중단됨: develop push 실패 (로컬 머지는 유지)
  원격 develop 동기화 후 재시도.
```

## 금지 사항

- `git branch -D` 자동 실행 금지.
- 원격 강제 푸시 금지 (`--force`, `--force-with-lease`).
- rebase / 머지 충돌 자동 해결 금지.
- `--no-verify` 사용 금지.
- `--no-ff` 머지 금지 (항상 rebase 후 fast-forward — 머지 커밋을 만들지 않는다).
