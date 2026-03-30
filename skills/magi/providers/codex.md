# Codex Provider Card

## Role

Best for implementation-oriented patterns, security/CI instincts, and concise
execution-focused advice.

## Transport

```bash
codex exec --sandbox read-only --skip-git-repo-check -- "[prompt]"
```

| Flag | Purpose |
|------|---------|
| `exec` | Non-interactive mode |
| `--sandbox` | `read-only`, `workspace-write`, `danger-full-access` |
| `--skip-git-repo-check` | Run outside git repositories |
| `--model` | Model override (default: gpt-5.4) |

## Setup

**Install**: `npm install -g @openai/codex && codex login`

**Config**: Set `model_reasoning_effort = "high"` in `~/.codex/config.toml`

## Common Failure Modes

| Status | Reason | Action |
|--------|--------|--------|
| `failed` | `network` | Sandboxed environment can't reach backend |
| `unavailable` | `auth` | Login missing or expired — run `codex login` |
| `unavailable` | `missing_cli` | Show install command above |
| `blocked` | `permission` | Host blocks subprocess execution |

## Notes

- If the current host IS Codex, do not launch `codex exec` as a second advisor
  by default — that duplicates the host and may fail in sandbox
- Only use a second Codex subprocess when the user explicitly wants an
  independent additional pass
