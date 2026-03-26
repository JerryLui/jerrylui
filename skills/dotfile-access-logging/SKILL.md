---
name: dotfile-access-logging
description: Set up a Claude Code hook to log all read/edit/write/shell access to dotfiles and hidden directories. Use when a user wants to audit or monitor sensitive file access.
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

### Logging hook

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Read|Edit|Write|Glob|Grep|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '(.tool_input.file_path // .tool_input.path // .tool_input.command // \"\") as $target | select($target | split(\"/\") | map(select(. != \"\" and . != \".\" and . != \"..\")) | any(startswith(\".\"))) | \"\\(now | strftime(\"%Y-%m-%dT%H:%M:%SZ\")) \\(.session_id) \\(.tool_name) \\($target)\"' >> ~/.claude/dotfile-access.log 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### Blocking hook (optional)

Use a `PreToolUse` hook to **block** dotfile access instead of (or in addition to) logging it. A `PreToolUse` hook must **exit 2** to block the tool call — any other exit code allows it through.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Edit|Write|Glob|Grep|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "INPUT=$(cat); TARGET=$(echo \"$INPUT\" | jq -r '(.tool_input.file_path // .tool_input.path // .tool_input.command // \"\")'); if echo \"$TARGET\" | grep -qE '(^|/)\\.(?!\\.|/|$)'; then echo \"Blocked: dotfile access not allowed\" >&2; exit 2; fi; exit 0"
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
| `"matcher": "Read\|Edit\|Write\|Glob\|Grep\|Bash"` | Fires on all file-access tools including shell commands |
| `(.tool_input.file_path // .tool_input.path // .tool_input.command // "")` | Extracts the relevant path from any tool type |
| `select(... any(startswith(".")))` | Filters to paths with a dotfile/hidden directory component |
| `map(select(. != "" and . != "." and . != ".."))` | Ignores empty segments, `.`, and `..` to avoid false positives |
| `.session_id` | Links each entry to a Claude Code session |
| `>> ~/.claude/dotfile-access.log` | Appends to a persistent log file |
| `exit 2` (blocking hook) | The **only** exit code that blocks a `PreToolUse` tool call — exit 0 and exit 1 both allow it through |

### Hook payload fields used

| Tool | Field containing the target path |
|------|----------------------------------|
| `Read`, `Edit`, `Write` | `tool_input.file_path` |
| `Glob` | `tool_input.path` (search directory) |
| `Grep` | `tool_input.path` (search directory) |
| `Bash` | `tool_input.command` (full shell command) |

Other common fields: `tool_name`, `session_id`, `cwd`, `hook_event_name`.

### Coverage and limitations

The `Bash` matcher catches shell commands like `cat ~/.env` or `head ~/.ssh/id_rsa`. Because the filter checks the *entire command string* for dotfile path segments, it will log any Bash command that mentions a dotfile path — even in comments or echo strings. This is intentional: it's better to over-log than miss access.

However, indirect access (e.g., `Bash: cat $(echo ~)/.env` or reading a symlink that points to a dotfile) cannot be reliably detected from the command string alone.

## Log Format

```
<ISO timestamp> <session-id> <tool> <target>
```

Example:

```
2026-03-26T13:27:53Z 6dd58299 Read /Users/you/.zshrc
2026-03-26T13:28:10Z 6dd58299 Edit /Users/you/.claude/settings.json
2026-03-26T13:29:00Z 6dd58299 Bash cat /Users/you/.ssh/id_rsa
2026-03-26T13:29:15Z 6dd58299 Grep /Users/you/.aws
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
jq -r '"\\(now | strftime(\"%Y-%m-%dT%H:%M:%SZ\")) \\(.session_id) \\(.tool_name) \\(.tool_input.file_path // .tool_input.path // .tool_input.command // \"N/A\")"' >> ~/.claude/file-access.log 2>/dev/null || true
```

### Narrower scope — file tools only

If you only want to log direct file reads/writes and not shell commands, use a narrower matcher. This avoids noisy Bash logging but leaves the shell as a blind spot:

```json
{
  "matcher": "Read|Edit|Write",
  "hooks": [
    {
      "type": "command",
      "command": "jq -r 'select((.tool_input.file_path // \"\") | split(\"/\") | map(select(. != \"\" and . != \".\" and . != \"..\")) | any(startswith(\".\"))) | \"\\(now | strftime(\"%Y-%m-%dT%H:%M:%SZ\")) \\(.session_id) \\(.tool_name) \\(.tool_input.file_path)\"' >> ~/.claude/dotfile-access.log 2>/dev/null || true"
    }
  ]
}
```
