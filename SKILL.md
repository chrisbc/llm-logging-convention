---
name: log
description: Log this LLM session to .llm/log.jsonl. ONLY use when user explicitly asks to log the session with /log.
argument-hint: [notes]
---

# LLM Session Logging

Append a log entry to `.llm/log.jsonl` at the end of each session.

## Log Schema

```json
{
  "timestamp": "2025-02-22T10:30:00Z",
  "session_start": "2025-02-22T09:15:00Z",
  "project_name": "Human-readable name",
  "project_root": "folder-name",
  "model": "claude-3-5-sonnet",
  "tool": "claude-code | opencode | cursor | other",
  "duration_min": 25,
  "tokens_in": 15000,
  "tokens_out": 8000,
  "commits": ["e263c5e", "abc1234"],
  "files": {
    "read": ["PLAN.md"],
    "created": ["README.md"],
    "modified": ["src/index.ts"]
  },
  "summary": "Brief description of work done",
  "user_notes": "Optional notes from user"
}
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | string | When the log entry was created (ISO 8601, UTC) |
| `project_name` | string | Human-readable project name |
| `project_root` | string | Root folder name (for aggregation) |
| `model` | string | Model identifier |
| `tool` | string | Tool name (claude-code, opencode, cursor, etc.) |
| `summary` | string | Brief description of session work |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `session_start` | string | When this work segment began (ISO 8601, UTC) |
| `duration_min` | number | Duration of this work segment in minutes |
| `tokens_in` | number | Input tokens used |
| `tokens_out` | number | Output tokens generated |
| `commits` | string[] | Git commit hashes made during this segment |
| `files` | object | File operations: `read`, `created`, `modified` arrays |
| `user_notes` | string | User-provided notes |

### Data Accuracy

- **Always precise**: `timestamp` (via `date -u`), `model`, `tool`, `commits`, `files`, `summary`
- **Precise when valid session file**: `session_start`, `duration_min`
- **Estimated**: `tokens_in`/`tokens_out` — agents typically lack direct access to their own session metadata

## /log Workflow

When user invokes `/log`:

1. **Get current timestamp** using bash:
   ```bash
   date -u +"%Y-%m-%dT%H:%M:%SZ"
   ```

2. **Check for session file** (`.llm/.session`):

   **If file does NOT exist:**
   - Ask user: "No session file found. Provide a session_start time? (YYYY-MM-DDTHH:MM:SSZ or 'skip')"
   - If user provides time: use as `session_start`, create `.session` file
   - If user skips: leave `session_start` and `duration_min` undefined

   **If file EXISTS:**
   - Read `start` and `topic` from `.llm/.session`

3. **Calculate duration** (if valid `session_start`):
   ```bash
   start="2025-02-27T09:52:52Z"  # from .llm/.session
   now=$(date -u +%s)
   start_epoch=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$start" +%s 2>/dev/null || date -d "$start" +%s 2>/dev/null)
   echo $(( (now - start_epoch) / 60 ))
   ```

4. **Threshold check** — if `duration_min > 120`:
   - Prompt: "Session shows **X hours** duration. Adjust start time? [y/no/abort]"
   - `y`: Ask "Enter new start (YYYY-MM-DDTHH:MM:SSZ or 'now'):"
     - `now` → use current timestamp, `duration_min` = 0
     - timestamp → validate and use, recalculate duration
   - `no`: Proceed with current duration (legitimate long session)
   - `abort`: Cancel `/log` command

5. **Find commits for this segment**:
   ```bash
   # If session_start is known:
   git log --pretty=format:"%h" --since="<session_start>"
   
   # If no session_start, use last log timestamp:
   git log --pretty=format:"%h" --since="<last_log_timestamp>"
   ```
   - Remind user: "Found X commits since session start"

6. **If no notes provided in arguments**: Ask "Add any notes for this log entry? (press Enter to skip)"

7. **Gather files data** — track files read, created, modified during conversation

8. **Append entry** to `.llm/log.jsonl`:
   ```bash
   mkdir -p .llm && printf '%s\n' '{"timestamp":"...","session_start":"..."}' >> .llm/log.jsonl
   ```
   - One JSON object per line
   - No trailing commas, no array wrapper

9. **Update `.session.start`** to the log entry timestamp:
   ```bash
   # After successful log append, update .session for next segment
   echo "{\"start\":\"$timestamp\",\"topic\":\"$topic\"}" > .llm/.session
   ```
   - This makes the next `/log` calculate duration since this log

10. **Confirm**: "Logged session to .llm/log.jsonl"

## Implementation Notes

- **JSONL format**: One JSON object per line, no trailing commas, no array wrapper
- **Append only**: Never modify existing log entries
- **Always use Bash `>>`**: LLM tools' Write and Edit commands overwrite file contents. Always use `printf '%s\n' '{...}' >> .llm/log.jsonl` via Bash to append
- **Create if missing**: `mkdir -p .llm` before appending if the directory doesn't exist
- **User notes from arguments**: If user typed `/log some notes`, use "some notes" as `user_notes`
- **Threshold for duration**: 120 minutes — prompts user to adjust if exceeded (handles pause/resume)
- **Update .session after log**: Each log entry represents a work segment; `.session.start` updated to track next segment
- **No cleanup**: `/log` does not delete `.llm/.session` — it persists and gets updated

## Arguments

$ARGUMENTS