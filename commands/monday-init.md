---
allowed-tools: Bash(test:*), Bash(cat:*), Bash(curl:*), Bash(jq:*), Bash(printf:*), Bash(tr:*), Bash(mv:*), Bash(mktemp:*)
description: Monday.com Release 보드 설정 파일(monday.config.json)을 프로젝트 루트에 생성한다
argument-hint: "[board_id] (선택 — 없으면 대화형으로 입력)"
---

## 공통 규칙 (반드시 준수)

- 토큰 전체를 어떤 출력에도 노출 금지. 앞 4자 + 길이 마스킹만 허용.
- 보드 컬럼 목록 조회 실패 시 사용자에게 명확히 실패 사유 표시 후 placeholder 모드로 fallback.
- 기존 `monday.config.json` 이 존재하면 반드시 덮어쓰기 확인 단계를 거침.

## 컨텍스트 수집 (병렬)

```bash
test -f monday.config.json && echo "기존 monday.config.json 존재"
TOKEN_FILE="$HOME/.claude/.gitflow-sph-monday.token"
MONDAY_TOKEN="${MONDAY_API_TOKEN:-}"
if [ -z "$MONDAY_TOKEN" ] && [ -f "$TOKEN_FILE" ]; then
  MONDAY_TOKEN=$(tr -d '\r\n' < "$TOKEN_FILE")
fi
[ -n "$MONDAY_TOKEN" ] && echo "토큰 로드됨 (자동 매핑 가능)" || echo "토큰 없음 (placeholder 모드)"
command -v curl >/dev/null && command -v jq >/dev/null && echo "curl/jq OK"
```

## 실행 절차

### 1) 기존 파일 확인

`monday.config.json` 이 이미 존재하면 `AskUserQuestion` 으로 덮어쓰기 여부 확인:

- 옵션 1: **기존 유지** (Recommended) — 종료.
- 옵션 2: **덮어쓰기** — 진행.

### 2) board_id 확보

- `$1` 가 있으면 그것을 `BOARD_ID` 로 사용.
- 없으면 `AskUserQuestion` 으로 "Monday 보드 URL 의 `/boards/<숫자>` 부분" 안내와 함께 입력 받음.
- 숫자만 허용 (`^[0-9]+$`). 아니면 거부 후 재입력 요청.

### 3) 컬럼 자동 매핑 (토큰 있을 때만)

토큰이 없거나 `curl`/`jq` 가 없으면 **4단계(placeholder 모드)** 로 건너뜀.

GraphQL 로 보드 컬럼 조회:

```bash
QUERY=$(jq -nc --arg id "$BOARD_ID" \
  '{query:"query($id:ID!){boards(ids:[$id]){columns{id title type}}}",variables:{id:$id}}')

COLUMNS_JSON=$(curl -sS -X POST https://api.monday.com/v2 \
  -H "Authorization: $MONDAY_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$QUERY")
```

응답 검증:

```bash
if echo "$COLUMNS_JSON" | jq -e '.errors' >/dev/null 2>&1; then
  echo "✗ Monday API 에러:"
  echo "$COLUMNS_JSON" | jq '.errors'
  echo "ℹ placeholder 모드로 진행합니다 (수동 편집 필요)."
  # → 4단계로
fi
```

컬럼 추출:

```bash
COLUMNS=$(echo "$COLUMNS_JSON" | jq -c '.data.boards[0].columns // []')
COUNT=$(echo "$COLUMNS" | jq 'length')
[ "$COUNT" -eq 0 ] && { echo "✗ 보드에서 컬럼을 찾을 수 없습니다 (board_id 확인)."; exit 1; }
```

4개의 필수 필드 각각에 대해 `AskUserQuestion` 으로 컬럼 선택. 추천 우선순위 (첫 번째 옵션에 `(Recommended)` 표기):

| 필드 | 추천 우선순위 |
|---|---|
| `version` | type=`text` 중 title 에 "version" 포함 → type=`text` 중 첫 번째 |
| `date` | type=`date` 중 첫 번째 → type=`date` 다음 |
| `feature_ids` | type=`long_text` → type=`text` 중 title 에 "feature" 포함 |
| `changelog` | type=`long_text` 중 title 에 "changelog/release/note" 포함 → type=`long_text` 첫 번째 |

각 옵션 label: `<title> (<type>, id=<id>)`. 최대 4개 옵션. 사용자가 "Other" 로 직접 ID 입력 가능.

선택된 컬럼 ID 를 변수에 저장:

```bash
COL_VERSION="..."
COL_DATE="..."
COL_FIDS="..."
COL_CHANGELOG="..."
```

### 4) placeholder 모드 fallback

토큰 없음 / API 실패 시:

```bash
COL_VERSION="REPLACE_WITH_COLUMN_ID"
COL_DATE="REPLACE_WITH_COLUMN_ID"
COL_FIDS="REPLACE_WITH_COLUMN_ID"
COL_CHANGELOG="REPLACE_WITH_COLUMN_ID"
PLACEHOLDER_MODE=1
```

### 5) 파일 작성

```bash
jq -n \
  --arg b "$BOARD_ID" \
  --arg cv "$COL_VERSION" \
  --arg cd "$COL_DATE" \
  --arg cf "$COL_FIDS" \
  --arg cc "$COL_CHANGELOG" \
  '{
    release_board_id: $b,
    release_item_name_template: "v{version}",
    columns: {
      version: $cv,
      date: $cd,
      feature_ids: $cf,
      changelog: $cc
    }
  }' > monday.config.json
```

### 6) 결과 출력

자동 매핑 모드:

```
✓ monday.config.json 생성 완료
  board:    <BOARD_ID>
  columns:  4/4 매핑됨 (version, date, feature_ids, changelog)
  next:     /release-finish 실행 시 자동 사용됩니다.
```

placeholder 모드:

```
⚠ monday.config.json 생성됨 (placeholder)
  board:    <BOARD_ID>
  columns:  0/4 매핑됨 — 수동 편집 필요
  next:     /monday-token <TOKEN> 등록 후 /monday-init <BOARD_ID> 재실행하거나
            monday.config.json 의 REPLACE_WITH_COLUMN_ID 4곳을 직접 편집하세요.
```

마지막 한 줄로 `.gitignore` 안내:

```
ℹ monday.config.json 은 토큰을 포함하지 않지만, 사내 정책에 따라 .gitignore 추가를 고려하세요.
```

## 출력 형식

위 6단계의 출력 그대로 사용.

## 금지 사항

- 토큰을 stdout/stderr/curl `-v` 등으로 노출 금지.
- 잘못된 응답을 그대로 파일에 쓰지 말 것 (반드시 jq 검증 후 작성).
- 사용자 확인 없이 기존 `monday.config.json` 덮어쓰기 금지.
- 보드 ID 검증 실패 시 임의 추정 금지.
