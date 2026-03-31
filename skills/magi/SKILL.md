---
name: magi
description: Multi-AI counsel system. Query Gemini, Codex, Claude advisors in parallel with synthesis. Use for second opinions, plan synthesis, debugging, researching APIs, reviewing code, architecture decisions, or alternative perspectives. Use this whenever the user wants advice, says "what do you think", asks for multiple viewpoints, or would benefit from a second opinion.
argument-hint: "[prompt or competing plans]"
allowed-tools: Bash, Read, Glob, Grep, Task
# Note: Write/Edit intentionally excluded - magi is advisory only
---

# Magi

Multi-advisor counsel: query AI advisors, normalize results, synthesize agreement
and disagreement, return one recommendation with inspectable evidence.

The workflow is host-neutral. Host adapters and provider cards supply execution
details; the protocol stays the same whether you're in Claude Code or Codex.

## When to Use

- The user wants multiple perspectives or says "what do you think"
- Architecture, design, or implementation decisions with real tradeoffs
- Debugging strategy where blind spots matter
- Competing plans or proposals that need blending
- Code review or research where breadth adds value

## When NOT to Use

- Trivial questions where you already know the answer — magi adds 30-60s latency
- Subjective style preferences with no meaningful tradeoffs
- Time-sensitive execution where the user wants action, not counsel
- Solo deep reasoning — use critical-thinking instead

## Modes

### Counsel (default)

One question, one decision. Each advisor gets the same prompt with enough project
context to avoid generic answers.

### Plan Synthesis

The user has 2-3 competing plans. Each advisor identifies what each plan does
better, what it misses, and proposes a hybrid. Output includes a conflict table:

```markdown
| Decision Point | Plan A | Plan B | Hybrid Choice | Reasoning |
|---|---|---|---|---|
```

## Workflow

### Step 1: Identify Host

Determine which environment you're running in — this controls how advisors are
invoked and what concurrency primitives exist.

- **Claude Code**: [hosts/claude.md](hosts/claude.md)
- **Codex**: [hosts/codex.md](hosts/codex.md)

### Step 2: Build Advisor Roster

Start with the host-native local advisor, then add external advisors.

**Why this order:** The local advisor is always available and free. External
advisors need CLI tools, API keys, and permissions — they may fail.

**Never double-count the host.** If you're running in Claude, don't launch
`claude -p` as a second advisor (causes session contention and hangs). If you're
in Codex, don't launch `codex exec` (duplicates the host and may fail in sandbox).
The host IS the local advisor already.

Provider details and CLI reference: [reference.md](reference.md)

### Step 3: Query Advisors in Parallel

**On Claude Code:** Run as a single background Task that queries all advisors and
returns the complete synthesis. This keeps the main thread clean — it only receives
the final report, not partial results.

```
Task:
  subagent_type: "general-purpose"
  model: "opus"
  run_in_background: true
  prompt: |
    You are orchestrating a magi counsel session.

    **User's question**: [prompt]

    **Instructions**:
    1. Run these Bash commands in parallel for external advisors:
       - gemini -p "[prompt]" --model gemini-3.1-pro-preview --sandbox -o text
       - Codex: prefer plugin companion if installed (see providers/codex.md),
         fall back to: codex exec --sandbox read-only --skip-git-repo-check -- "[prompt]"
    2. Formulate your own response as the Claude advisor
    3. Wait for ALL results before proceeding
    4. If Gemini fails with 429/capacity: wait 60s, retry once, then skip
    5. If EITHER command fails with "denied by policy": STOP — return only
       the setup message (see Failure Handling below)
    6. Normalize each result and synthesize per the output format below
```

**On Codex:** Use the current session as the local advisor. Launch external CLIs
for the other advisors:

```bash
# Gemini
gemini -p "[prompt]" --model gemini-3.1-pro-preview --sandbox -o text

# Claude (use JSON for reliable error detection)
claude -p "[prompt]" --output-format json
```

For Claude JSON output, check `is_error` before normalizing — `true` means auth
or other failure. See [hosts/codex.md](hosts/codex.md) for full details.

