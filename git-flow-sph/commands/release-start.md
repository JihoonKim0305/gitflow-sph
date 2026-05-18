---
allowed-tools: Bash(git:*), Bash(git flow:*), Bash(node:*), Bash(jq:*), Bash(cat:*)
description: SemVer 자동 산정 + git flow release start + package.json/CHANGELOG 갱신
argument-hint: "[--major|--minor|--patch] (선택)"
---

## 공통 규칙 (반드시 준수)

- `--no-verify`, `--force`, `--force-with-lease` 금지.
- `git pull --ff-only` 만 사용. 충돌/non-ff 발생 시 즉시 중단.
- 충돌 자동 해결 금지.
- 커밋 메시지에 `Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>` 트레일러 필수.

## 컨텍스트 수집 (병렬)

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git remote -v
git tag --list --sort=-v:refname | head -n 5
git describe --tags --abbrev=0 2>/dev/null
test -f package.json && cat package.json | head -n 20
test -f CHANGELOG.md && head -n 30 CHANGELOG.md
```

확인:
- working tree 깨끗.
- git-flow 초기화 완료 (`.git/config` 의 `[gitflow "branch"]`).
- `package.json` 존재 여부 — 없어도 진행 가능, 있으면 version 필드를 갱신할 대상.
- `CHANGELOG.md` 존재 여부 — 없으면 Keep a Changelog 헤더로 새로 생성.

## SemVer 산정 절차

### 1) 현재 버전 확인

우선순위:
1. `git describe --tags --abbrev=0` 의 결과 (있으면 가장 신뢰).
2. `package.json` 의 `version` 필드.
3. 둘 다 없으면 `0.0.0` 으로 가정.

`vX.Y.Z` 형태이면 `v` 접두사를 떼고 `X.Y.Z` 로 보관.

### 2) develop 에 들어간 신규 커밋 분석

```bash
git fetch origin main develop --tags
LAST_TAG=$(git describe --tags --abbrev=0 origin/main 2>/dev/null || echo "")
RANGE="${LAST_TAG:+$LAST_TAG..}origin/develop"
git log --no-merges --format='%h%x09%s%x09%b%x00' "$RANGE"
```

각 커밋의 subject + body 를 Conventional Commits 로 파싱하여 분류:

- `BREAKING CHANGE:` 가 body 에 있거나 type 뒤에 `!` (예: `feat!:`) → **MAJOR**
- `feat` 가 하나라도 있음 → **MINOR**
- `fix`, `perf`, `refactor` 만 있음 → **PATCH**
- `docs`, `chore`, `test`, `style`, `ci` 만 있으면 PATCH 로 처리 (배포 가치 없으면 사용자 확인).

### 3) 다음 버전 후보 제시

현재 `X.Y.Z` 기준으로:
- MAJOR: `(X+1).0.0`
- MINOR: `X.(Y+1).0`
- PATCH: `X.Y.(Z+1)`

**`AskUserQuestion` 으로 후보 1~3개를 제시하고 사용자 확인을 받는다.** 자동 분석에 기반한 추천을 첫 번째 옵션으로 두고 "(Recommended)" 표기.

사용자가 인자로 `--major`/`--minor`/`--patch` 를 명시했다면 그것을 우선 사용.

확정된 버전을 `NEW_VERSION` 으로 저장.

## 실행 절차

### 4) main / develop 최신화

```bash
git checkout main
git pull --ff-only origin main
git checkout develop
git pull --ff-only origin develop
```

ff-only 실패 시 즉시 중단 + 보고.

### 5) git flow release 시작

```bash
git flow release start "$NEW_VERSION"
```

이후 현재 브랜치는 `release/$NEW_VERSION` 이 됨.

### 6) package.json 버전 갱신 (있을 때만)

`jq` 가 있으면:

```bash
tmp=$(mktemp)
jq --arg v "$NEW_VERSION" '.version = $v' package.json > "$tmp" && mv "$tmp" package.json
```

`jq` 가 없으면 `node` 로:

```bash
node -e 'const fs=require("fs");const p=JSON.parse(fs.readFileSync("package.json"));p.version=process.argv[1];fs.writeFileSync("package.json",JSON.stringify(p,null,2)+"\n")' "$NEW_VERSION"
```

`package-lock.json` 이 함께 있으면 `npm version` 류 대신 동일한 방법으로 `version` / `packages[""].version` 두 곳을 갱신. (없으면 건너뜀.)

### 7) CHANGELOG.md 갱신

[Keep a Changelog](https://keepachangelog.com/) 1.1.0 형식.

파일이 없으면 다음 헤더로 생성:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
```

새 버전 섹션을 `# Changelog` 헤더 바로 뒤(혹은 첫 `## [` 섹션 앞)에 삽입:

```markdown
## [<NEW_VERSION>] - <YYYY-MM-DD>

### Added
- ...

### Changed
- ...

### Fixed
- ...

### Removed
- ...

### Deprecated
- ...

### Security
- ...

### Internal
- ...
```

각 카테고리 매핑:

| Conventional type | 카테고리 |
|---|---|
| `feat` | Added |
| `feat!` / `BREAKING CHANGE` | Changed (상단에 `**BREAKING**` 라벨) |
| `fix` | Fixed |
| `perf` | Changed |
| `refactor` | Internal |
| `docs` / `test` / `chore` / `style` / `ci` | Internal (한 줄로 묶어 요약) |
| `revert` | Removed |
| `deprecate` 키워드 | Deprecated |
| `sec`/`security` 라벨 | Security |

규칙:
- 빈 카테고리는 출력하지 않는다.
- 항목은 commit subject 에서 `<type>(scope): <feature-id> ` 접두를 제거하고 사람이 읽을 수 있게 정리.
- 항목 끝에 `(<feature-id>)` 를 괄호로 붙임. 없으면 단축 SHA `(abc1234)`.
- `Internal` 은 5개 넘으면 "문서/리팩토링/내부 정리 외 N건" 으로 압축.

### 8) 릴리즈 준비 커밋

```bash
git add package.json package-lock.json CHANGELOG.md
git commit -m "$(cat <<'EOF'
chore(release): <NEW_VERSION>

- bump version to <NEW_VERSION>
- update CHANGELOG with release notes

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

hook 실패 시 즉시 중단, `--no-verify` 금지.

## 출력 형식

성공:
```
✓ release/<NEW_VERSION> 시작 완료
  base: develop @ <sha>
  bumped: <OLD_VERSION> → <NEW_VERSION> (<MAJOR|MINOR|PATCH>)
  changelog: <N> entries added across <K> categories
  next: 추가 수정 → 일반 커밋, 릴리즈 마무리 → /release-finish
```

## 금지 사항

- `git push` 를 이 단계에서 호출하지 않음. push 는 `/release-finish` 의 책임.
- 사용자 확인 없이 MAJOR 자동 bump 금지.
- 한 번에 여러 버전 bump 금지.
