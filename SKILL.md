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
| `session_start` | string | When the session began (ISO 8601, UTC, estimated) |
| `duration_min` | number | Approximate session duration in minutes |
| `tokens_in` | number | Input tokens used |
| `tokens_out` | number | Output tokens generated |
| `commits` | string[] | Git commit hashes made during session |
| `files` | object | File operations: `read`, `created`, `modified` arrays |
| `user_notes` | string | User-provided notes |

### Data Accuracy

- **Precise**: `timestamp`, `model`, `tool`, `commits`, `files`, `summary`
- **Estimated**: `session_start`, `duration_min`, `tokens_in`/`tokens_out` — agents typically lack direct access to their own session metadata

## /log Workflow

When user invokes `/log`:

1. **Check for session file**: Read `.llm/.session` if it exists:
   - Use `start` as `session_start` (precise, user-provided)
   - Use `topic` as a seed for writing the `summary`
   - Calculate `duration_min` from `start` to now (precise when session file exists)

2. **If no notes provided**: Ask "Add any notes for this log entry? (press Enter to skip)"

3. **Gather session data**:
   - Run `git log --oneline -10` to find commits made this session
   - If session file exists with `start`, use `git log --oneline --since="<start>"` for more accurate commit filtering
   - Track files read, created, modified during conversation
   - Note token usage if available

4. **Append entry** to `.llm/log.jsonl` in project root:
   - Create `.llm/` directory if missing: `mkdir -p .llm`
   - **Use Bash with `>>` to append** — do NOT use Write or Edit tools (they overwrite the file):
     ```bash
     printf '%s\n' '{"timestamp":"...","project_name":"..."}' >> .llm/log.jsonl
     ```
   - Never modify existing entries

5. **Confirm**: "Logged session to .llm/log.jsonl"

## Implementation Notes

- **JSONL format**: One JSON object per line, no trailing commas, no array wrapper
- **Append only**: Never modify existing log entries
- **Always use Bash `>>`**: LLM tools' Write and Edit commands overwrite file contents. Always use `printf '%s\n' '{...}' >> .llm/log.jsonl` via Bash to append
- **Create if missing**: `mkdir -p .llm` before appending if the directory doesn't exist
- **User notes from arguments**: If user typed `/log some notes`, use "some notes" as `user_notes`
- **Session file**: If `.llm/.session` exists, `session_start` and `duration_min` become precise rather than estimated
- **No cleanup**: `/log` does not delete `.llm/.session` — it persists until the next `/log-start` overwrites it. This allows multiple `/log` calls per session.

## Arguments

$ARGUMENTS