### Step 4: Normalize Results

Every advisor response becomes:

```yaml
advisor_id: "gemini|codex|claude"
status: "ok|unavailable|blocked|failed"
summary: "1-3 sentence essence of the advisor's position"
content: "full response or unavailability message"
```

**Why normalize:** Without this, synthesis devolves into comparing apples to
oranges. Structured results make agreement and disagreement explicit, and let
the synthesizer work from a consistent shape.

Status meanings:
- `ok` — usable content returned
- `unavailable` — dependency missing or not configured (tolerable)
- `blocked` — host policy prevented execution (setup problem — see Failure Handling)
- `failed` — execution attempted but errored

### Step 5: Synthesize

This is the step that makes magi more than fan-out plus transcript dumping.
Without synthesis, the user has to read three responses and do the comparison
work themselves.

| Pattern | When | Action |
|---|---|---|
| **Consensus** | Advisors mostly agree | Proceed, but note shared blind spots |
| **Complementary** | Different but compatible | Combine strongest non-overlapping insights |
| **Conflict** | Direct contradiction | Compare evidence quality, prefer simpler/reversible |
| **Gap** | One advisor silent | Note the gap; do not invent agreement |

**On conflicts:** Good evidence is specific, testable, or grounded in project
constraints. Weak evidence is vague confidence without support. When in doubt,
prefer existing project patterns.

Answer these explicitly:
1. What did the advisors broadly agree on?
2. Where did they disagree?
3. Which disagreements matter vs style-level noise?
4. What decision best fits the actual project context?
5. What uncertainty remains?

See [synthesis-guide.md](synthesis-guide.md) for the full report template. Label the
host-native advisor by its actual name (Claude or Codex), not a generic "local
advisor" — the user should see which model said what.

### Step 6: Review (when needed)

Add a separate review pass when:
- The decision is high-stakes or will directly drive implementation
- Advisors materially disagree
- The synthesis collapsed important nuance

The reviewer inspects the synthesized artifact for false consensus, dropped
caveats, overweighting one advisor, or hard-to-reverse recommendations. It does
not re-run the entire council.

## Failure Handling

| Error Type | Action | Why |
|---|---|---|
| Permission denied | **STOP** — show host-specific setup guidance | Magi's value is multi-perspective. Single-advisor fallback is just a normal conversation |
| 429 / capacity | Wait 60s → retry → proceed without | Gemini pro preview has limited capacity |
| Auth / missing API key | Show setup instructions per [reference.md](reference.md) | Non-interactive CLI mode needs explicit env vars |
| CLI not found | Show install instructions per [reference.md](reference.md) | |
| Network error | Retry once, then mark unavailable | |

**Degraded council rules:**
- 3/3 advisors → full synthesis
- 2/3 advisors → partial synthesis, note who's missing and why
- 1/3 (host-only) → only if failures are capacity/network. State it's single-advisor
- Permission blocked → **STOP**, do not synthesize. Show setup message:

```
## Magi Setup Required

The magi skill needs permission to run external AI advisors.

Add these to your `.claude/settings.local.json` permissions.allow array:
- "Bash(gemini *)"
- "Bash(codex *)"

Then restart Claude Code and try again.
```

(Codex hosts: show Codex-specific sandbox/approval guidance instead.)

## Usage Examples

When `$ARGUMENTS` is empty, show:

```
Usage:
  /magi "prompt"                     # Counsel mode (default)
  /magi "Plan A vs Plan B"           # Plan synthesis mode

Examples:
  /magi "How should we implement caching for this service?"
  /magi "What's the best approach for authentication here?"
  /magi "Review this architecture for potential issues"
  /magi "Compare these two migration strategies and recommend a hybrid"
```

## References

- Provider capabilities and CLI details: [reference.md](reference.md)
- Synthesis patterns and report template: [synthesis-guide.md](synthesis-guide.md)
- Host adapters: [hosts/claude.md](hosts/claude.md), [hosts/codex.md](hosts/codex.md)
