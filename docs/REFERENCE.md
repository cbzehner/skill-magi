# Magi Reference

Complete reference for tool capabilities, prompt templates, synthesis patterns, and disagreement handling.

## Contents
- [Tool Capabilities](#tool-capabilities)
- [Prompt Templates](#prompt-templates)
- [Synthesis Patterns](#synthesis-patterns)
- [Disagreement Handling](#disagreement-handling)

---

## Tool Capabilities

### Execution Methods

See [ARCHITECTURE.md](ARCHITECTURE.md) for full details.

| Advisor | Method | Why |
|---------|--------|-----|
| Gemini | Bash → CLI | External CLI tool |
| Codex | Bash → CLI | External CLI tool |
| Claude | Task subagent (opus) | IS Claude, no subprocess needed |

### Gemini CLI

**Strengths**: Web search (`google_web_search`), 1M token context, `codebase_investigator`

```bash
gemini -p "[prompt]" --sandbox -o text
```

| Flag | Purpose |
|------|---------|
| `-p` / `--prompt` | Non-interactive mode (required for headless/subprocess use) |
| `--sandbox` | Read-only mode, no file writes |
| `-o` | Output: text, json, stream-json |
| `-m` | Model: flash, pro, flash-lite |

**Rate limits (free)**: 60/min, 1000/day

### Codex CLI

**Strengths**: Fast (Rust), fine-grained security, multi-provider

```bash
codex exec --sandbox read-only --skip-git-repo-check -- "[prompt]"
```

| Flag | Purpose |
|------|---------|
| `exec` | Non-interactive mode |
| `--sandbox` | read-only, workspace-write, danger-full-access |
| `--skip-git-repo-check` | Run outside git repositories |
| `--model` | Model override |

### Claude (Task Subagent)

**Strengths**: Deep reasoning (Opus model), code expertise, codebase awareness

**Method**: Task subagent with `model: "opus"`. The subagent IS Claude—it responds directly without spawning a process or calling an API.

```
Task:
  subagent_type: "general-purpose"
  model: "opus"
  prompt: "You are a senior software architect advisor. [prompt]"
```

**Why not CLI?** Running `claude -p` as a subprocess from within Claude Code hangs due to session contention. Task subagents avoid this entirely.

### Selection Matrix

| Task | Use | Why |
|------|-----|-----|
| Web/API docs | Gemini | google_web_search |
| Current info | Gemini | Web access |
| Library research | Gemini | Can compare options |
| Fast analysis | Codex | Speed |
| Large context | Gemini | 1M tokens |
| Security ops | Codex | Sandboxing |
| CI/CD | Codex | Automation focus |
| Deep reasoning | Claude (opus) | Best reasoning model |
| Code review | Claude (opus) | Code expertise |
| Architecture | Claude (opus) | Reasoning depth |

### Error Handling

If a CLI fails, Claude interprets the error and responds appropriately:

| Issue | Fix |
|-------|-----|
| Rate limit | Wait, use different model |
| Auth expired | `gemini --login` or `codex login` |
| CLI not found | Install per error message |

---

## Prompt Templates

### Planning

```
Task: {description}
Context: {codebase context}
Constraints: {limitations}

Provide:
1. APPROACH: Recommended approach (2-3 sentences)
2. STEPS: Implementation steps
3. RISKS: Challenges
4. ALTERNATIVES: Other approaches
```

### Research

```
Topic: {question}
Context: {why needed}

Provide:
1. KEY FINDINGS: Main information
2. SOURCES: Where from
3. APPLICABILITY: How it applies
4. CAVEATS: Limitations
```

### Debugging

```
Error: {message}
Code: {relevant code}
Context: {what was attempted}

Provide:
1. LIKELY CAUSE: Explanation
2. INVESTIGATION: Confirm steps
3. FIX: Solution with code
4. PREVENTION: Future prevention
```

### Code Review

```
Code: {code}
Intent: {what it should do}

Provide:
1. CORRECTNESS: Works as intended?
2. ISSUES: Bugs or edge cases
3. IMPROVEMENTS: Suggested changes
4. STYLE: Best practice alignment
```

### Architecture

```
Decision: {what to decide}
Current: {existing architecture}
Requirements: {must achieve}

Provide:
1. RECOMMENDATION: Approach
2. TRADE-OFFS: Pros/cons
3. ALTERNATIVES: Other options
4. MIGRATION: Implementation steps
```

### Best Practices

1. Be specific (file paths, error messages)
2. Provide context (what's been tried)
3. Request structure (numbered lists)
4. Set scope (overview vs detail)

---

## Synthesis Patterns

### Consensus

**When**: All advisors agree.

**Action**: Proceed with confidence, but verify critical assumptions.

**Watch for**: Shared blind spots, outdated practices.

### Complementary

**When**: Different but compatible perspectives.

**How to combine**:
1. Gemini's research for background
2. Codex's patterns for implementation
3. Merge leveraging both

**Example**:
- Gemini: "OWASP recommends bcrypt"
- Codex: "Project uses argon2"
- Synthesis: "Use argon2 (meets OWASP standards, matches existing pattern)"

### Conflict

**When**: Direct contradiction.

**Resolution**:
1. Categorize (factual, architectural, stylistic, risk)
2. Evaluate evidence quality
3. Apply project context
4. Decide: simpler > complex, reversible > permanent

### Gap Filling

**When**: One advisor silent.

**Options**:
1. Query other advisor specifically
2. Use own knowledge
3. Web search (Gemini)
4. Mark as assumption

### Synthesis Checklist

Before finalizing:
- [ ] Understood all perspectives?
- [ ] Identified agreements/conflicts?
- [ ] Applied own analysis?
- [ ] Final plan better than any single advisor?
- [ ] Can explain reasoning?

### Communication Format

```
Gemini: [key points]
Codex: [key points]
Claude: [key points]
Synthesis: [recommendation]
Reasoning: [why, noting agreements/conflicts]
```

---

## Disagreement Handling

### Decision Framework

#### 1. Categorize

| Type | Example | Strategy |
|------|---------|----------|
| Factual | API syntax | Verify docs |
| Architectural | Pattern choice | Consider constraints |
| Stylistic | Naming | Follow project |
| Risk | Security level | Evaluate evidence |

#### 2. Evaluate Evidence

**Good evidence**: Specific references, reasoned arguments, trade-off consideration, uncertainty acknowledgment

**Weak evidence**: Vague claims, assertions without reasoning, single perspective, false confidence

#### 3. Apply Project Context

- Existing patterns and conventions
- Team expertise
- Performance requirements
- Maintenance considerations

#### 4. Decide

- Prefer simpler when outcomes similar
- Prefer reversible when uncertain
- Prefer established patterns

#### 5. Communicate

```
Gemini suggested: [approach] because [reasoning]
Codex suggested: [approach] because [reasoning]
Claude suggested: [approach] because [reasoning]
I recommend: [decision] because [synthesis]
```

### Common Scenarios

**Library choice**: Prefer existing dependencies unless compelling reason to add new ones.

**Architecture**: Consider actual scale. Don't over-engineer.

**Security**: When in doubt, prefer more secure approach.

**Performance**: Measure before optimizing.

### Anti-Patterns

- **False consensus**: Agreement does not equal correctness
- **Authority bias**: Preferring one advisor without reason
- **Analysis paralysis**: Over-deliberating minor decisions
- **Ignoring context**: Applying generic advice to specific situation
