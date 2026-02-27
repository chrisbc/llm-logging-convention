---
name: log-start
description: Mark the start of a work session. ONLY use when user explicitly invokes /log-start.
argument-hint: [topic]
---

# Start LLM Session

Record the start time and optional topic for the current work session. This data is used later by `/log` to produce an accurate log entry.

## Workflow

When user invokes `/log-start` or `/log-start <topic>`:

1. **Check for existing session** — if `.llm/.session` exists:
   - Read the file to get `start` time
   - Check for unlogged commits since that start:
     ```bash
     git log --oneline --since="<start>"
     ```
   - If commits found, prompt user:
     > "Previous session has unlogged commits. Log it first? [y/n]"
     - `y`: Run `/log` workflow, then continue with step 2
     - `n`: Warn "Previous session will be lost", continue to step 2

2. **Create directory** if needed:
   ```bash
   mkdir -p .llm
   ```

3. **Record session start** — write `.llm/.session` with live UTC timestamp:
   ```bash
   echo "{\"start\":\"$(date -u +"%Y-%m-%dT%H:%M:%SZ")\",\"topic\":\"<topic>\"}" > .llm/.session
   ```
   - If no topic provided, omit the `topic` field

4. **Confirm**:
   - With topic: "Session started: *<topic>*"
   - Without topic: "Session started."

## Implementation Notes

- **Use bash**: Always write via `echo ... > .llm/.session` to get accurate UTC timestamp
- **Pre-overwrite check**: Warn user if previous session has unlogged work
- **Transient file**: `.llm/.session` should be in `.gitignore`
- The `/log` skill reads this file when creating the log entry

## Arguments

$ARGUMENTS