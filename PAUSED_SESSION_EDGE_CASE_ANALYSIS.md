# Paused Session Edge Case Analysis

## Problem Scenario

1. User runs `/log-start new2` at 09:52
2. Work happens (commits, file changes)
3. User pauses session (lunch, meeting, end of day) — no `/log` called
4. User resumes at 17:28 (7.5 hours later)
5. User runs `/log`
6. Duration incorrectly calculated as 7.5 hours instead of actual work time

## The Core Issue

- `/log-start` freezes the session start time in `.session`
- `/log` calculates `duration = now - session_start`
- If user paused without logging, duration is inflated
- This affects time statistics and makes logs misleading

## Options Considered

### Option 1: Always Confirm session_start

**Approach:** `/log` always asks user to confirm the start time.

**Pros:**
- Never wrong

**Cons:**
- Tedious for short sessions (most common case)
- Friction for 95% of sessions where timing is correct

**Verdict:** Rejected — too much friction

### Option 2: Threshold Prompt in `/log`

**Approach:** `/log` prompts only when `duration_min > X` (e.g., 120 minutes).

```
If duration_min > 120 minutes:
  "Session shows X hours duration. Adjust start time? [y/no/abort]"
  - y: "Enter new start (YYYY-MM-DDTHH:MM:SSZ or 'now'):"
  - no: proceed with current duration
  - abort: cancel /log
```

**Pros:**
- Only asks when timing looks suspicious
- Handles actual long sessions (user confirms "no, it's correct")
- Catches pause/resume scenarios automatically

**Cons:**
- Threshold (120 min) is arbitrary
- Might trigger for legitimate long sessions

**Verdict:** Recommended — best balance of safety and UX

### Option 3: Dedicated `/log-resume` Command

**Approach:** New command explicitly for resuming after pause.

```
/log-resume [topic]
```

**Pros:**
- Explicit intent — user knows what they're doing
- Clean separation of concerns

**Cons:**
- Another command to remember
- User might forget and just use `/log` anyway

**Verdict:** Rejected — adds complexity without solving the core problem

### Option 4: Smart `/log-start` + `/log-resume`

**Approach:** Both commands handle different scenarios:
- `/log-start`: Detects gap since last log, warns if >4 hours
- `/log-resume`: Updates `.session.start` for current session

**Pros:**
- Comprehensive coverage
- Clear user intent

**Cons:**
- Two new commands/concepts
- Still doesn't prevent user from running `/log` directly after pause

**Verdict:** Rejected — over-engineered

## Chosen Solution: Threshold Prompt in `/log`

### Implementation

```markdown
## /log Workflow (updated)

1. **Get current timestamp** using bash

2. **Check for session file** — read `session_start` if exists

3. **Calculate duration** if valid `session_start`

4. **Threshold check** — if `duration_min > 120`:
   ```
   "Session shows X hours duration. Adjust start time? [y/no/abort]"
   - y: "Enter new start (YYYY-MM-DDTHH:MM:SSZ or 'now'):"
       - 'now' → use current timestamp
       - timestamp → validate and use
   - no: proceed with current duration
   - abort: cancel /log command
   ```

5. **Find commits, gather files, append entry**
```

### Rationale

1. **Covers pause/resume:** User sees suspicious duration, corrects it
2. **Preserves normal flow:** Short sessions (<2 hours) skip the prompt
3. **Allows legitimate long sessions:** User can say "no, this is correct"
4. **Reversible:** "abort" lets user back out and fix manually

### Alternative Thresholds

| Threshold | Trade-off |
|-----------|-----------|
| 60 min | More prompts, catches shorter pauses |
| 120 min | Balanced (recommended) |
| 240 min | Fewer prompts, misses some pauses |

Recommendation: Start with 120 min, adjust based on real usage.

## Integration with `.session.start` Update

As decided separately, `/log` will **update `.session.start`** after each log entry:

```
Before /log:
  .session.start = "2026-02-27T09:52:52Z"

After first /log (at 11:30):
  .session.start = "2026-02-27T11:30:00Z"
  duration_min = 97 (time since 09:52)

After pause and second /log (at 17:28):
  .session.start = "2026-02-27T17:28:00Z"
  duration_min = 358? → TRIGGERS THRESHOLD PROMPT
  User enters "now" → duration_min = 0 (new work segment starts)
```

This means:
- First `/log` after pause will always trigger threshold (since `.session.start` is stale)
- User corrects it with "now" or specific time
- `.session.start` gets updated for next `/log`

## Files to Update

1. `~/.claude/skills/log/SKILL.md` — Add threshold prompt step
2. `llm-logging-convention/SKILL.md` — Same
3. `llm-logging-convention/README.md` — Document pause handling
4. `llm-logging-convention/AGENTS.md.example` — Update workflow

## Future Considerations

- Could track "active" vs "paused" state in `.session`
- Could use git commit timestamps to infer actual work duration
- Could add `/log-pause` and `/log-resume` for explicit marking (if threshold proves insufficient)