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

Simple queries may only need Quick Answer + Synthesis table.
