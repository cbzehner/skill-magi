# Magi Reference

Advisor capabilities, CLI details, and setup instructions. Loaded on demand.

---

## Advisor Capabilities

| Advisor | Model | Strengths | Best For |
|---------|-------|-----------|----------|
| Gemini | gemini-3.1-pro-preview | Web search, 1M token context, advanced reasoning | Current info, API docs, library research |
| Codex | gpt-5.4 (high reasoning) | Fast (Rust-based), fine-grained sandboxing | Quick analysis, CI/CD patterns, security |
| Claude | Opus 4.6 | Deep reasoning, code expertise | Architecture, code review, complex problems |

---

## CLI Reference

### Gemini

```bash
gemini -p "[prompt]" --model gemini-3.1-pro-preview --sandbox -o text
```

| Flag | Purpose |
|------|---------|
| `-p` / `--prompt` | Non-interactive mode (required for subprocess use) |
| `--model` | `gemini-3.1-pro-preview` (max reasoning) |
| `--sandbox` | Read-only mode, no file writes |
| `-o` | Output format: `text`, `json`, `stream-json` |

**Capacity errors**: Pro preview has limited capacity. On 429: wait 60s → retry
with same model → skip if still fails.

**Install**: `npm install -g @google/gemini-cli && gemini --login`

**Non-interactive auth**: The CLI's non-interactive mode (`-p` flag) does not
read stored credentials. Create `~/.gemini/.env` with `GEMINI_API_KEY=your-key`
for subprocess use.

For "must specify GEMINI_API_KEY" errors, show:
```
The Gemini CLI stores your API key in encrypted storage, but non-interactive
mode (used by magi) only reads from environment variables / .env files.

Fix: Create ~/.gemini/.env with your API key:
  echo 'GEMINI_API_KEY=your-key-here' > ~/.gemini/.env
  chmod 600 ~/.gemini/.env

Get your key from https://aistudio.google.com/app/apikey
```

### Codex

```bash
codex exec --sandbox read-only --skip-git-repo-check -- "[prompt]"
```

| Flag | Purpose |
|------|---------|
| `exec` | Non-interactive mode |
| `--sandbox` | `read-only`, `workspace-write`, `danger-full-access` |
| `--skip-git-repo-check` | Run outside git repositories |
| `--model` | Model override (default: gpt-5.4) |

**Config**: `model_reasoning_effort = "high"` set in `~/.codex/config.toml`

**Install**: `npm install -g @openai/codex && codex login`

### Claude

Transport depends on host:

**From Claude Code (host-native):** Task subagent with `model: "opus"`.

```
Task:
  subagent_type: "general-purpose"
  model: "opus"
  prompt: "You are a senior software architect advisor. [prompt]"
```

Don't use `claude -p` from within Claude Code — causes session contention.

**From Codex (external CLI):**

```bash
claude -p "[prompt]" --output-format json
```

| Flag | Purpose |
|------|---------|
| `-p` / `--print` | Non-interactive mode |
| `--output-format json` | Machine-readable with `is_error` field |
| `--bare` | Headless mode for subprocess use |

**Auth**: Requires existing login session. No env-var auth. Fails fast (~0.25s)
with "Not logged in" if not authenticated.

**Install**: Bundled with Claude Code desktop, or `npm install -g @anthropic-ai/claude-code`

---

## Normalization Schema

Every advisor response is normalized to this shape before synthesis:

```yaml
advisor_id: "gemini|codex|claude"
status: "ok|unavailable|blocked|failed"
summary: "1-3 sentence essence of the advisor's position"
assumptions: ["beliefs taken for granted in this response"]
information_gaps: ["what the advisor didn't know or couldn't verify"]
implications: ["consequences if this advice is followed, especially hard-to-reverse ones"]
content: "full response or unavailability message"
```

The three analytical fields align with the Paul-Elder Elements of Thought
(Assumptions, Information, Implications). They replace self-reported confidence
— which LLMs cannot reliably calibrate — with observable, verifiable inputs.

**Field guidance for the synthesizer:**
- An advisor with many unstated assumptions carries less weight than one that
  made its assumptions explicit and justified them
- Incompatible assumptions between advisors are often the root cause of
  apparent disagreement — resolving the assumption resolves the conflict
- Information gaps that overlap across advisors represent shared blind spots
- Implications flagged as hard-to-reverse should bias toward the more cautious
  recommendation

Fields are populated only when `status: "ok"`. Omit them for other statuses.

---

## Prompt Templates

### Planning
```
Task: [description]
Context: [codebase context]
Constraints: [limitations]

Provide: 1) Approach 2) Steps 3) Risks 4) Alternatives
```

### Debugging
```
Error: [message]
Code: [relevant code]
Context: [what was attempted]

Provide: 1) Likely cause 2) Investigation steps 3) Fix 4) Prevention
```

### Research
```
Topic: [question]
Context: [why needed]

Provide: 1) Key findings 2) Sources 3) Applicability 4) Caveats
```

### Code Review
```
Code: [code]
Intent: [what it should do]

Provide: 1) Correctness 2) Issues 3) Improvements 4) Style
```

---

## Provider Cards

For host-specific failure modes and transport quirks, see:
- [providers/gemini.md](providers/gemini.md)
- [providers/codex.md](providers/codex.md)
- [providers/claude.md](providers/claude.md)
