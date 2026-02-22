# Tool Compatibility Analysis

## Skills (Shared Convention)

Both Claude Code and OpenCode support this path:

```
~/.claude/skills/<name>/SKILL.md
```

The file **must** be named `SKILL.md` inside a directory named after the skill. The directory name becomes the `/slash-command`.

| Tool | Path Support | Notes |
|------|--------------|-------|
| Claude Code | Primary | Full support, reads from `~/.claude/skills/` |
| OpenCode | Compatible | Listed as "Claude-compatible" path |

**Recommendation**: Use `~/.claude/skills/` for skills that should work in both tools.

### Skill Discovery Paths

**Claude Code** (global):
- `~/.claude/skills/<name>/SKILL.md`

**Claude Code** (project-local):
- `.claude/skills/<name>/SKILL.md`
- Also discovers skills from nested `.claude/skills/` directories (monorepo support)
- Skills from `--add-dir` directories are also loaded

**OpenCode** (global):
- `~/.config/opencode/skills/<name>/SKILL.md`
- `~/.claude/skills/<name>/SKILL.md` (Claude-compatible)
- `~/.agents/skills/<name>/SKILL.md` (Agent-compatible)

**OpenCode** (project-local):
- `.opencode/skills/<name>/SKILL.md`
- `.claude/skills/<name>/SKILL.md` (Claude-compatible)
- `.agents/skills/<name>/SKILL.md` (Agent-compatible)
- Walks up from cwd to git worktree root

### Skill Format

```markdown
---
name: my-skill
description: What this skill does
argument-hint: [optional-args]
---

Skill instructions here...
```

### Frontmatter Fields

| Field | Claude Code | OpenCode |
|-------|-------------|----------|
| `name` | Optional (defaults to directory name) | Required (1-64 chars, `^[a-z0-9]+(-[a-z0-9]+)*$`) |
| `description` | Recommended (defaults to first paragraph) | Required (1-1024 chars) |
| `argument-hint` | Supported | Not documented |
| `disable-model-invocation` | Supported | Not documented |
| `user-invocable` | Supported | Not documented |
| `allowed-tools` | Supported | Not documented |
| `context` | Supported (`fork`) | Not documented |
| `agent` | Supported | Not documented |
| `model` | Supported | Not documented |
| `license` | Not documented | Supported |
| `compatibility` | Not documented | Supported |
| `metadata` | Not documented | Supported (string-to-string map) |

**Cross-tool safe frontmatter**: Both tools ignore unknown fields, so including Claude-specific fields like `argument-hint` won't break OpenCode, and vice versa.

### String Substitutions

| Variable | Claude Code | OpenCode |
|----------|-------------|----------|
| `$ARGUMENTS` | Supported | Unknown |
| `$ARGUMENTS[N]` / `$N` | Supported | Unknown |
| `${CLAUDE_SESSION_ID}` | Supported | Unknown |

## Commands (No Shared Convention)

Each tool uses a different path:

| Tool | Commands Path |
|------|---------------|
| Claude Code | `~/.claude/commands/<name>.md` |
| OpenCode | `~/.config/opencode/commands/<name>.md` |

Claude Code treats commands and skills as equivalent — a file at `.claude/commands/review.md` and `.claude/skills/review/SKILL.md` both create `/review`. If both exist, the skill takes precedence.

**Recommendation**: Use skills instead of commands for cross-tool compatibility.

## Status (2026-02-23)

This analysis is based on:
- Claude Code docs: https://code.claude.com/docs/en/skills
- OpenCode docs: https://opencode.ai/docs/skills/

**Both tools are evolving.** Check current docs if things seem broken.

## Action Items

- [ ] Monitor for changes in either tool's skill/command system
- [ ] Test `$ARGUMENTS` substitution in OpenCode
- [ ] Document any edge cases discovered in practice
