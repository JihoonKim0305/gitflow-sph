---
allowed-tools: Bash(mkdir:*), Bash(rm:*), Bash(printf:*), Bash(test:*), Bash(wc:*), Bash(cut:*), Bash(tr:*)
description: Monday.com API 토큰을 저장한다 (/release-finish, /monday-init 에서 사용)
argument-hint: "<TOKEN> | --clear"
---

## 공통 규칙 (반드시 준수)

- **토큰 전체를 stdout/stderr/로그/메모리덤프 어디에도 출력 금지.** 마스킹된 형태만 노출.
- `set -x` 또는 디버그 트레이스 사용 금지.
- 토큰을 변수에 담은 후에는 즉시 파일에 기록하고, 화면 출력 시에는 반드시 마스킹.

## 컨텍스트 수집

```bash
TOKEN_FILE="$HOME/.claude/.gitflow-sph-monday.token"
test -f "$TOKEN_FILE" && echo "기존 토큰 파일 존재 (덮어쓰기 예정)" || true
```

## 실행 절차

### 1) 인자 파싱

`$1` 처리:

- `--clear` → 토큰 파일 삭제 후 종료.
- 비어 있음 → 사용법 안내 후 종료. 절대 인자 없이 토큰을 묻지 말 것 (이 명령은 토큰을 인자로 받는 단순 유틸).
- 그 외 → 토큰 값으로 처리.

### 2) `--clear` 처리

```bash
if [ "$1" = "--clear" ]; then
  if [ -f "$TOKEN_FILE" ]; then
    rm -f "$TOKEN_FILE"
    echo "✓ Monday 토큰 삭제됨 ($TOKEN_FILE)"
  else
    echo "ℹ 저장된 토큰이 없습니다."
  fi
  exit 0
fi
```

### 3) 인자 비었을 때 사용법 안내

```bash
if [ -z "$1" ]; then
  cat <<'EOF'
사용법:
  /monday-token <TOKEN>        토큰을 저장
  /monday-token --clear        저장된 토큰 삭제

저장 위치: ~/.claude/.gitflow-sph-monday.token
EOF
  exit 0
fi
```

### 4) 토큰 저장

```bash
TOKEN="$1"
mkdir -p "$HOME/.claude"
printf '%s' "$TOKEN" > "$TOKEN_FILE"
```

- `printf '%s'` 로 개행 없이 정확히 토큰만 저장.
- `mkdir -p` 로 디렉터리 없으면 생성.

### 5) 마스킹 출력

```bash
LEN=$(printf '%s' "$TOKEN" | wc -c | tr -d ' ')
HEAD4=$(printf '%s' "$TOKEN" | cut -c1-4)
echo "✓ Monday 토큰 저장됨: ${HEAD4}**** (${LEN}자, $TOKEN_FILE)"
```

## 출력 형식

저장 성공:
```
✓ Monday 토큰 저장됨: abcd**** (40자, /home/user/.claude/.gitflow-sph-monday.token)
```

삭제 성공:
```
✓ Monday 토큰 삭제됨 (/home/user/.claude/.gitflow-sph-monday.token)
```

## 금지 사항

- 토큰 전체 echo / cat / log 금지.
- 토큰을 git 추적 파일에 저장 금지.
- 토큰을 환경변수로 export 하여 부모 셸에 노출 금지 (이 명령은 파일 저장만 한다).
- `set -x`, `bash -x` 등 디버그 트레이스 금지.
