# Claude Code Statusline - context-bar.sh

## 개요

Claude Code 터미널 하단에 표시되는 커스텀 상태줄(statusline) 스크립트.
모델 정보, Git 상태, 컨텍스트 사용량, Rate limit 등을 한 줄로 보여준다.

## 설정 방법

### settings.json 위치
`~/.claude/settings.json`

### 설정 내용
```json
{
  "statusLine": {
    "type": "command",
    "command": "bash /Users/[user]/.claude/scripts/context-bar.sh"
  }
}
```

---

## 표시 형식

### 1번째 줄 (메인)
```
모델명 | 📁폴더명 | 🔀브랜치 (파일상태, 동기화상태) | ▓▓▓░░░░░░░ 12% of 200k tokens | 5h ██░░░ 35% 7d ▄░░░░ 8%
```

| 섹션 | 설명 |
|------|------|
| 모델명 | `model.display_name` 또는 `model.id` |
| 📁폴더명 | 현재 작업 디렉토리 basename |
| 🔀브랜치 | Git 현재 브랜치 + 커밋 안 된 파일 수 + upstream 동기화 상태 |
| 컨텍스트 바 | 10칸 시각 바 (`█▄░`) + 사용률 % + 최대 토큰 수 |
| Rate limit | 5시간/7일 Claude.ai 구독 사용률을 5칸 바로 표시 (있을 때만) |

### 2번째 줄
```
💬 마지막 사용자 메시지 (1번째 줄 너비만큼 잘림)
```

---

## 기능 상세

### 색상 테마

스크립트 상단의 `COLOR` 변수로 변경:

```bash
COLOR="blue"  # 기본값
```

| 테마 | ANSI 코드 |
|------|-----------|
| `orange` | `\033[38;5;173m` |
| `blue` | `\033[38;5;74m` |
| `teal` | `\033[38;5;66m` |
| `green` | `\033[38;5;71m` |
| `lavender` | `\033[38;5;139m` |
| `rose` | `\033[38;5;132m` |
| `gold` | `\033[38;5;136m` |
| `slate` | `\033[38;5;60m` |
| `cyan` | `\033[38;5;37m` |
| 기타 | gray (기본 텍스트 색과 동일) |

### Git 상태 표시

- **파일 0개**: `(0 files uncommitted, synced 5m ago)`
- **파일 1개**: `(src/index.ts uncommitted, 2 ahead)` — 파일명 직접 표시
- **파일 N개**: `(3 files uncommitted, 1 behind)`
- **upstream 없음**: `(2 files uncommitted, no upstream)`

동기화 상태:
| 상태 | 표시 |
|------|------|
| 동기화됨 | `synced` 또는 `synced 5m ago` |
| 로컬이 앞섬 | `N ahead` |
| 리모트가 앞섬 | `N behind` |
| 양쪽 다름 | `N ahead, M behind` |
| upstream 미설정 | `no upstream` |

### 컨텍스트 사용량 바

transcript 파일에서 실제 토큰 사용량을 계산:

```
█ = 80% 이상 채워진 칸
▄ = 30~79% 채워진 칸
░ = 30% 미만 (빈 칸)
```

- **대화 시작 시**: 시스템 프롬프트/도구/메모리 등 ~20k 베이스라인 추정치 표시 (`~10%`)
- **대화 진행 중**: 마지막 응답의 `input_tokens + cache_read + cache_creation` 합산
- **바 너비**: 10칸 (칸당 10%p)

### Rate Limit 바

Claude.ai 구독 사용률을 5칸 바로 표시:

```
5h ██░░░ 35%   — 5시간 윈도우 사용률
7d ▄░░░░ 8%    — 7일 윈도우 사용률
```

- **바 너비**: 5칸 (칸당 20%p)
- 데이터가 없으면 해당 섹션 미표시
- `make_bar` 함수로 컨텍스트 바와 동일한 로직 공유

### make_bar 함수

바 너비를 가변적으로 받아 재사용 가능한 바 생성 함수:

