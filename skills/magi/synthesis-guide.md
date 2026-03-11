# Synthesis Guide

Patterns for combining advisor responses. Loaded on demand.

---

## Synthesis Patterns

| Pattern | When | Action |
|---------|------|--------|
| **Consensus** | All agree | Proceed with confidence; watch for shared blind spots |
| **Complementary** | Different but compatible | Combine: Gemini's research + Codex's patterns + Claude's architecture |
| **Conflict** | Direct contradiction | Evaluate evidence quality, apply project context, prefer simpler/reversible |
| **Gap** | One advisor silent | Note gap, query specifically if critical, or mark as assumption |

**On conflicts**: Good evidence = specific references, reasoned arguments. Weak evidence = vague claims, false confidence. When in doubt, prefer existing project patterns.

---

## Report Template

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
<summary>Claude Response</summary>

[Full response]

</details>

## Synthesis
| Advisor | Key Insight |
|---------|-------------|
| Gemini | ... |
| Codex | ... |
| Claude | ... |

**Consensus**: [What they agreed on]
**Conflicts**: [Disagreements and resolution]
**Recommendation**: [Your synthesized advice]
```

Simple queries may only need Quick Answer + Synthesis table.
