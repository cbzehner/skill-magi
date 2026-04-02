# Critique Round

An optional deliberation step where advisors cross-critique each other's
responses before synthesis. Borrowed from llm-council's peer review stage,
adapted with Paul-Elder-aligned critique prompts.

## When It Runs

- **Explicit**: user passes `--debate` flag
- **Auto-escalation**: Tier 2 or Tier 3 (see escalation rules in SKILL.md)
- **Never**: Tier 1 (Quick Counsel) — skip entirely

## How It Works

### 1. Anonymize

Strip advisor names from the normalized summaries. Assign randomized labels
to prevent position bias:

```python
import random
labels = ["Response A", "Response B", "Response C"]
random.shuffle(labels)  # randomize assignment order
```

Each advisor sees the summaries of the OTHER two responses, not its own.
Include the `assumptions`, `information_gaps`, and `implications` fields
from normalization — these give critiques concrete material to work with.

### 2. Prompt for Critique

Send each advisor the anonymized summaries with this prompt:

```
Two other advisors responded to the same question. Their responses are below,
anonymized. You have not seen your own response repeated here.

[Response A summary, assumptions, information_gaps, implications]
[Response B summary, assumptions, information_gaps, implications]

Critique these responses:
1. **Assumption challenges**: What assumptions do these responses make that
   may not hold? Why?
2. **Logic challenges**: Where do the inferences not follow from the evidence
   given? What reasoning gaps exist?
3. **Missing information**: What relevant information or perspectives are
   missing from both responses?
```

### 3. Collect and Attach

Each advisor's critique is attached to the normalized results:

```yaml
critiques_received:
  - from: "anonymous"
    assumption_challenges: "..."
    logic_challenges: "..."
    missing_information: "..."
  - from: "anonymous"
    assumption_challenges: "..."
    logic_challenges: "..."
    missing_information: "..."
```

The `from` field stays "anonymous" in the data structure. The synthesizer
does not need to know which advisor critiqued which — it only needs the
substance.

### 4. Feed to Synthesizer

The synthesizer now receives:
- Original normalized responses (with assumptions/gaps/implications)
- Cross-critiques (with assumption challenges/logic challenges/missing info)

This is strictly richer input than synthesis without the critique round.
The synthesizer should note where critiques successfully identified flawed
assumptions or reasoning gaps, and weight those original responses accordingly.

## Transport

The critique round is a second parallel fan-out, identical in mechanics to
the original query (Step 3). Each advisor gets one Bash call (external) or
one Task (host-native) with the critique prompt.

On Claude Code: the orchestrator Task handles this internally — it runs the
first fan-out, collects and anonymizes, then runs the second fan-out, then
synthesizes. All within the same background Task.

## Cost

The critique round roughly doubles the total advisor calls (3 original + 3
critique). For a typical counsel session:
- Tier 1: 3 calls, ~30-60s
- Tier 2: 6 calls, ~60-90s
- Tier 3: 6 calls + review, ~90-120s

This is why the critique round is optional and gated by escalation rules.
