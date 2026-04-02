---
name: magi
description: Multi-AI counsel system. Query Gemini, Codex, Claude advisors in parallel with synthesis. Use for second opinions, plan synthesis, debugging, researching APIs, reviewing code, architecture decisions, or alternative perspectives. Use this whenever the user wants advice, says "what do you think", asks for multiple viewpoints, or would benefit from a second opinion.
argument-hint: "[prompt or competing plans]"
allowed-tools: Bash, Read, Glob, Grep, Task, Write
# Note: Write is scoped to session persistence (~/.claude/magi/sessions/) only
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
       - gemini -p "[advisor prompt]" --model gemini-3.1-pro-preview --sandbox -o json
       - Codex: prefer plugin companion if installed (see providers/codex.md),
         fall back to: codex exec --sandbox read-only --skip-git-repo-check -- "[advisor prompt]"
    2. Formulate your own response as the Claude advisor
    3. Wait for ALL results before proceeding
    4. If Gemini fails with 429/capacity: wait 60s, retry once, then skip
    5. If EITHER command fails with "denied by policy": STOP — return only
       the setup message (see Failure Handling below)
    6. Normalize each result and synthesize per the output format below

    **Advisor prompt format** — wrap the user's question with:
    "[user's question]

    In your response, explicitly state:
    - What assumptions you are making
    - What information you don't have or couldn't verify
    - What implications follow from your recommendation, especially anything hard to reverse"
```

**On Codex:** Use the current session as the local advisor. Launch external CLIs
for the other advisors:

```bash
# Gemini (JSON for metrics — parse response field for content)
gemini -p "[prompt]" --model gemini-3.1-pro-preview --sandbox -o json

# Claude (use JSON for reliable error detection)
claude -p "[prompt]" --output-format json
```

For Claude JSON output, check `is_error` before normalizing — `true` means auth
or other failure. See [hosts/codex.md](hosts/codex.md) for full details.

### Step 3.5: Assess Tier

Determine how much process depth this question needs. Three tiers, each a
strict superset of the previous (mirroring the critical-thinking skill's
Tier 1/2/3 escalation):

| Tier | Name | Process | When |
|------|------|---------|------|
| 1 | Quick Counsel | query → normalize → synthesize | Default. Straightforward questions, advisors broadly agree |
| 2 | Structured Counsel | query → normalize → **critique round** → synthesize | Advisors materially disagree, OR question involves decisions with real tradeoffs |
| 3 | Deep Counsel | query → normalize → **critique round** → synthesize → **review** | User passes `--debate`, OR Tier 2 conditions + hard-to-reverse consequences |

**`--debate` override:** Always Tier 3, no further assessment needed.

**Two-phase assessment** (when `--debate` is not passed):

1. **Pre-query (before Step 3):** Set an initial tier floor from the question.
   Does it involve architecture, data models, public APIs, security, or other
   decisions that are hard to reverse? If yes → floor is Tier 2.
2. **Post-normalization (after Step 4):** Evaluate actual results. Are the
   advisors' assumptions incompatible? Do core recommendations contradict?
   If yes → escalate to Tier 2 (or to Tier 3 if already at Tier 2 and
   consequences are irreversible). Never downgrade from the pre-query floor.

Report the selected tier and escalation reason in the session metrics.

### Step 4: Normalize Results

Every advisor response becomes:

```yaml
advisor_id: "gemini|codex|claude"
status: "ok|unavailable|blocked|failed"
summary: "1-3 sentence essence of the advisor's position"
assumptions: ["beliefs taken for granted in this response"]
information_gaps: ["what the advisor didn't know or couldn't verify"]
implications: ["consequences if this advice is followed, especially hard-to-reverse ones"]
content: "full response or unavailability message"
```

The three analytical fields (assumptions, information_gaps, implications) are
drawn from the Paul-Elder Elements of Thought. They give the synthesizer
observable, verifiable inputs rather than unreliable self-reported confidence.
The orchestrator prompt must instruct each advisor to surface these explicitly.

- `assumptions` — what the advisor took for granted. When two advisors disagree,
  it's often because they assumed different things.
- `information_gaps` — what the advisor didn't know. More useful than confidence
  because it's observable: "I couldn't verify whether X supports Y" beats
  "I'm 70% sure."
- `implications` — where the advice leads, especially irreversible consequences.
  Feeds the existing synthesis principle of "prefer simpler/reversible."

These fields are populated only when status is `ok`. For other statuses, omit them.

**Why normalize:** Without this, synthesis devolves into comparing apples to
oranges. Structured results make agreement and disagreement explicit, and let
the synthesizer work from a consistent shape.

Status meanings:
- `ok` — usable content returned
- `unavailable` — dependency missing or not configured (tolerable)
- `blocked` — host policy prevented execution (setup problem — see Failure Handling)
- `failed` — execution attempted but errored

### Step 4.5: Critique Round (Tier 2+ only)

At Tier 2 or above (or when `--debate` is passed), run an anonymized
cross-critique before synthesis. See [critique-round.md](critique-round.md)
for the full protocol.

In brief: anonymize each advisor's summary, feed each advisor the other two
summaries, and ask them to challenge assumptions, identify logic gaps, and
flag missing information. Attach critiques to the normalized results before
passing everything to the synthesizer.

Skip this step entirely at Tier 1.

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
prefer existing project patterns. Use the normalized fields to resolve conflicts:
- Check whether disagreeing advisors made incompatible **assumptions** — resolving
  the assumption often resolves the conflict
- Check **information gaps** — an advisor missing key context may have reached a
  different conclusion for fixable reasons
- Weight **implications** — prefer recommendations with fewer irreversible consequences

Answer these explicitly:
1. What did the advisors broadly agree on?
2. Where did they disagree?
3. Which disagreements matter vs style-level noise?
4. What decision best fits the actual project context?
5. What uncertainty remains (common information gaps across advisors)?

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

### Step 7: Persist Session

After synthesis (and review, if applicable), write the full session to
`~/.claude/magi/sessions/`. See [sessions.md](sessions.md) for the file
format and storage details.

This runs unconditionally — every magi invocation produces a session file,
regardless of tier or whether advisors were unavailable.

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
  /magi "prompt"                     # Counsel mode (default, Tier 1)
  /magi --debate "prompt"            # Force Tier 3 (critique round + review)
  /magi "Plan A vs Plan B"           # Plan synthesis mode

Examples:
  /magi "How should we implement caching for this service?"
  /magi "What's the best approach for authentication here?"
  /magi --debate "Should we use a monorepo or polyrepo for this project?"
  /magi "Compare these two migration strategies and recommend a hybrid"
```

## References

- Provider capabilities and CLI details: [reference.md](reference.md)
- Synthesis patterns and report template: [synthesis-guide.md](synthesis-guide.md)
- Anonymized critique protocol: [critique-round.md](critique-round.md)
- Session persistence format: [sessions.md](sessions.md)
- Host adapters: [hosts/claude.md](hosts/claude.md), [hosts/codex.md](hosts/codex.md)
