# Session Persistence

After synthesis, persist the full counsel session as a structured YAML file
for auditability and cross-run comparison.

## Storage Location

```
~/.claude/magi/sessions/YYYY-MM-DD-<slug>.yaml
```

The slug is derived from the first ~50 characters of the question, lowercased,
with spaces replaced by hyphens and non-alphanumeric characters removed.

Create the directory if it doesn't exist.

## Session File Shape

```yaml
question: "original prompt"
mode: "counsel|plan-synthesis"
tier: 1|2|3
tier_reason: "default|pre-query escalation|post-normalization escalation|--debate"
timestamp: "2026-04-01T14:30:00Z"
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
    content: "..."
    critiques_received: [...]  # only present at Tier 2+
  - advisor_id: "gemini"
    status: "ok"
    wall_time_s: 31
    tokens: { input: 8677, output: 152 }
    raw_stats: { ... }  # full provider-specific metrics blob
    summary: "..."
    assumptions: [...]
    information_gaps: [...]
    implications: [...]
    content: "..."
    critiques_received: [...]
  - advisor_id: "codex"
    status: "ok"
    wall_time_s: 28
    tokens: { total: 9854 }  # codex reports aggregate only
    summary: "..."
    assumptions: [...]
    information_gaps: [...]
    implications: [...]
    content: "..."
    critiques_received: [...]
synthesis:
  consensus: "..."
  conflicts: "..."
  recommendation: "..."
review: "..."  # only present at Tier 3
```

## What Gets Stored

- Full normalized results including all analytical fields
- Raw provider metrics (stored in `raw_stats` per advisor for future use)
- Tier selection and reason
- Critique round results when applicable
- Review results when applicable
- Total wall time

## What Does NOT Get Stored

- The rendered markdown report (that's in the conversation)
- Provider API keys or auth tokens
- Any user-identifying information beyond what's in the question

## Usage

Sessions are write-only by default. The skill writes them after every
invocation but does not read them back automatically.

Future use cases (not implemented now):
- `grep` across sessions to find past counsel on a topic
- Diff two sessions to see how advice changed over time
- Reference a past session as input to a new counsel
