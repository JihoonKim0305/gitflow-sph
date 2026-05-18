---
allowed-tools: Bash(git:*), Bash(gh:*), Bash(command:*)
description: 내가 author 인 머지된 feature 브랜치를 일괄 조회 후 한 번에 삭제 (사용자 확인 1회)
---

## 공통 규칙 (반드시 준수)

- `--no-verify`, `--force` 사용 금지.
- 로컬 브랜치 삭제는 항상 `git branch -d` (소문자) 만 사용. `-D` 금지.
- 머지 기준 브랜치 갱신은 `git pull --ff-only` 만.
- **PR 이 머지되지 않은 브랜치는 건드리지 않는다.** 후보 목록에서 자동 제외.
- 원격 브랜치가 이미 사라진 경우(GitHub auto-delete) `git push origin --delete` 실패는 정상으로 간주하고 계속.
- 일괄 삭제 직전 사용자에게 **반드시 1회 확인**을 받는다. 사용자가 거절하거나 응답이 모호하면 어떤 삭제도 수행하지 않는다.

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
- working tree 가 더러우면 중단 후 사용자에게 안내.
- 원격 `origin` 이 존재해야 함.
- `gh` 가 없으면 중단 (PR author / 머지 여부 확인 불가).

## 후보 수집

### 1) 내가 author 인 머지된 PR 목록

```bash
gh pr list \
  --author @me \
  --state merged \
  --json number,title,headRefName,mergedAt \
  --limit 100
```

- `headRefName` 이 `feature/` 로 시작하는 항목만 사용.
- 비어 있으면 "정리할 머지된 PR 이 없습니다." 보고 후 종료.

### 2) 로컬에 실제로 존재하는 feature 브랜치

```bash
git branch --list 'feature/*' --format '%(refname:short)'
```

- 비어 있으면 "로컬에 남아 있는 feature 브랜치가 없습니다." 보고 후 종료.

### 3) 교집합 = 정리 대상

두 목록의 교집합만 cleanup 후보로 채택. 결과가 0개면 "정리할 대상이 없습니다." 보고 후 종료.

각 후보에 대해 다음 정보를 사용자에게 표 형태로 보여준다:

```
정리 후보 (총 <N> 개):

  feature/<id-1>   PR #<num> merged <mergedAt>   <title 일부>
  feature/<id-2>   PR #<num> merged <mergedAt>   <title 일부>
  ...
```

## 사용자 확인 (필수)

`AskUserQuestion` 으로 1회 확인:

- 옵션 1: `위 N 개 전부 삭제` (Recommended)
- 옵션 2: `중단`

사용자가 "전부 삭제" 를 선택한 경우에만 다음 단계로 진행. 그 외에는 어떤 삭제도 수행하지 않는다.

(부분 선택은 이 명령에서 지원하지 않는다. 일부만 지우고 싶다면 `/feature-cleanup <id>` 를 개별 호출하라고 안내.)

## 실행 절차

### 1) 현재 브랜치가 후보에 포함되면 develop 으로 이동

```bash
CURRENT=$(git branch --show-current)
case "<후보 목록>" in
  *"$CURRENT"*) git checkout develop ;;
esac
```

### 2) develop 최신화

```bash
git pull --ff-only origin develop
```

ff-only 실패 시 즉시 중단 + 보고. develop 이 최신이 아니면 `git branch -d` 가 "이미 머지됨" 으로 판단하지 못한다.

### 3) 후보 순회 — 로컬 + 원격 삭제

각 `TARGET_BRANCH` 에 대해:

```bash
# 로컬 삭제 (안전 모드 -d)
git branch -d "$TARGET_BRANCH"
```

- 성공 → 결과 리스트에 `local: deleted` 기록 후 원격 삭제로 진행.
- 실패(미머지 커밋 존재) → **`-D` 재시도 금지.** 결과 리스트에 `local: SKIPPED (unmerged commits)` 기록하고 해당 브랜치의 원격 삭제도 건너뜀. 다음 후보로 넘어감.

```bash
# 원격에 남아 있는지 확인
git ls-remote --heads origin "$TARGET_BRANCH"
```

- 비어 있음 → `remote: already gone` 기록.
- 존재 → 삭제:
  ```bash
  git push origin --delete "$TARGET_BRANCH"
  ```
  성공 → `remote: deleted` 기록. 실패 → stderr 요약을 `remote: FAILED (<reason>)` 로 기록 (재시도 금지).

각 후보 처리는 독립적이다. 한 브랜치가 실패해도 나머지 후보 처리는 계속 진행.

### 4) 원격 추적 정리

전체 순회 종료 후 한 번:

```bash
git fetch --prune origin
```

## 출력 형식

성공/부분 성공:

```
✓ feature cleanup-all 완료 (성공 <S> / 건너뜀 <K> / 실패 <F>)

  feature/<id-1>   local: deleted   remote: deleted
  feature/<id-2>   local: deleted   remote: already gone
  feature/<id-3>   local: SKIPPED   remote: SKIPPED   reason: unmerged commits
  ...

  pruned: origin tracking refs
```

전체 중단(ff-only 실패 등):

```
✗ 중단됨: <사유>
  develop 최신화에 실패했습니다. 수동 확인 후 다시 시도해주세요.
```

후보 없음:

```
ℹ 정리할 대상이 없습니다.
  - 내가 author 인 머지된 PR: <X> 개
  - 로컬 feature 브랜치:       <Y> 개
  - 교집합:                    0 개
```

## 금지 사항

- `git branch -D` 자동 실행 금지.
- 원격 강제 삭제 재시도 금지.
- 사용자 확인 없이 삭제 진행 금지.
- 한 브랜치 실패를 이유로 나머지 후보 처리 중단 금지(부분 진행이 더 안전).
- `--no-verify` 사용 금지.