```bash
make_bar <pct> <width>
# 예: make_bar 35 5  →  ██░░░
# 예: make_bar 35 10 →  ███░░░░░░░
```

각 칸은 `100/width`% 범위를 담당하며 채워진 정도에 따라 `█`, `▄`, `░` 중 하나를 출력한다.

### 마지막 사용자 메시지

- transcript에서 마지막 `user` 타입 메시지의 텍스트만 추출
- 줄바꿈 → 공백으로 치환
- `[Request interrupted`, `[Request cancelled` 등 불필요한 메시지 스킵
- 1번째 줄 너비를 초과하면 `...`으로 잘림
- 줄 너비 계산 시 ANSI 색상 코드를 제외한 plain text 기준으로 계산 (`rate_limits_plain` 사용)

---

## 입력 데이터

Claude Code가 stdin으로 JSON을 전달한다. 주요 필드:

```json
{
  "model": { "display_name": "Claude Sonnet 4", "id": "claude-sonnet-4-..." },
  "cwd": "/path/to/project",
  "transcript_path": "/tmp/claude-transcript-xxx.jsonl",
  "context_window": { "context_window_size": 200000 },
  "rate_limits": {
    "five_hour": { "used_percentage": 35.5 },
    "seven_day": { "used_percentage": 8.2 }
  }
}
```

---

## 스크립트 전체 소스

