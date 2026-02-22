---
name: log-start
description: Mark the start of a work session. ONLY use when user explicitly invokes /log-start.
argument-hint: [topic]
---

# Start LLM Session

Record the start time and optional topic for the current work session. This data is used later by `/log` to produce an accurate log entry.

## Workflow

When user invokes `/log-start` or `/log-start <topic>`:

1. **Record session start** — write a JSON file to `.llm/.session`:
   ```json
   {
     "start": "2025-02-22T09:15:00Z",
     "topic": "optional topic from user"
   }
   ```
   - `start`: current time in ISO 8601 UTC
   - `topic`: the argument text if provided, otherwise omit

2. **Create directory** if `.llm/` doesn't exist

3. **Confirm**:
   - With topic: "Session started: *<topic>*"
   - Without topic: "Session started."

## Implementation Notes

- **Overwrite** `.llm/.session` each time — only one active session at a time
- **Minimal I/O**: just one small file write, no other processing needed
- `.llm/.session` is a transient file — it should be in `.gitignore`
- The `/log` skill reads this file when creating the log entry

## Arguments

$ARGUMENTS
