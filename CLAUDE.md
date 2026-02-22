# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A convention + reference implementation for cross-tool LLM session logging. It defines a shared JSONL log schema and provides skill files that work across Claude Code, OpenCode, and Cursor.

This repo has no build system, tests, or dependencies — it is a documentation and convention repo.

## Key Files

| File | Purpose |
|------|---------|
| `SKILL.md` | The `/log` skill — copy to `~/.claude/skills/log/SKILL.md` |
| `SKILL-log-start.md` | The `/log-start` skill — copy to `~/.claude/skills/log-start/SKILL.md` |
| `AGENTS.md.example` | Template to copy into target projects as `AGENTS.md` |
| `log.jsonl.example` | Reference log entries showing valid schema |
| `TOOL-COMPATIBILITY.md` | Skills/commands path differences between Claude Code and OpenCode |

## Architecture

The convention has two deployment modes:

1. **Per-project** (`AGENTS.md` in project root): The `AGENTS.md.example` is copied and customized. LLM tools read it and auto-log on session exit.

2. **Global skill** (`~/.claude/skills/log/`): `SKILL.md` and `SKILL-log-start.md` installed globally. Tools load them on-demand when `/log` or `/log-start` is invoked.

### Session lifecycle

- `/log-start [topic]` → writes `{"start":"<UTC>","topic":"..."}` to `.llm/.session`
- `/log [notes]` → reads `.llm/.session` (if present), gathers git commits/files, appends one JSON line to `.llm/log.jsonl`
- `.llm/.session` persists across multiple `/log` calls; only overwritten by next `/log-start`
- `.llm/.session` should be in `.gitignore`; `.llm/log.jsonl` should be committed

### Critical: appending to log.jsonl

The `/log` skill must use `Bash` with `>>` (shell redirect-append) to write to `log.jsonl`. LLM tools' Write and Edit commands overwrite file contents, which destroys existing log entries. This was discovered in testing and is explicitly documented in all three instruction files.

### Log schema required fields

`timestamp`, `project_name`, `project_root`, `model`, `tool`, `summary`

Optional but important: `session_start`, `duration_min`, `commits`, `files.{read,created,modified}`, `user_notes`

Data accuracy note: `session_start` and `duration_min` are precise only when `/log-start` was used; otherwise estimated.

## Skill Format

Skills use frontmatter + markdown:

```markdown
---
name: skill-name
description: When to invoke this skill
argument-hint: [optional-args]
---

Instructions here...
```

Skills in `~/.claude/skills/<name>/SKILL.md` work in both Claude Code and OpenCode (shared path). Commands (`~/.claude/commands/`) are Claude Code only — see `TOOL-COMPATIBILITY.md` for workarounds.

## When Making Changes

- Keep `SKILL.md`, `SKILL-log-start.md`, and `AGENTS.md.example` in sync — they document the same `/log` and `/log-start` behaviors from different perspectives (skill invocation vs. AGENTS.md auto-logging)
- The JSONL schema in all three files must stay identical
- `TOOL-COMPATIBILITY.md` tracks skill/command path differences between tools and should be updated as those tools evolve
