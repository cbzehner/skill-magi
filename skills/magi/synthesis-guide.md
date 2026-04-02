# Synthesis Guide

Patterns for combining advisor responses and the standard report template.

---

## Why Synthesis Exists

Without synthesis, magi is just fan-out plus transcript dumping. The synthesis
step compresses multiple advisor outputs into one decision artifact, makes
agreement and disagreement explicit, and creates a stable object for review.

## Synthesis Patterns

| Pattern | When | Action |
|---------|------|--------|
| **Consensus** | Advisors mostly agree | Proceed with confidence; watch for shared blind spots |
| **Complementary** | Different but compatible | Combine deliberately: e.g. Gemini's research + Codex's patterns + Claude's architecture |
| **Conflict** | Direct contradiction | Evaluate evidence quality, apply project context, prefer simpler/reversible |
| **Gap** | One advisor silent | Note gap; do not invent agreement |

**On conflicts**: Good evidence = specific references, reasoned arguments,
testable claims. Weak evidence = vague confidence without support. When in
doubt, prefer existing project patterns.

## Review Separation

The reviewer should inspect the synthesized artifact for:
- false consensus
- dropped caveats
- overweighting one advisor without evidence
- recommendations that are hard to reverse

Review is a separate step from synthesis. Keep them apart.

---

## Report Template

Replace `[Host Advisor]` below with the actual host name — Claude or Codex —
so the user sees which model said what.

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

Tier: [1/2/3] [escalation reason if not Tier 1]

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
```

**Metrics availability by provider:**
- **Gemini** (`-o json`): `stats.models.*.tokens` for input/output/thoughts,
  `stats.models.*.api.totalLatencyMs` for API latency. Wall time measured by
  orchestrator.
- **Codex** (`exec`): aggregate token count only (single number, no in/out split).
  No timing from CLI — wall time measured by orchestrator.
- **Claude** (host-native Task): token counts and cost available from task metadata
  when running via `-p --output-format json`. From Agent/Task tool, usage may be
  reported in task output.

Use `—` for any metric a provider doesn't report. Never estimate or fabricate
metrics.

Simple queries may only need Quick Answer + Session Metrics + Synthesis table
(skip the collapsible individual responses).
