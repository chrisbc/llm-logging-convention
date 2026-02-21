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
| `timestamp` | Yes | Session start time (ISO 8601, UTC) |
| `model` | Yes | Model identifier |
| `tool` | Yes | Tool name (claude-code, opencode, cursor, etc.) |
| `duration_min` | Yes | Session duration in minutes |
| `tokens_in` | No | Input tokens |
| `tokens_out` | No | Output tokens |
| `commits` | No | Git commit hashes from this session |
| `files.read` | No | Files read during session |
| `files.created` | No | New files created |
| `files.modified` | No | Existing files modified |
| `summary` | Yes | Brief work description |
| `user_notes` | No | User-provided context |

## Usage

### For Users

```
/log                           # Log session, prompts for notes
/log decided to use RoboClaw   # Log with notes
```

On session exit, tools should ask: "Log this session before exiting?"

### For LLM Tool Developers

1. Read `AGENTS.md` if present in project root
2. Parse the `/log` command section
3. Implement the logging behavior as specified
4. Append entries to `.llm/log.jsonl`

## Supported Tools

| Tool | Status | Notes |
|------|--------|-------|
| OpenCode | Supported | Reads AGENTS.md |
| Claude Code | Supported | Reads AGENTS.md |
| Cursor | Supported | Reads AGENTS.md |
| Other tools | Easy to add | Just read AGENTS.md and follow convention |

## Files

```
llm-logging-convention/
├── README.md           # This file
├── AGENTS.md.example   # Copy to your project root as AGENTS.md
└── log.jsonl.example   # Example log entries
```

## Installation

1. Copy `AGENTS.md.example` to your project root as `AGENTS.md`
2. Customize the "Project Context" section for your project
3. Create `.llm/` directory (tools will create `log.jsonl` on first run)

## Why JSONL?

- **Append-only**: No need to read/modify existing content
- **Line-delimited**: Each line is independent, no array wrapper needed
- **Streaming friendly**: Can process entries one at a time
- **Conflict resistant**: Multiple tools can append without merge conflicts
- **Human readable**: One JSON object per line, easy to inspect

## Example Log

```json
{"timestamp":"2025-02-22T10:30:00Z","model":"claude-3-5-sonnet","tool":"opencode","duration_min":25,"tokens_in":15000,"tokens_out":8000,"commits":["e263c5e"],"files":{"read":["PLAN.md","TRACTOR.md"],"created":["README.md"],"modified":[]},"summary":"Add project README and initial docs","user_notes":"decided to use RoboClaw over RC ESC"}
{"timestamp":"2025-02-22T14:00:00Z","model":"claude-3-5-sonnet","tool":"claude-code","duration_min":45,"tokens_in":25000,"tokens_out":12000,"commits":["a1b2c3d"],"files":{"read":["README.md","tractor/MOTOR_CONTROLLER.md"],"created":["tractor/ROS2_SETUP.md"],"modified":["README.md"]},"summary":"Document ROS2 installation and Linorobot2 setup","user_notes":"next: test on hardware"}
```

## Future Enhancements

- [ ] CLI tool to query/filter logs
- [ ] Web dashboard for log visualization
- [ ] Integration with git hooks
- [ ] Cost tracking (tokens × pricing)

## License

MIT