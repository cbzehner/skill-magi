# Magi Reference

Provider details, normalization schema, and report template.

---

## Providers

| Advisor | Model | Unique Capability | Best For |
|---------|-------|-------------------|----------|
| Gemini | gemini-3.1-pro-preview | Native web search (`google_web_search`) | Current info, API docs, library research |
| Codex | gpt-5.4 | Sandboxed code execution | Verification, CI/CD patterns, security |
| Claude | Opus 4.6 | Deep reasoning, project context | Architecture, code review, complex problems |

### Gemini CLI

```bash
gemini -p "[prompt]" --model gemini-3.1-pro-preview --sandbox -o json
```

| Flag | Purpose |
|------|---------|
| `-p` | Non-interactive mode |
| `--model` | Model selection |
| `--sandbox` | Read-only, no file writes |
| `-o json` | JSON output with `response` (content) and `stats` (metrics) |

**Metrics from JSON:** Token counts in `stats.models.<model>.tokens` (input,
candidates, thoughts, total). Latency in `stats.models.<model>.api.totalLatencyMs`.

**Setup:** `npm install -g @google/gemini-cli && gemini --login`

**Non-interactive auth:** `-p` mode doesn't read stored credentials. Create
`~/.gemini/.env` with `GEMINI_API_KEY=your-key` for subprocess use.
Get a key from https://aistudio.google.com/app/apikey

**Capacity errors:** On 429: wait 60s → retry once → skip.

### Codex CLI

Prefer the plugin companion when installed:

```bash
CODEX_COMPANION="${CLAUDE_PLUGIN_ROOT:-$HOME/.claude/plugins/cache/openai-codex/codex}/scripts/codex-companion.mjs"
node "$CODEX_COMPANION" task --read-only --json "[prompt]"
```

Fallback:

```bash
codex exec --sandbox read-only --skip-git-repo-check -- "[prompt]"
```

| Flag | Purpose |
|------|---------|
| `exec` | Non-interactive mode |
| `--sandbox` | `read-only`, `workspace-write`, `danger-full-access` |
| `--skip-git-repo-check` | Run outside git repos |

**Metrics:** Aggregate token count only (single number, no in/out split). No
timing from CLI — measure wall time externally.

**Setup:** `npm install -g @openai/codex && codex login`

**Config:** `model_reasoning_effort = "high"` in `~/.codex/config.toml`

### Claude

**From Claude Code (host-native):** Task subagent — no CLI needed.

```
Task:
  subagent_type: "general-purpose"
  model: "opus"
  prompt: "[advisor prompt]"
```

Don't use `claude -p` from within Claude Code — causes session contention.

**From Codex (external CLI):**

```bash
claude -p "[prompt]" --output-format json
```

Check `is_error` before normalizing — `true` means auth or other failure.

**Metrics from JSON:** `cost_usd`, `duration_ms`, `input_tokens`, `output_tokens`.

**Auth:** Requires existing login. No env-var auth. Fails fast with "Not logged
in" if not authenticated. Must auth outside sandbox environments first.

**Setup:** `npm install -g @anthropic-ai/claude-code`

### Common Failure Detection

| Error String | Provider | Status | Reason |
|---|---|---|---|
| `Not logged in` | Claude | `blocked` | `auth` |
| `Failed to start OAuth callback server` | Claude | `blocked` | `auth` |
| `must specify GEMINI_API_KEY` | Gemini | `unavailable` | `auth` |
| `command not found` | Any | `unavailable` | `missing_cli` |
| Sandbox/approval rejection | Any | `blocked` | `permission` |
| Network timeout | Any | `failed` | `network` |

---

## Normalization Schema

```yaml
advisor_id: "gemini|codex|claude"
status: "ok|unavailable|blocked|failed"
summary: "1-3 sentence essence of the advisor's position"
assumptions: ["beliefs taken for granted in this response"]
information_gaps: ["what the advisor didn't know or couldn't verify"]
implications: ["consequences if this advice is followed, especially hard-to-reverse ones"]
evidence_basis: ["sources cited, code inspected, commands run, docs referenced"]
content: "full response or unavailability message"
```

These fields replace self-reported confidence with observable inputs:

- An advisor with many unstated assumptions carries less weight than one that
  made its assumptions explicit and justified them
- Incompatible assumptions between advisors often cause apparent disagreement —
  resolving the assumption resolves the conflict
- Information gaps that overlap across advisors represent shared blind spots
- An advisor that grounded its response in evidence (cited docs, ran code,
  searched the web) carries more weight than one reasoning from general knowledge
- Implications flagged as hard-to-reverse should bias toward the more cautious
  recommendation

Fields populated only when `status: "ok"`. Omit for other statuses.

---

## Synthesis Patterns

| Pattern | When | Action |
|---------|------|--------|
| **Consensus** | Advisors mostly agree | Proceed; watch for shared blind spots |
| **Complementary** | Different but compatible | Combine strongest non-overlapping insights |
| **Conflict** | Direct contradiction | Compare evidence quality, prefer simpler/reversible |
| **Gap** | One advisor silent | Note gap; do not invent agreement |

Good evidence = specific references, tested claims, grounded in project
constraints. Weak evidence = vague confidence without support. When in doubt,
prefer existing project patterns.

---

## Report Template

Replace `[Host Advisor]` with Claude or Codex.

```markdown
## Quick Answer
[1-2 sentence actionable recommendation]

## Session Metrics
| Advisor | Status | Wall Time | Tokens (in/out) |
|---------|--------|-----------|-----------------|
| [Host Advisor] | ✓/✗ | Xs | Nk / Nk |
| Gemini | ✓/✗ | Xs | Nk / Nk |
| Codex | ✓/✗ | Xs | Nk total |
| Critique | ✓/— | Xs | — |
| **Total** | | **Xs** | |

Debate: [yes/no]

<details>
<summary>Gemini Response</summary>

[Full response or "Unavailable: [reason]"]

</details>

<details>
<summary>Codex Response</summary>

[Full response or "Unavailable: [reason]"]

</details>

<details>
<summary>[Host Advisor] Response</summary>

[Full response or "Unavailable: [reason]"]

</details>

## Synthesis
| Advisor | Key Insight |
|---------|-------------|
| Gemini | ... |
| Codex | ... |
| [Host Advisor] | ... |

**Consensus**: [What they agreed on]
**Conflicts**: [Disagreements and resolution]
**Recommendation**: [Your synthesized advice]

_Session saved to `[file path]`_
```

**Metrics availability:**
- **Gemini** (`-o json`): input/output/thoughts tokens + API latency
- **Codex** (`exec`): aggregate token count only, no timing
- **Claude** (host-native): varies; `-p --json` reports full metrics

Use `—` for unavailable metrics. Never estimate or fabricate.

Simple queries may skip collapsible individual responses — Quick Answer +
Session Metrics + Synthesis table is sufficient.
