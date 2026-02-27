# Platforms and Date Strategy

## Problem

Duration calculation in `/log` skill failed on macOS due to BSD `date` misinterpreting ISO 8601 timestamps with `Z` suffix.

**Observed behavior:**
- Session start: 2026-02-27T17:55:31Z
- Log time: 2026-02-27T18:00:35Z
- Expected duration: ~5 minutes
- Actual result: 785 minutes (13 hours off)

**Root cause:** BSD `date` on macOS ignores the `Z` suffix and interprets timestamps as local time instead of UTC.

```
Without TZ=UTC: 1772168131  (treats "Z" as local time)
With TZ=UTC:    1772214931  (correct UTC interpretation)
Difference:     46800 sec = 13 hours
```

## Solution: TZ=UTC Prefix

The fix is to prefix BSD and GNU date commands with `TZ=UTC`:

```bash
start_epoch=$(TZ=UTC date -j -f "%Y-%m-%dT%H:%M:%SZ" "$start" +%s 2>/dev/null || TZ=UTC date -d "$start" +%s 2>/dev/null)
```

## OS Compatibility Matrix

| OS | Date flavor | Command | Works with TZ=UTC? |
|----|------------|---------|-------------------|
| macOS | BSD | `TZ=UTC date -j -f ...` | Yes |
| Linux | GNU | `TZ=UTC date -d ...` | Yes |
| Windows (Git Bash) | GNU | `TZ=UTC date -d ...` | Yes |
| Windows (PowerShell/CMD) | — | N/A | Not supported (skills target bash) |

## Epoch Math is OS-Agnostic

Once you have epoch seconds, duration calculation is pure POSIX arithmetic:

```bash
now=$(date -u +%s)
duration_min=$(( (now - start_epoch) / 60 ))
```

- `date -u +%s` returns epoch seconds on all systems
- `$(( ))` arithmetic is POSIX shell standard
- No parsing, no timezone conversion

## Why Not Store Epoch in log.jsonl?

Analyzed adding epoch fields to log entries:

```json
{
  "timestamp": "2026-02-27T18:00:35Z",
  "timestamp_epoch": 1740678035,
  ...
}
```

**Decision: No.** Keep epoch internal to `.session` file only.

### Use Case Analysis

| Use Case | ISO 8601 | Epoch |
|----------|----------|-------|
| Human reading in editor | Excellent | Poor |
| Log aggregation (ELK, Splunk) | Native support | Native support |
| Time range queries | Requires parsing | Instant comparison |
| Duration calculation | TZ-aware parsing needed | Trivial subtraction |
| Cross-language parsing | Standard lib support | Trivial |

### Trade-offs of Adding Epoch

| Pro | Con |
|-----|-----|
| Instant duration math | Redundant data (DRY violation) |
| Avoids TZ parsing bugs | Risk of fields becoming inconsistent |
| No parsing overhead | Bloats entries (~30 chars per field) |
| Better for CLI queries | Log consumers expect ISO 8601 |

### Rationale

1. ISO 8601 is the JSON timestamp standard
2. Most log tools parse it correctly (the BSD date bug is shell-specific)
3. Python/JS/etc. handle ISO 8601 with timezone-aware libraries
4. The TZ=UTC fix solves the producer side
5. Consumers can derive epoch at query time if needed

## Implementation

Apply fix in:
- `SKILL.md` (duration calculation)
- `SKILL-log-start.md` (if any date parsing)
- `AGENTS.md.example` (duration calculation)

## Related Files

- `SKILL.md` — `/log` skill implementation
- `SKILL-log-start.md` — `/log-start` skill implementation
- `AGENTS.md.example` — Per-project agent instructions