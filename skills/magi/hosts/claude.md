# Claude Host Adapter

How magi runs when the host is Claude Code.

## Local Advisor

The Claude host itself — use a Task subagent with `model: "opus"`.

## Default Roster

1. Claude (host-native local advisor via Task)
2. Gemini (CLI) when available
3. Codex (CLI) when available

## Concurrency

Run the local advisor and external CLIs in parallel using a single background
Task that handles all queries and returns the complete synthesis.

## Permission Setup

If external advisors are blocked by policy ("denied by policy" errors), STOP
and show this message — do not fall back to Claude-only:

```
Add these to your .claude/settings.local.json permissions.allow array:
- "Bash(gemini *)"
- "Bash(codex *)"

Then restart Claude Code and try again.
```

## Report Labels

- Host-native advisor is labeled "Claude"
- External advisors labeled by provider name
