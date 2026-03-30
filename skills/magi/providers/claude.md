# Claude Provider Card

## Role

Best for architectural framing, nuanced tradeoff analysis, and reviewer-style
critique.

## Transport

Transport depends on the host:

### From Claude Code (host-native)

Use Task subagent — no CLI needed. This avoids session contention.

```
Task:
  subagent_type: "general-purpose"
  model: "opus"
  prompt: "You are a senior software architect advisor. [prompt]"
```

**Why not CLI from Claude?** Running `claude -p` as a subprocess from within a
Claude Code session causes session contention and hangs.

### From Codex (external CLI)

```bash
claude -p "[prompt]" --output-format json
```

| Flag | Purpose |
|------|---------|
| `-p` / `--print` | Non-interactive mode (required for subprocess use) |
| `--output-format json` | Machine-readable output with `is_error`, `result` fields |
| `--output-format text` | Plain text output (simpler but no error metadata) |
| `--bare` | Headless mode — may improve subprocess reliability |
| `--allowed-tools` | Constrain advisor capabilities if needed |
| `--add-dir` | Grant filesystem scope if the advisor needs project context |

**Prefer `--output-format json`** for normalization — it exposes `is_error` and
structured error messages, making failure detection reliable.

## Setup

**Auth**: Claude CLI uses OAuth with a local callback server. Non-interactive
mode (`-p`) requires an existing auth session — there is no env-var or API-key
auth path like Gemini's `.env`.

To authenticate: run `claude` interactively (opens browser OAuth flow), or
`claude auth login` (needs a local port for the OAuth callback).

If not authenticated, `-p` fails fast (~0.25s) with:
```
Not logged in · Please run /login
```

**Sandbox limitation**: `claude auth login` requires a local OAuth callback
server. In sandboxed environments (like Codex), this fails with:
```
Login failed: Failed to start OAuth callback server: Failed to start server. Is port 0 in use?
```
Users must authenticate Claude **outside the sandbox** first (e.g., in a normal
terminal), then the credential is available to subprocess calls.

**Install**: Bundled with Claude Code desktop app, or `npm install -g @anthropic-ai/claude-code`

## JSON Response Shape

When using `--output-format json`, the response includes:

```json
{
  "is_error": false,
  "result": "advisor response text",
  "duration_ms": 12345,
  "total_cost_usd": 0.01,
  "session_id": "...",
  "usage": { ... }
}
```

Parse `is_error` first — if `true`, the `result` field contains the error message.

## Common Failure Modes

| Status | Reason | Error String | Action |
|--------|--------|-------------|--------|
| `blocked` | `auth` | `Not logged in · Please run /login` | Authenticate outside sandbox, then retry |
| `blocked` | `auth` | `Failed to start OAuth callback server` | Can't auth in sandbox — auth externally first |
| `unavailable` | `missing_cli` | `command not found` | Install Claude Code CLI |
| `blocked` | `permission` | Host disallows subprocess execution | Show host-specific setup |

All failures are fast (<0.3s) and clean — no hanging in `-p` mode.

## Notes

- In Claude-hosted runs, Claude is the local advisor — no external invocation needed
- From Codex, `claude -p` is structurally viable but auth is the practical blocker.
  Users must authenticate in an unsandboxed environment first
- JSON output mode is preferred: parse `is_error` field to detect failures
  before attempting to normalize the response content
- `--bare` does not bypass auth requirements
