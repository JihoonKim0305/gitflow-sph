# git-flow-sph

> git-flow + Conventional Commits + SemVer 자동화 Claude Code 플러그인 (SPH)

Feature 개발과 Release 작업을 두 갈래의 슬래시 커맨드로 자동화합니다.
모든 커맨드는 **PR 기반 워크플로우**를 전제로 합니다 (`git flow feature finish`는 사용하지 않음).

---

## 설치

### GitHub 저장소에서 바로 설치 (권장)

Claude Code 안에서 두 줄만 실행하면 됩니다.

```
/plugin marketplace add JihoonKim0305/gitflow-sph
/plugin install git-flow-sph@gitflow-sph
```

- 1줄: 이 저장소를 플러그인 마켓플레이스로 등록 (루트의 [.claude-plugin/marketplace.json](.claude-plugin/marketplace.json) 매니페스트를 읽음)
- 2줄: 마켓플레이스의 `git-flow-sph` 플러그인을 설치

설치 후 `/plugin` 으로 활성 상태를 확인할 수 있고, 슬래시 커맨드 `/feature-start`, `/feature-commit`, `/feature-finish`, `/release-start`, `/release-finish` 가 노출됩니다.

업데이트는 `/plugin marketplace update gitflow-sph` 후 `/plugin install git-flow-sph@gitflow-sph` 를 다시 실행하면 됩니다.

### 로컬 디렉터리로 설치 (오프라인/개발용)

```bash
git clone https://github.com/JihoonKim0305/gitflow-sph.git
# Claude Code 에서:
#   /plugin marketplace add <클론한 절대경로>
#   /plugin install git-flow-sph@gitflow-sph
```

### 의존성

- `git` (2.30+ 권장)
- `git-flow` (AVH edition 권장: `brew install git-flow-avh`)
- `gh` (GitHub CLI) — PR 생성용. 없으면 `/feature-finish` 가 push 까지만 수행하고 안내.
- `jq` 또는 `node` — `package.json` 버전 갱신용 (선택).

저장소 초기 세팅:

```bash
git flow init -d   # 기본값 사용 (main / develop)
gh auth status     # 인증 확인
```

---

## 슬래시 커맨드

| 커맨드 | 용도 |
|---|---|
| `/feature-start <id-or-name>` | develop 최신화 후 `git flow feature start` |
| `/feature-commit [메시지 힌트]` | Conventional Commits 형식 커밋 (Feature ID + Co-Authored-By 자동 포함) |
| `/feature-finish` | squash rebase → push → `gh pr create` |
| `/release-start [--major\|--minor\|--patch]` | SemVer 산정 → `git flow release start` → package.json/CHANGELOG 갱신 |
| `/release-finish` | release → main/develop 머지 + 태그 + push |

---

## 전체 워크플로우 흐름도

```mermaid
flowchart TD
    classDef cmd fill:#1f6feb,stroke:#0b3d91,color:#fff,font-weight:bold
    classDef branch fill:#fff3b0,stroke:#b08900,color:#3d2c00
    classDef action fill:#e6f4ea,stroke:#1e8e3e,color:#0b3d18
    classDef guard fill:#fde7e9,stroke:#c0392b,color:#5b1a12

    DEV[("develop")]:::branch
    MAIN[("main")]:::branch

    %% ===== Feature flow =====
    subgraph FEATURE["Feature 플로우 (PR 기반)"]
        direction TB
        FS["/feature-start &lt;id&gt;"]:::cmd
        FB["feature/&lt;id&gt;"]:::branch
        FC["/feature-commit"]:::cmd
        FCMSG["Conventional Commits 형식<br/>feat(scope): &lt;id&gt; subject<br/>+ Co-Authored-By"]:::action
        FF["/feature-finish"]:::cmd
        FSQ["develop 기준 rebase<br/>+ 비대화형 squash"]:::action
        FPSH["push -u origin feature/&lt;id&gt;"]:::action
        FPR(["gh pr create --base develop"]):::action

        FS --> FB
        FB --> FC
        FC --> FCMSG
        FCMSG -. 반복 .-> FC
        FCMSG --> FF
        FF --> FSQ --> FPSH --> FPR
    end

    %% ===== Release flow =====
    subgraph RELEASE["Release 플로우 (SemVer + CHANGELOG)"]
        direction TB
        RS["/release-start"]:::cmd
        RANA["develop 커밋 분석<br/>feat! / feat / fix·perf·refactor"]:::action
        RVER["SemVer 산정<br/>MAJOR / MINOR / PATCH"]:::action
        RB["release/&lt;ver&gt;"]:::branch
        RBUMP["package.json version bump<br/>CHANGELOG.md 섹션 추가<br/>chore(release) 커밋"]:::action
        RFIN["/release-finish"]:::cmd
        RMERGE["main 머지(no-ff)<br/>v&lt;ver&gt; 태그<br/>develop 머지(no-ff)"]:::action
        RPUSH["push main / develop / tag"]:::action

        RS --> RANA --> RVER --> RB --> RBUMP --> RFIN --> RMERGE --> RPUSH
    end

    %% ===== Cross edges =====
    DEV -- "pull --ff-only" --> FS
    FPR -. "리뷰 후 머지" .-> DEV
    DEV -- "pull --ff-only" --> RS
    RPUSH --> MAIN
    RPUSH -. "동기화" .-> DEV

    %% ===== Safety guard =====
    GUARD["안전 규칙<br/>--no-verify 금지 · 강제 push 금지<br/>충돌 자동 해결 금지 · --ff-only 만 허용"]:::guard
    FF -.- GUARD
    RFIN -.- GUARD
```

