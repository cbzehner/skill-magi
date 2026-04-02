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

### Step 1: Build Advisor Roster

Start with the host-native local advisor, then add external advisors.

**Never double-count the host.** If you're running in Claude Code, don't launch
`claude -p` as a second advisor (causes session contention). If you're in Codex,
don't launch `codex exec` (duplicates the host). The host IS the local advisor.

See [reference.md](reference.md) for provider CLI details, setup, and failure modes.

### Step 2: Query Advisors in Parallel

**On Claude Code:** Run as a single background Task that queries all advisors and
returns the complete synthesis.

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
       - Codex: prefer plugin companion if installed (see reference.md),
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
    - What implications follow from your recommendation, especially anything hard to reverse
    - What evidence you're grounding your response in (sources, docs, code inspected, commands run)"

    **Capability framing** — append per provider:
    - Gemini: "You have native web search. Ground claims in current sources where relevant — cite what you find."
    - Codex: "You can execute shell commands in a sandboxed environment. If a factual claim is quickly verifiable by running code, verify it."
    - Claude: (no additional framing — already has project context)
```

**On Codex:** Use the current session as the local advisor. Launch external CLIs:

```bash
# Gemini (JSON for metrics — parse response field for content)
gemini -p "[advisor prompt]" --model gemini-3.1-pro-preview --sandbox -o json

# Claude (JSON for reliable error detection)
claude -p "[advisor prompt]" --output-format json
```

For Claude JSON output, check `is_error` before normalizing — `true` means auth
or other failure.

### Step 3: Normalize Results

Every advisor response becomes:

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

The four analytical fields give the synthesizer observable, verifiable inputs
rather than unreliable self-reported confidence:

- `assumptions` — what the advisor took for granted. Incompatible assumptions
  between advisors are often the root cause of apparent disagreement.
- `information_gaps` — what the advisor didn't know. "I couldn't verify X"
  beats "I'm 70% sure."
- `implications` — where the advice leads, especially irreversible consequences.
- `evidence_basis` — what grounds the response. An advisor that cites docs or
  ran code carries more weight than one reasoning from general knowledge.

Populated only when status is `ok`. Omit for other statuses.

Status meanings:
- `ok` — usable content returned
- `unavailable` — dependency missing or not configured (tolerable)
- `blocked` — host policy prevented execution (see Failure Handling)
- `failed` — execution attempted but errored

### Step 3.5: Critique Round (--debate only)

When the user passes `--debate`, run an anonymized cross-critique before
synthesis. See [critique-round.md](critique-round.md) for the full protocol.

In brief: anonymize each advisor's summary, feed each advisor the other two
summaries, ask them to challenge assumptions, identify logic gaps, and flag
missing information. Attach critiques to the normalized results before
synthesizing.

Without `--debate`, skip this step entirely. The default path is fast:
query → normalize → synthesize.

### Step 4: Synthesize

This is the step that makes magi more than fan-out plus transcript dumping.

| Pattern | When | Action |
|---|---|---|
| **Consensus** | Advisors mostly agree | Proceed, but note shared blind spots |
| **Complementary** | Different but compatible | Combine strongest non-overlapping insights |
| **Conflict** | Direct contradiction | Compare evidence quality, prefer simpler/reversible |
| **Gap** | One advisor silent | Note the gap; do not invent agreement |

**On conflicts:** Use the normalized fields to resolve:
- Check whether disagreeing advisors made incompatible **assumptions**
- Check **information gaps** — an advisor missing key context may have reached
  a different conclusion for fixable reasons
- Compare **evidence basis** — grounded claims outweigh general reasoning
- Weight **implications** — prefer recommendations with fewer irreversible consequences

Answer these explicitly:
1. What did the advisors broadly agree on?
2. Where did they disagree?
3. Which disagreements matter vs style-level noise?
4. What decision best fits the actual project context?
5. What uncertainty remains (common information gaps)?

See [reference.md](reference.md) for the report template. Label the host-native
advisor by its actual name (Claude or Codex).

### Step 5: Persist Session

Write the full session to `~/.claude/magi/sessions/YYYY-MM-DD-<slug>.yaml`.
Create the directory if needed. The slug is the first ~50 chars of the question,
lowercased, spaces to hyphens, non-alphanumeric removed.

This runs unconditionally — every magi invocation produces a session file.
Include the file path in the report so the user can find it.

Session file shape:

```yaml
question: "original prompt"
mode: "counsel|plan-synthesis"
debate: true|false
timestamp: "2026-04-02T14:30:00Z"
duration_s: 47
advisors:
  - advisor_id: "claude"
    status: "ok"
    wall_time_s: 12
    tokens: { input: 3200, output: 1800 }  # where available
    summary: "..."
    assumptions: [...]
    information_gaps: [...]
    implications: [...]
    evidence_basis: [...]
    content: "..."
    critiques_received: [...]  # only present with --debate
  # ... other advisors
synthesis:
  consensus: "..."
  conflicts: "..."
  recommendation: "..."
```

## Failure Handling

| Error Type | Action | Why |
|---|---|---|
| Permission denied | **STOP** — show setup guidance | Magi's value is multi-perspective. Single-advisor fallback is just a normal conversation |
| 429 / capacity | Wait 60s → retry → proceed without | Gemini has limited capacity |
| Auth / missing API key | Show setup instructions per [reference.md](reference.md) | |
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

(Codex hosts: show sandbox/approval guidance instead of settings.local.json.)

## Usage Examples

When `$ARGUMENTS` is empty, show:

```
Usage:
  /magi "prompt"                     # Counsel mode (default)
  /magi --debate "prompt"            # Add anonymized critique round
  /magi "Plan A vs Plan B"           # Plan synthesis mode

Examples:
  /magi "How should we implement caching for this service?"
  /magi "What's the best approach for authentication here?"
  /magi --debate "Should we use a monorepo or polyrepo for this project?"
  /magi "Compare these two migration strategies and recommend a hybrid"
```

## References

- Provider CLIs, normalization, and report template: [reference.md](reference.md)
- Anonymized critique protocol: [critique-round.md](critique-round.md)
