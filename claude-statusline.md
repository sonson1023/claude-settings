# Claude Code Statusline Script - Full Content

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

# Format remaining time until rate limit reset
format_remaining() {
 local reset_at="$1"
 [[ -z "$reset_at" ]] && return
 local now
 now=$(date +%s)
 # reset_at may be in milliseconds (>1e12) or seconds
 if [[ ${#reset_at} -ge 13 ]]; then
  reset_at=$((reset_at / 1000))
 fi
 local remaining=$((reset_at - now))
 [[ $remaining -le 0 ]] && echo "resetting" && return
 local hours=$((remaining / 3600))
 local mins=$(( (remaining % 3600) / 60 ))
 if [[ $hours -gt 0 ]]; then
  echo "${hours}h${mins}m left"
 else
  echo "${mins}m left"
 fi
}

# Extract rate limits as bars
rate_limits=""
rate_limits_plain=""
five_pct_raw=$(echo "$input" | jq -r '.rate_limits.five_hour.used_percentage // empty')
five_reset_raw=$(echo "$input" | jq -r '.rate_limits.five_hour.reset_at // empty')
seven_pct_raw=$(echo "$input" | jq -r '.rate_limits.seven_day.used_percentage // empty')
seven_reset_raw=$(echo "$input" | jq -r '.rate_limits.seven_day.reset_at // empty')
if [[ -n "$five_pct_raw" ]]; then
 five_pct=$(printf '%.0f' "$five_pct_raw")
 [[ $five_pct -gt 100 ]] && five_pct=100
 five_remaining=$(format_remaining "$five_reset_raw")
 if [[ -n "$five_remaining" ]]; then
  rate_limits="${C_GRAY}5h $(make_bar $five_pct 5) ${five_pct}% (${five_remaining})"
  rate_limits_plain="5h xxxxx ${five_pct}% (${five_remaining})"
 else
  rate_limits="${C_GRAY}5h $(make_bar $five_pct 5) ${five_pct}%"
  rate_limits_plain="5h xxxxx ${five_pct}%"
 fi
fi
if [[ -n "$seven_pct_raw" ]]; then
 seven_pct=$(printf '%.0f' "$seven_pct_raw")
 [[ $seven_pct -gt 100 ]] && seven_pct=100
 seven_remaining=$(format_remaining "$seven_reset_raw")
 if [[ -n "$seven_remaining" ]]; then
  rate_limits+="${rate_limits:+ }${C_GRAY}7d $(make_bar $seven_pct 5) ${seven_pct}% (${seven_remaining})"
  rate_limits_plain+="${rate_limits_plain:+ }7d xxxxx ${seven_pct}% (${seven_remaining})"
 else
  rate_limits+="${rate_limits:+ }${C_GRAY}7d $(make_bar $seven_pct 5) ${seven_pct}%"
  rate_limits_plain+="${rate_limits_plain:+ }7d xxxxx ${seven_pct}%"
 fi
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