- **파란 박스** = 슬래시 커맨드, **노란 박스** = git 브랜치, **초록 박스** = 자동 수행 작업, **빨간 박스** = 모든 커맨드 공통 안전 규칙.
- `develop` / `main` 은 항상 `--ff-only` 로만 최신화하며, 커맨드 어떤 단계에서도 충돌·non-ff 가 발생하면 즉시 중단됩니다.

---

## 사용 시나리오

### Feature 시나리오

```bash
# 1. 작업 시작
/feature-start 11754215659
#   → develop pull --ff-only
#   → git flow feature start 11754215659
#   → 현재 브랜치: feature/11754215659

# 2. 코드 수정 후 커밋 (필요한 만큼 반복)
/feature-commit "토큰 만료 시 401 처리 추가"
#   → feat(auth): 11754215659 handle 401 on expired token
#     <body>
#     Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>

# 3. 마무리: squash + push + PR
/feature-finish
#   → develop pull --ff-only
#   → git rebase develop
#   → git rebase -i HEAD~N (비대화형 squash)
#   → git push -u origin feature/11754215659
#   → gh pr create --base develop
```

원격에 같은 브랜치가 이미 있는 경우 `/feature-finish` 는 자동으로 force push 하지 않고 **사용자에게 어떻게 진행할지 묻고 중단**합니다.

### Release 시나리오

```bash
# 1. 릴리즈 준비
/release-start
#   → develop 커밋 분석 → 다음 버전 후보 1~3개 제시
#   → 사용자 확정 → git flow release start <version>
#   → package.json version bump
#   → CHANGELOG.md 갱신 (Keep a Changelog 포맷)
#   → chore(release): <version> 커밋

# (선택) 릴리즈 브랜치에서 추가 수정 및 일반 커밋

# 2. 릴리즈 마무리
/release-finish
#   → main/develop pull --ff-only
#   → git flow release finish -f <changelog-section>
#     - main 머지 (no-ff)
#     - v<version> 태그 (메시지: CHANGELOG 해당 섹션)
#     - develop 머지 (no-ff)
#     - release 브랜치 삭제
#   → push main / develop / 태그
```

릴리즈 도중 충돌이 발생하면 **자동 해결하지 않고**, `git merge --abort` + 모든 브랜치/태그를 시작 전 SHA 로 reset 한 뒤 사용자에게 보고합니다.

---

## 안전 규칙 (모든 커맨드 공통)

1. **`--no-verify` 금지** — pre-commit hook 실패 시 그대로 보고하고 중단.
2. **강제 푸시 금지** — `--force`, `--force-with-lease` 자동 수행 안 함. 필요해 보이면 사용자에게 결정 위임.
3. **머지 기준 브랜치 최신화는 `--ff-only` 만** — non-ff 발생 시 즉시 중단.
4. **충돌 자동 해결 금지** — `rebase --abort` / `merge --abort` 로 원상복구.
5. **한 커밋 = 하나의 논리적 변경** — Conventional Commits + Feature ID + Co-Authored-By 트레일러.
6. **`git add -A` / `.` 금지** — 파일을 명시적으로 지정해 스테이징.

---

## 커밋 메시지 포맷

```
<type>[(<scope>)]: <feature-id> <subject>

<body — 왜 변경했는지에 초점>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

type: `feat` / `fix` / `refactor` / `docs` / `test` / `chore` / `style` / `perf` / `ci`
BREAKING CHANGE 는 type 뒤에 `!` 또는 body 에 `BREAKING CHANGE:` 표기.

---

## SemVer 산정 규칙

`/release-start` 가 develop 의 신규 커밋을 분석해서:

| 발견된 커밋 | bump |
|---|---|
| `feat!` 또는 `BREAKING CHANGE:` | MAJOR |
| `feat` 하나라도 포함 | MINOR |
| `fix` / `perf` / `refactor` 만 | PATCH |
| 문서/내부 정리만 | PATCH (확인 후) |

추천 버전을 첫 번째 옵션으로 제시하고 사용자 확정을 받습니다.

---

## CHANGELOG.md 포맷

[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) 1.1.0 + [SemVer 2.0](https://semver.org/spec/v2.0.0.html).

카테고리 매핑:

| Conventional type | 카테고리 |
|---|---|
| `feat` | Added |
| `feat!` / `BREAKING` | Changed (`**BREAKING**` 라벨) |
| `fix` | Fixed |
| `perf` | Changed |
| `refactor` | Internal |
| `docs` / `test` / `chore` / `style` / `ci` | Internal (요약) |
| `revert` | Removed |
| deprecate 키워드 | Deprecated |
| security 라벨 | Security |

빈 카테고리는 출력하지 않습니다. `Internal` 항목이 5개를 넘으면 한 줄로 묶어 요약합니다.

---

## 향후 작업

- **Monday.com 보드 업데이트는 후속 작업**입니다. `/release-finish` 종료 시 보드 항목 상태 전환 / 릴리즈 노트 게시 기능은 현재 placeholder (`# TODO`) 로만 들어 있으며, Monday MCP 연동을 명시적으로 요청하시면 별도 작업으로 구현 예정입니다.
- 자동 changelog 그루핑(예: `BREAKING` 정렬)에 대한 사용자 정의 룰.
- pre-commit hook 실패 시 표준 안내 메시지 템플릿화.

---

## 디렉터리 구조

```
gitflow-sph/
├── .claude-plugin/
│   ├── marketplace.json
│   └── plugin.json
├── commands/
│   ├── feature-start.md
│   ├── feature-commit.md
│   ├── feature-finish.md
│   ├── release-start.md
│   └── release-finish.md
└── README.md
```

---

## 라이선스

Internal use (SPH). 자세한 사용 범위는 팀 정책을 따릅니다.
