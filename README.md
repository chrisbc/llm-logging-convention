# LLM Logging Convention

A cross-tool convention for logging LLM coding assistant sessions to a project repository.

## Problem

Different LLM tools (Claude Code, OpenCode, Cursor, etc.) work on the same projects but don't share session history. This makes it hard to:

- Track what work was done across sessions
- Know which tool did what
- Capture decisions made during sessions
- Maintain continuity between different AI assistants

## Solution

A simple JSONL log file (`.llm/log.jsonl`) + an `AGENTS.md` file that instructs all LLM tools how to log their sessions.

```
your-project/
├── .llm/
│   └── log.jsonl        # Append-only session log
├── AGENTS.md            # Instructions for LLM tools (includes logging convention)
└── ... (your project files)
```

## How It Works

1. **`AGENTS.md`** — Many LLM tools read this file for project context. We include the logging convention here.

2. **`.llm/log.jsonl`** — Append-only JSON Lines file. Each session adds one line. Any tool can append without conflicts.

3. **`/log` command** — A slash command users can invoke to add entries with their own notes.

## Log Schema

```json
{
  "timestamp": "2025-02-22T10:30:00Z",
  "session_start": "2025-02-22T09:15:00Z",
  "project_name": "NutBot CityLimit",
  "project_root": "nutbot-citylimit",
  "model": "claude-3-5-sonnet",
  "tool": "claude-code",
  "duration_min": 25,
  "tokens_in": 15000,
  "tokens_out": 8000,
  "commits": ["e263c5e", "abc1234"],
  "files": {
    "read": ["PLAN.md", "TRACTOR.md"],
    "created": ["README.md"],
    "modified": ["config.yaml"]
  },
  "summary": "Brief description of work done",
  "user_notes": "Optional user-provided notes"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `timestamp` | Yes | When the log entry was created (ISO 8601, UTC) |
| `session_start` | No | When the session began (ISO 8601, UTC) |
| `project_name` | Yes | Human-readable project name |
| `project_root` | Yes | Root folder name (for aggregation across projects) |
| `model` | Yes | Model identifier |
| `tool` | Yes | Tool name (claude-code, opencode, cursor, etc.) |
| `duration_min` | No | Approximate session duration in minutes |
| `tokens_in` | No | Input tokens |
| `tokens_out` | No | Output tokens |
| `commits` | No | Git commit hashes from this session |
| `files.read` | No | Files read during session |
| `files.created` | No | New files created |
| `files.modified` | No | Existing files modified |
| `summary` | Yes | Brief work description |
| `user_notes` | No | User-provided context |

### Data Accuracy

Not all fields are equally reliable. LLM agents typically don't have direct access to their own session metadata (start time, token counts, etc.), so some values are best-effort estimates.

- **Always precise**: `timestamp` (log creation time), `model`, `tool`, `commits`, `files`, `summary`
- **Precise if `/log-start` was used**: `session_start`, `duration_min`
- **Estimated otherwise**: `session_start` (inferred from first commit or heuristic), `duration_min` (approximate), `tokens_in`/`tokens_out` (only if the tool exposes usage data)

Consumers of log data should treat estimated fields as indicative, not exact.

## Usage

### For Users

```
/log-start                     # Mark session start (records timestamp)
/log-start updating ROS2 deps  # Mark start with a topic
/log                           # Log session, prompts for notes
/log decided to use RoboClaw   # Log with notes
```

Using `/log-start` at the beginning of a session makes `session_start` and `duration_min` precise instead of estimated.

On session exit, tools should ask: "Log this session before exiting?"

### For LLM Tool Developers

1. Read `AGENTS.md` if present in project root
2. Parse the `/log` command section
3. Implement the logging behavior as specified
4. Append entries to `.llm/log.jsonl`

## Supported Tools

| Tool | Status | Notes |
|------|--------|-------|
| OpenCode | ✅ Supported | Reads AGENTS.md; loads `~/.claude/skills/` (global skill) |
| Claude Code | ✅ Supported | Reads AGENTS.md; loads `~/.claude/skills/` (global skill) |
| Cursor | ✅ Supported | Reads AGENTS.md |
| Other tools | Easy to add | Just read AGENTS.md and follow convention |

## Files

```
llm-logging-convention/
├── README.md              # This file
├── AGENTS.md.example      # Copy to your project root as AGENTS.md (per-project)
├── SKILL.md               # /log skill for ~/.claude/skills/log/
├── SKILL-log-start.md     # /log-start skill for ~/.claude/skills/log-start/
└── log.jsonl.example      # Example log entries
```

## Installation

### Option 1: Per-Project (Recommended)

1. Copy `AGENTS.md.example` to your project root as `AGENTS.md`
2. Customize the "Project Context" section for your project
3. Create `.llm/` directory (tools will create `log.jsonl` on first run)

### Option 2: Global Skill (All Projects)

Install the skills globally so they're available across all projects without modifying each `AGENTS.md`:

```bash
mkdir -p ~/.claude/skills/log ~/.claude/skills/log-start
cp SKILL.md ~/.claude/skills/log/SKILL.md
cp SKILL-log-start.md ~/.claude/skills/log-start/SKILL.md
```

**Critical**: The file must be named `SKILL.md` inside a directory named after the skill (e.g. `log/SKILL.md`). The directory name becomes the `/slash-command`. Both Claude Code and OpenCode discover skills from `~/.claude/skills/`.

**Benefits**:
- Works across all projects automatically
- Loaded on-demand (efficient — only loads schema when you invoke `/log`)
- Compatible with both OpenCode and Claude Code (both read `~/.claude/skills/`)

**Trade-off**: Does NOT auto-log on session exit. You must invoke `/log` manually.

## Gitignore

Add `.llm/.session` to your `.gitignore` — it's a transient file that shouldn't be committed:

```
.llm/.session
```

The log file (`.llm/log.jsonl`) _should_ be committed — that's the whole point.

## Why JSONL?

- **Append-only**: No need to read/modify existing content
- **Line-delimited**: Each line is independent, no array wrapper needed
- **Streaming friendly**: Can process entries one at a time
- **Conflict resistant**: Multiple tools can append without merge conflicts
- **Human readable**: One JSON object per line, easy to inspect

## Example Log

```json
{"timestamp":"2025-02-22T10:30:00Z","session_start":"2025-02-22T09:15:00Z","project_name":"NutBot CityLimit","project_root":"nutbot-citylimit","model":"claude-3-5-sonnet","tool":"opencode","duration_min":25,"tokens_in":15000,"tokens_out":8000,"commits":["e263c5e"],"files":{"read":["PLAN.md","TRACTOR.md"],"created":["README.md"],"modified":[]},"summary":"Add project README and initial docs","user_notes":"decided to use RoboClaw over RC ESC"}
{"timestamp":"2025-02-22T14:00:00Z","project_name":"NutBot CityLimit","project_root":"nutbot-citylimit","model":"claude-3-5-sonnet","tool":"claude-code","duration_min":45,"commits":["a1b2c3d"],"files":{"read":["README.md","tractor/MOTOR_CONTROLLER.md"],"created":["tractor/ROS2_SETUP.md"],"modified":["README.md"]},"summary":"Document ROS2 installation and Linorobot2 setup","user_notes":"next: test on hardware"}
```

## Future Enhancements

- [ ] CLI tool to query/filter logs
- [ ] Web dashboard for log visualization
- [ ] Integration with git hooks
- [ ] Cost tracking (tokens × pricing)

## License

MIT