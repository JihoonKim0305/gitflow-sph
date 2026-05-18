---
allowed-tools: Bash(git:*), Bash(gh:*), Bash(GIT_SEQUENCE_EDITOR=*), Bash(GIT_EDITOR=*)
description: feature 브랜치의 모든 커밋을 squash → develop 기준 rebase → push → PR 생성
---

## 공통 규칙 (반드시 준수)

- `--no-verify`, `--force`, `--force-with-lease` 사용 금지. 강제 푸시가 필요해 보이면 항상 사용자에게 묻는다.
- 머지 기준 브랜치 갱신은 `git pull --ff-only` 만 사용.
- 충돌 발생 시 자동 해결 금지. `git rebase --abort` 로 즉시 원상복구하고 사용자에게 보고 후 중단.
- `git flow feature finish` 는 **사용하지 않는다** — 본 워크플로우는 PR 기반이라 develop 머지는 PR 머지로 처리.

## 컨텍스트 수집 (병렬)

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git remote -v
git rev-parse --verify HEAD
```

검증:
- 현재 브랜치가 `feature/<id>` 패턴이어야 함. 아니면 즉시 중단.
- working tree 가 깨끗해야 함. 더러우면 중단하고 사용자에게 알림.
- 원격 `origin` 이 있어야 함.

`FEATURE_BRANCH = git branch --show-current` 의 결과로 저장.  
`FEATURE_ID = FEATURE_BRANCH` 에서 `feature/` 접두사 제거한 부분.

## 실행 절차

### 1) 원상복구용 SHA 백업

```bash
PRE_FINISH_SHA=$(git rev-parse HEAD)
```

이후 단계에서 문제가 생기면 `git reset --hard "$PRE_FINISH_SHA"` 로 되돌릴 수 있도록 사용자에게 출력.

### 2) develop 최신화

```bash
git checkout develop
git pull --ff-only origin develop
git checkout "$FEATURE_BRANCH"
```

ff-only 실패 시 즉시 중단 + 보고.

### 3) develop 기준 rebase

```bash
git rebase develop
```

충돌 발생 시:

```bash
git rebase --abort
```

이후 사용자에게 "충돌이 발생해 자동 해결하지 않고 중단했습니다. 수동으로 rebase 후 다시 호출해주세요." 라고 보고하고 종료.

### 4) 커밋 개수 계산 후 squash rebase

```bash
BASE=$(git merge-base HEAD develop)
N=$(git rev-list --count "$BASE"..HEAD)
```

- `N <= 1` 이면 squash 불필요. 5단계로 건너뜀.
- `N >= 2` 이면 비대화형 interactive rebase 실행:

```bash
GIT_SEQUENCE_EDITOR="sed -i.bak -e '2,\$s/^pick /squash /'" \
GIT_EDITOR=true \
git rebase -i HEAD~"$N"
```

설명:
- `GIT_SEQUENCE_EDITOR` 가 todo 리스트를 받아 2번째 줄부터 끝까지를 `squash` 로 변환 (첫 줄은 `pick` 유지).
- `GIT_EDITOR=true` 로 squash 합산 메시지 편집 단계를 비대화형으로 통과시킴 (원본 메시지가 모두 보존됨).
- macOS sed 호환을 위해 `-i.bak` 사용. 끝나면 `.bak` 파일 정리는 git 이 임시 디렉터리에서 처리하므로 별도 작업 불필요.

충돌 시 동일하게 `git rebase --abort` 후 중단/보고.

### 5) Squash 결과 메시지 정리

```bash
git log -1 --pretty=%B
```

결과를 사용자에게 보여주고, Feature ID 가 description 앞에 명시되어 있는지 확인.

명시되어 있지 않거나 메시지가 누적된 raw 형태로 어지러우면, 다음 형식으로 amend:

```
<type>(<scope>): <feature-id> <통합 subject>

<body — 이 브랜치에서 한 일과 그 이유를 3~6줄로 정리>
```

사용자에게 amend 메시지 초안을 한 번 보여주고 동의를 받은 뒤:

```bash
git commit --amend -m "$(cat <<'EOF'
<final message>
EOF
)"
```

### 6) 원격 상태 확인 후 push

원격에 같은 브랜치가 이미 존재하는지 확인:

```bash
git ls-remote --heads origin "$FEATURE_BRANCH"
```

- **결과가 비어 있음 (원격에 없음)** → 일반 push 진행:
  ```bash
  git push -u origin "$FEATURE_BRANCH"
  ```
- **결과가 존재함 (원격에 이미 있음)** → squash rebase 후의 push 는 강제 푸시가 필요함. **자동으로 진행하지 않는다.** 다음 메시지를 출력하고 중단:
  ```
  ⚠ 원격에 feature/<id> 가 이미 존재합니다.
    squash rebase 결과를 푸시하려면 강제 푸시가 필요합니다.
    이 플러그인은 자동으로 force push 하지 않습니다.

    선택지:
      a) git push --force-with-lease origin <branch>  (사용자가 직접 실행)
      b) PR 을 닫고 새 브랜치/PR 로 진행
      c) 중단하고 PRE_FINISH_SHA(<sha>) 로 reset --hard 후 재구성

    어떻게 진행할지 알려주세요.
  ```
  사용자 응답을 받기 전까지는 아무 것도 하지 않는다.

### 7) PR 생성

`gh` CLI 가 설치되어 있는지 확인:

```bash
command -v gh
```

없으면 사용자에게 알리고 push 까지만 완료한 상태로 종료.

설치되어 있으면:

```bash
gh pr create \
  --base develop \
  --head "$FEATURE_BRANCH" \
  --title "<squashed commit subject>" \
  --body "$(cat <<'EOF'
## Summary
- <변경 요약 — squashed 커밋 메시지 기반 1~3줄>

## Why
- <변경한 이유 / 배경>

## Test Plan
- [ ] <확인 포인트 1>
- [ ] <확인 포인트 2>

## Related
- Feature ID: <feature-id>
EOF
)"
```

- title 은 squash 후 커밋 subject 와 동일하게.
- PR 생성 후 반환되는 URL 을 사용자에게 출력.

## 출력 형식

성공:
```
✓ feature/<id> 정리 완료
  squashed: <N> commits → 1
  rebased onto: develop @ <sha>
  pushed: origin/feature/<id>
  PR: <url>
  rollback: git reset --hard <PRE_FINISH_SHA>
```

중단(충돌/원격 충돌):
```
✗ 중단됨: <사유>
  현재 상태: <branch> @ <sha>
  복구: git reset --hard <PRE_FINISH_SHA>
```

## 금지 사항

- `git flow feature finish` 실행 금지 (develop 머지는 PR 머지로).
- 강제 푸시 자동 수행 금지.
- 충돌 자동 해결 금지.
- `--no-verify` 사용 금지.
