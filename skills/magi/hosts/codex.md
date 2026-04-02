# Codex Host Adapter

How magi runs when the host is Codex.

## Local Advisor

The current Codex session — do NOT launch `codex exec` as a second advisor.
That duplicates the host and may fail in sandboxed or offline environments.

## Default Roster

1. Codex (host-native local advisor — the current session)
2. Gemini (CLI) when the sandbox and network policy allow
3. Claude (CLI) — optional third advisor, requires pre-authentication outside sandbox

The default council is **Codex + Gemini**. Two advisors from different model
families is enough for useful multi-perspective counsel. Claude is an upgrade
if the user has pre-authenticated it (see Permission Setup below).

## Querying External Advisors

Launch external CLIs via Bash. If parallel execution is safe, run them
concurrently. Otherwise, run sequentially — keep the same normalized result shape.

```bash
# Gemini (use JSON for metrics — parse response field for content)
gemini -p "[prompt]" --model gemini-3.1-pro-preview --sandbox -o json

# Claude (use JSON for reliable error detection)
claude -p "[prompt]" --output-format json
```

For Claude JSON output, check `is_error` before normalizing:
- `is_error: false` → use `result` as advisor content, status `ok`
- `is_error: true` → check `result` for error string, set status accordingly

## Detecting Failures

| Error String | Provider | Status | Reason |
|---|---|---|---|
| `Not logged in · Please run /login` | Claude | `blocked` | `auth` |
| `Failed to start OAuth callback server` | Claude | `blocked` | `auth` |
| `must specify GEMINI_API_KEY` | Gemini | `unavailable` | `auth` |
| `command not found` | Either | `unavailable` | `missing_cli` |
| Sandbox/approval rejection | Either | `blocked` | `permission` |
| Network timeout | Either | `failed` | `network` |

## Permission Setup

If external advisors are blocked, explain in Codex terms:

- Command approval or sandbox policy blocked the subprocess
- Network restrictions prevented external CLI use
- Authentication or installation is missing

**For Claude auth**: Claude uses OAuth which requires a local callback server.
This won't work inside Codex's sandbox. Authenticate `claude` in a normal
terminal first (`claude auth login` or launch `claude` interactively), then the
credential is available to subprocess calls from Codex.

**For Gemini auth**: Create `~/.gemini/.env` with `GEMINI_API_KEY=your-key`.
See [providers/gemini.md](../providers/gemini.md) for details.

Do NOT tell Codex users to edit `.claude/settings.local.json` — that's a
Claude Code concept.

## Report Labels

- Host-native advisor is labeled "Codex"
- External advisors labeled by provider name (Gemini, Claude)
