# Critique Round

An optional deliberation step where advisors cross-critique each other's
responses before synthesis. Activated by the `--debate` flag.

## When It Runs

- **Explicit only**: user passes `--debate`
- **Default**: skipped — the fast path is query → normalize → synthesize

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
Include the `assumptions`, `information_gaps`, `implications`, and
`evidence_basis` fields — these give critiques concrete material to work with.

### 2. Prompt for Critique

Send each advisor the anonymized summaries with this prompt:

```
Two other advisors responded to the same question. Their responses are below,
anonymized. You have not seen your own response repeated here.

[Response A summary, assumptions, information_gaps, implications, evidence_basis]
[Response B summary, assumptions, information_gaps, implications, evidence_basis]

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

### 4. Feed to Synthesizer

The synthesizer receives both original responses and cross-critiques. It should
note where critiques identified flawed assumptions or reasoning gaps, and weight
those original responses accordingly.

## Transport

The critique round is a second parallel fan-out, identical in mechanics to the
original query. Each advisor gets one Bash call (external) or one Task
(host-native) with the critique prompt. On Claude Code, the orchestrator Task
handles both fan-outs internally.

## Cost

The critique round roughly doubles advisor calls (3 original + 3 critique).
Default session: ~30-60s. With `--debate`: ~60-120s.
