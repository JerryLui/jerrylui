---
name: dotfile-access-logging
description: Set up a Claude Code hook to log all read/edit/write access to dotfiles and hidden directories. Use when a user wants to audit or monitor sensitive file access.
origin: custom
---

# Dotfile Access Logging

Log all Claude Code file access to dotfiles (`.env`, `.ssh/`, `.claude/`, `.bashrc`, etc.) for auditing purposes.

## Prerequisites

- `jq` installed (`brew install jq` on macOS, `apt install jq` on Ubuntu/Debian)
- No plugins required — hooks are a built-in Claude Code feature

## Setup

Add the following hook to `~/.claude/settings.json` (global) or `.claude/settings.json` (per-project).

If the file doesn't exist, create it with the full JSON below. If it already exists, merge the `PostToolUse` entry into the existing `hooks` object.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Read|Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r 'select((.tool_input.file_path // \"\") | split(\"/\") | any(startswith(\".\"))) | \"\\(now | strftime(\"%Y-%m-%dT%H:%M:%SZ\")) \\(.session_id) \\(.tool_name) \\(.tool_input.file_path)\"' >> ~/.claude/dotfile-access.log 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### Optional: Log failed access attempts

Add a `PostToolUseFailure` hook to catch attempts to read dotfiles that don't exist:

```json
{
  "hooks": {
    "PostToolUseFailure": [
      {
        "matcher": "Read|Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r 'select((.tool_input.file_path // \"\") | split(\"/\") | any(startswith(\".\"))) | \"\\(now | strftime(\"%Y-%m-%dT%H:%M:%SZ\")) \\(.session_id) FAILED \\(.tool_name) \\(.tool_input.file_path)\"' >> ~/.claude/dotfile-access.log 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

## How It Works

| Component | Purpose |
|-----------|---------|
| `"matcher": "Read\|Edit\|Write"` | Fires only on file access tools |
| `select(... any(startswith(".")))` | Filters to paths with a dotfile/hidden directory component |
| `.session_id` | Links each entry to a Claude Code session |
| `>> ~/.claude/dotfile-access.log` | Appends to a persistent log file |

### Hook payload fields used

- `tool_input.file_path` — the file being accessed
- `tool_name` — `Read`, `Edit`, or `Write`
- `session_id` — unique ID for the Claude Code session

## Log Format

```
<ISO timestamp> <session-id> <tool> <file-path>
```

Example:

```
2026-03-26T13:27:53Z 6dd58299-c2a6-445a-8a52-8acc25f8ea1b Read /Users/you/.zshrc
2026-03-26T13:28:10Z 6dd58299-c2a6-445a-8a52-8acc25f8ea1b Edit /Users/you/.claude/settings.json
2026-03-26T13:29:00Z 6dd58299-c2a6-445a-8a52-8acc25f8ea1b FAILED Read /Users/you/.bashrc
```

## Reading the Logs

```bash
# View all logged access
cat ~/.claude/dotfile-access.log

# Filter by session
grep "your-session-id" ~/.claude/dotfile-access.log

# Filter by tool type
grep "Edit" ~/.claude/dotfile-access.log

# Watch in real-time (run in a separate terminal)
tail -f ~/.claude/dotfile-access.log

# Truncate the log (manual rotation)
> ~/.claude/dotfile-access.log
```

## Customization

### Log ALL file access (not just dotfiles)

Remove the `select(...)` filter:

```
jq -r '"\\(now | strftime(\"%Y-%m-%dT%H:%M:%SZ\")) \\(.session_id) \\(.tool_name) \\(.tool_input.file_path // \"N/A\")"' >> ~/.claude/file-access.log 2>/dev/null || true
```

### Block dotfile access instead of logging

Use a `PreToolUse` hook that exits non-zero to deny the operation:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -e 'select((.tool_input.file_path // \"\") | split(\"/\") | any(startswith(\".\"))) | halt_error(1)' > /dev/null 2>&1; test $? -ne 1"
          }
        ]
      }
    ]
  }
}
```
