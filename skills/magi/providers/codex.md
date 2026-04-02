# Codex Provider Card

## Role

Best for implementation-oriented patterns, security/CI instincts, and concise
execution-focused advice.

## Transport

Prefer the Codex plugin companion script when installed — it uses the App Server
(persistent connection, broker for reuse) instead of one-shot CLI invocations.

**Plugin (preferred):**
```bash
CODEX_COMPANION="${CLAUDE_PLUGIN_ROOT:-$HOME/.claude/plugins/cache/openai-codex/codex}/scripts/codex-companion.mjs"
node "$CODEX_COMPANION" task --read-only --json "[prompt]"
```

**CLI (fallback):**
```bash
codex exec --sandbox read-only --skip-git-repo-check -- "[prompt]"
```

| Flag (CLI) | Purpose |
|------|---------|
| `exec` | Non-interactive mode |
| `--sandbox` | `read-only`, `workspace-write`, `danger-full-access` |
| `--skip-git-repo-check` | Run outside git repositories |
| `--model` | Model override (default: gpt-5.4) |

## Setup

**Install**: `npm install -g @openai/codex && codex login`

**Plugin** (optional, better transport): `/plugin marketplace add openai/codex-plugin-cc && /plugin install codex@openai-codex`

**Config**: Set `model_reasoning_effort = "high"` in `~/.codex/config.toml`

## Common Failure Modes

| Status | Reason | Action |
|--------|--------|--------|
| `failed` | `network` | Sandboxed environment can't reach backend |
| `unavailable` | `auth` | Login missing or expired — run `codex login` |
| `unavailable` | `missing_cli` | Show install command above |
| `blocked` | `permission` | Host blocks subprocess execution |

## Capability Framing

Append to the advisor prompt (after the shared question):

> "You can execute shell commands in a sandboxed environment. If a factual
> claim is quickly verifiable by running code, verify it."

This references a verified built-in capability, not an aspirational persona.

## Notes

- If the current host IS Codex, do not launch `codex exec` as a second advisor
  by default — that duplicates the host and may fail in sandbox
- Only use a second Codex subprocess when the user explicitly wants an
  independent additional pass
