---
name: llm-log
description: Log this LLM session to .llm/log.jsonl. ONLY use when user explicitly asks to log the session with /log or /llm-log.
argument-hint: [notes]
---

# LLM Session Logging

Append a log entry to `.llm/log.jsonl` at the end of each session.

## Log Schema

```json
{
  "timestamp": "2025-02-22T10:30:00Z",
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
| `timestamp` | string | Session start time (ISO 8601, UTC) |
| `project_name` | string | Human-readable project name |
| `project_root` | string | Root folder name (for aggregation) |
| `model` | string | Model identifier |
| `tool` | string | Tool name (claude-code, opencode, cursor, etc.) |
| `duration_min` | number | Approximate session duration in minutes |
| `summary` | string | Brief description of session work |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `tokens_in` | number | Input tokens used |
| `tokens_out` | number | Output tokens generated |
| `commits` | string[] | Git commit hashes made during session |
| `files` | object | File operations: `read`, `created`, `modified` arrays |
| `user_notes` | string | User-provided notes |

## /log Workflow

When user invokes `/log` or `/llm-log`:

1. **If no notes provided**: Ask "Add any notes for this log entry? (press Enter to skip)"

2. **Gather session data**:
   - Run `git log --oneline -10` to find commits made this session
   - Track files read, created, modified during conversation
   - Calculate duration from first user message to now
   - Note token usage if available

3. **Append entry** to `.llm/log.jsonl` in project root:
   - Create directory if missing
   - Append one valid JSON object per line
   - Never modify existing entries

4. **Confirm**: "Logged session to .llm/log.jsonl"

## Implementation Notes

- **JSONL format**: One JSON object per line, no trailing commas, no array wrapper
- **Append only**: Never modify existing log entries
- **Create if missing**: If `.llm/log.jsonl` doesn't exist, create it with the first entry
- **User notes from arguments**: If user typed `/log some notes`, use "some notes" as `user_notes`

## Arguments

$ARGUMENTS