```bash
#!/bin/bash

# Color theme: gray, orange, blue, teal, green, lavender, rose, gold, slate, cyan
# Preview colors with: bash scripts/color-preview.sh
COLOR="blue"

# Color codes
C_RESET='\033[0m'
C_GRAY='\033[38;5;245m' # explicit gray for default text
C_BAR_EMPTY='\033[38;5;238m'
case "$COLOR" in
 orange) C_ACCENT='\033[38;5;173m' ;;
 blue) C_ACCENT='\033[38;5;74m' ;;
 teal) C_ACCENT='\033[38;5;66m' ;;
 green) C_ACCENT='\033[38;5;71m' ;;
 lavender) C_ACCENT='\033[38;5;139m' ;;
 rose) C_ACCENT='\033[38;5;132m' ;;
 gold) C_ACCENT='\033[38;5;136m' ;;
 slate) C_ACCENT='\033[38;5;60m' ;;
 cyan) C_ACCENT='\033[38;5;37m' ;;
 *) C_ACCENT="$C_GRAY" ;; # gray: all same color
esac

input=$(cat)

# Extract model, directory, and cwd
model=$(echo "$input" | jq -r '.model.display_name // .model.id // "?"')
cwd=$(echo "$input" | jq -r '.cwd // empty')
dir=$(basename "$cwd" 2>/dev/null || echo "?")

# Get git branch, uncommitted file count, and sync status
branch=""
git_status=""
if [[ -n "$cwd" && -d "$cwd" ]]; then
 branch=$(git -C "$cwd" branch --show-current 2>/dev/null)
 if [[ -n "$branch" ]]; then
  # Count uncommitted files
  file_count=$(git -C "$cwd" --no-optional-locks status --porcelain -uall 2>/dev/null | wc -l | tr -d ' ')

  # Check sync status with upstream
  sync_status=""
  upstream=$(git -C "$cwd" rev-parse --abbrev-ref @{upstream} 2>/dev/null)
  if [[ -n "$upstream" ]]; then
   # Get last fetch time
   fetch_head="$cwd/.git/FETCH_HEAD"
   fetch_ago=""
   if [[ -f "$fetch_head" ]]; then
    fetch_time=$(stat -f %m "$fetch_head" 2>/dev/null || stat -c %Y "$fetch_head" 2>/dev/null)
    if [[ -n "$fetch_time" ]]; then
     now=$(date +%s)
     diff=$((now - fetch_time))
     if [[ $diff -lt 60 ]]; then
      fetch_ago="<1m ago"
     elif [[ $diff -lt 3600 ]]; then
      fetch_ago="$((diff / 60))m ago"
     elif [[ $diff -lt 86400 ]]; then
      fetch_ago="$((diff / 3600))h ago"
     else
      fetch_ago="$((diff / 86400))d ago"
     fi
    fi
   fi

   counts=$(git -C "$cwd" rev-list --left-right --count HEAD...@{upstream} 2>/dev/null)
   ahead=$(echo "$counts" | cut -f1)
   behind=$(echo "$counts" | cut -f2)
   if [[ "$ahead" -eq 0 && "$behind" -eq 0 ]]; then
    if [[ -n "$fetch_ago" ]]; then
     sync_status="synced ${fetch_ago}"
    else
     sync_status="synced"
    fi
   elif [[ "$ahead" -gt 0 && "$behind" -eq 0 ]]; then
    sync_status="${ahead} ahead"
   elif [[ "$ahead" -eq 0 && "$behind" -gt 0 ]]; then
    sync_status="${behind} behind"
   else
    sync_status="${ahead} ahead, ${behind} behind"
   fi
  else
   sync_status="no upstream"
  fi

  # Build git status string
  if [[ "$file_count" -eq 0 ]]; then
   git_status="(0 files uncommitted, ${sync_status})"
  elif [[ "$file_count" -eq 1 ]]; then
   # Show the actual filename when only one file is uncommitted
   single_file=$(git -C "$cwd" --no-optional-locks status --porcelain -uall 2>/dev/null | head -1 | sed 's/^...//')
   git_status="(${single_file} uncommitted, ${sync_status})"
  else
   git_status="(${file_count} files uncommitted, ${sync_status})"
  fi
 fi
fi

# Get transcript path for context calculation and last message feature
transcript_path=$(echo "$input" | jq -r '.transcript_path // empty')

# Get context window size from JSON (accurate), but calculate tokens from transcript
# (more accurate than total_input_tokens which excludes system prompt/tools/memory)
# See: github.com/anthropics/claude-code/issues/13652
max_context=$(echo "$input" | jq -r '.context_window.context_window_size // 200000')
max_k=$((max_context / 1000))
if [[ $max_k -ge 1000 ]]; then
 max_display="$((max_k / 1000))M"
else
 max_display="${max_k}k"
fi

# Build a colored bar: make_bar <pct> <width>
make_bar() {
 local pct=$1 width=$2
 local bar=""
 for ((i=0; i<width; i++)); do
  local bar_start=$((i * 100 / width))
  local progress=$((pct - bar_start))
  local cell_size=$((100 / width))
  if [[ $progress -ge $((cell_size * 8 / 10)) ]]; then
   bar+="${C_ACCENT}█${C_RESET}"
  elif [[ $progress -ge $((cell_size * 3 / 10)) ]]; then
   bar+="${C_ACCENT}▄${C_RESET}"
  else
   bar+="${C_BAR_EMPTY}░${C_RESET}"
  fi
 done
 echo -n "$bar"
}

# Calculate context bar from transcript
if [[ -n "$transcript_path" && -f "$transcript_path" ]]; then
 context_length=$(jq -s '
 map(select(.message.usage and .isSidechain != true and .isApiErrorMessage != true)) |
 last |
 if . then
 (.message.usage.input_tokens // 0) +
 (.message.usage.cache_read_input_tokens // 0) +
 (.message.usage.cache_creation_input_tokens // 0)
 else 0 end
 ' < "$transcript_path")

 # 20k baseline: includes system prompt (~3k), tools (~15k), memory (~300),
 # plus ~2k for git status, env block, XML framing, and other dynamic context
 baseline=20000

 if [[ "$context_length" -gt 0 ]]; then
  pct=$((context_length * 100 / max_context))
  pct_prefix=""
 else
  # At conversation start, ~20k baseline is already loaded
  pct=$((baseline * 100 / max_context))
  pct_prefix="~"
 fi

 [[ $pct -gt 100 ]] && pct=100
 ctx="$(make_bar $pct 10) ${C_GRAY}${pct_prefix}${pct}% of ${max_display} tokens"
else
 # Transcript not available yet - show baseline estimate
 baseline=20000
 pct=$((baseline * 100 / max_context))
 [[ $pct -gt 100 ]] && pct=100
 ctx="$(make_bar $pct 10) ${C_GRAY}~${pct}% of ${max_display} tokens"
fi

# Extract rate limits as bars
rate_limits=""
rate_limits_plain=""
five_pct_raw=$(echo "$input" | jq -r '.rate_limits.five_hour.used_percentage // empty')
seven_pct_raw=$(echo "$input" | jq -r '.rate_limits.seven_day.used_percentage // empty')
if [[ -n "$five_pct_raw" ]]; then
 five_pct=$(printf '%.0f' "$five_pct_raw")
 [[ $five_pct -gt 100 ]] && five_pct=100
 rate_limits="${C_GRAY}5h $(make_bar $five_pct 5) ${five_pct}%"
 rate_limits_plain="5h xxxxx ${five_pct}%"
fi
if [[ -n "$seven_pct_raw" ]]; then
 seven_pct=$(printf '%.0f' "$seven_pct_raw")
 [[ $seven_pct -gt 100 ]] && seven_pct=100
 rate_limits+="${rate_limits:+ }${C_GRAY}7d $(make_bar $seven_pct 5) ${seven_pct}%"
 rate_limits_plain+="${rate_limits_plain:+ }7d xxxxx ${seven_pct}%"
fi

# Build output: Model | Dir | Branch (uncommitted) | Context
output="${C_ACCENT}${model}${C_GRAY} | 📁${dir}"
[[ -n "$branch" ]] && output+=" | 🔀${branch} ${git_status}"
output+=" | ${ctx}"
[[ -n "$rate_limits" ]] && output+=" | ${rate_limits}"
output+="${C_RESET}"

printf '%b\n' "$output"

# Get user's last message (text only, not tool results, skip unhelpful messages)
if [[ -n "$transcript_path" && -f "$transcript_path" ]]; then
 # Calculate visible length (without ANSI codes) using plain text equivalents
 plain_output="${model} | 📁${dir}"
 [[ -n "$branch" ]] && plain_output+=" | 🔀${branch} ${git_status}"
 plain_output+=" | xxxxxxxxxx ${pct}% of ${max_display} tokens"
 [[ -n "$rate_limits_plain" ]] && plain_output+=" | ${rate_limits_plain}"
 max_len=${#plain_output}
 last_user_msg=$(jq -rs '
 # Messages to skip (not useful as context)
 def is_unhelpful:
 startswith("[Request interrupted") or
 startswith("[Request cancelled") or
 . == "";

 [.[] | select(.type == "user") |
 select(.message.content | type == "string" or
 (type == "array" and any(.[]; .type == "text")))] |
 reverse |
 map(.message.content |
 if type == "string" then .
 else [.[] | select(.type == "text") | .text] | join(" ") end |
 gsub("\n"; " ") | gsub(" +"; " ")) |
 map(select(is_unhelpful | not)) |
 first // ""
 ' < "$transcript_path" 2>/dev/null)

 if [[ -n "$last_user_msg" ]]; then
  if [[ ${#last_user_msg} -gt $max_len ]]; then
   echo "💬 ${last_user_msg:0:$((max_len - 3))}..."
  else
   echo "💬 ${last_user_msg}"
  fi
 fi
fi
```

---

## 커스터마이징

### 색상 변경
`context-bar.sh` 상단의 `COLOR="blue"` 값을 원하는 테마로 변경.

### 섹션 제거
`output` 조합 부분에서 불필요한 라인을 주석 처리.

### 바 너비 변경
`make_bar` 호출 시 두 번째 인자로 원하는 칸 수 지정:
- 컨텍스트 바: `make_bar $pct 10` (기본 10칸)
- Rate limit 바: `make_bar $pct 5` (기본 5칸)

### 2번째 줄 (마지막 메시지) 비활성화
스크립트 하단의 `# Get user's last message` 블록 전체를 주석 처리.
