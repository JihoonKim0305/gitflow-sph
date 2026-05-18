---
allowed-tools: Bash(git:*)
description: 변경 사항을 Conventional Commits 형식으로 커밋한다 (Feature ID 자동 포함)
argument-hint: "[메시지 힌트]"
---

## 공통 규칙 (반드시 준수)

- `--no-verify` 절대 금지. pre-commit / commit-msg hook 실패 시 그 사실만 보고하고 중단한다 (재시도/우회 없음).
- 한 커밋 = 하나의 논리적 변경. 여러 의도가 섞여 있으면 분리.
- Feature ID는 description 맨 앞에 명시한다.
- `git add -A` / `git add .` 금지 — 명시적으로 파일을 지정해 스테이징한다.
- 시크릿 의심 파일(`.env`, `*credentials*`, `*.pem`, `*.key`) 은 사용자 명시 동의 없이는 스테이징하지 않는다.

## 인자

`$ARGUMENTS` — 선택적인 메시지 힌트(사용자가 의도를 짧게 알려줄 때). 비어 있을 수 있다.

## 컨텍스트 수집 (반드시 병렬 실행)

```bash
git branch --show-current
git status --short
git diff --staged
git diff
git log -n 5 --oneline
```

수집 후 다음을 결정한다:

1. **Feature ID 추출**: 현재 브랜치 이름이 `feature/<id>` 패턴인지 확인.
   - `feature/` 가 없으면 즉시 중단하고 사용자에게 "현재 feature 브랜치가 아닙니다" 보고.
   - `<id>` 부분을 그대로 Feature ID로 사용한다 (숫자든 slug든 무관).
2. **변경 범위 분석**: staged 가 비어 있고 unstaged 만 있으면, 어떤 파일을 커밋할지 결정한 뒤 사용자에게 짧게 보여주고 동의를 받아 `git add <files>` 한다.
3. **다중 의도 감지**: diff에 무관해 보이는 변경이 섞여 있으면(예: 기능 코드 + 다른 디렉터리의 포맷팅) 사용자에게 "이 커밋들을 분리할까요?"라고 한 번 묻는다. 동의 시 분리 커밋 plan을 제시.

## 커밋 메시지 작성 규칙

포맷:

```
<type>[(<scope>)]: <feature-id> <subject>

<body — 왜 변경했는지에 초점. 무엇을 했는지는 diff가 말해줌>
```

타입 자동 선택 (diff 분석 기반):

| 변경 성격 | type |
|---|---|
| 새로운 기능/엔드포인트/UI 추가 | `feat` |
| 버그 수정 | `fix` |
| 내부 구조 변경, 동작 동일 | `refactor` |
| 문서만 | `docs` |
| 테스트 코드만 | `test` |
| 빌드/의존성/도구 설정 | `chore` |
| 포맷팅/공백/세미콜론 | `style` |
| 성능 개선 | `perf` |
| CI 설정 | `ci` |

- `scope` 는 변경된 디렉터리/모듈 이름이 명확할 때만 추가. 애매하면 생략.
- subject 는 명령형 현재시제, 한국어 권장(영어도 허용). 50자 내외 권장, 최대 72자.
- body 는 한 줄 빈 줄로 분리. 2~5줄 정도로 "왜" 를 설명.
- 사용자 메시지 힌트(`$ARGUMENTS`)가 있으면 그것을 우선 반영해서 subject/body를 작성.

## 실행 절차

1. 위 컨텍스트 수집 → 분석.
2. 메시지 초안을 사용자에게 한 번 보여준다 (긴 응답 금지, 코드블록 하나로 짧게).
3. 사용자가 OK 하면 다음을 단일 메시지에서 병렬 실행:
   ```bash
   git add <specific-files>
   git commit -m "$(cat <<'EOF'
   <type>(<scope>): <feature-id> <subject>

   <body>
   EOF
   )"
   git status --short
   ```
4. hook 실패 시:
   - 출력 전부를 사용자에게 그대로 보여주고 중단.
   - **절대** `--no-verify` 로 재시도하지 않음.
   - 사용자가 직접 수정 후 재호출하도록 안내.

## 분리 커밋이 필요한 경우

여러 논리 단위를 하나의 슬래시 호출에서 처리해야 한다면:
- 각 커밋에 대해 위 절차를 순차적으로 반복.
- 각 커밋의 메시지 초안을 모두 한 번에 보여준 뒤 일괄 동의를 받고 진행.

## 출력 형식

성공:
```
✓ committed: <type>(<scope>): <feature-id> <subject>
  files: <count> changed
  next: 추가 커밋 → /feature-commit, PR 생성 → /feature-pr
```

실패(hook):
```
✗ commit blocked by hook (<hook-name>)
  <stderr 요약>
  hook을 통과시킨 뒤 다시 호출해주세요.
```
