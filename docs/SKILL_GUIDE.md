# Building Excellent Claude Code Skills

Patterns and insights for creating effective skills, learned from building magi and studying Anthropic's official documentation.

> For comprehensive documentation, see [Anthropic's Skills Guide](https://code.claude.com/docs/en/skills) and the [Agent Skills Engineering Blog](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills).

---

## Core Principle: Progressive Disclosure

Skills work best when they load information **on demand**, not all at once.

```
Level 1: name + description     → Always in context (decides if skill applies)
Level 2: SKILL.md content       → Loaded when skill is invoked
Level 3: Referenced files       → Loaded when Claude follows links
```

### What Stays in SKILL.md

Keep **operational content** in SKILL.md - things Claude needs to execute immediately:

- Tool invocation patterns (Bash commands, Task configurations)
- Command routing logic
- Error handling rules
- Critical workflows

### What Goes in Reference Files

Move **informational content** to separate files:

- Detailed capabilities tables
- Selection matrices ("when to use X vs Y")
- Template libraries
- Background context

**Key insight**: Don't move execution mechanics to reference files. If Claude needs to read another file before it can act, you've added latency and token overhead.

```markdown
# Good: SKILL.md has the command
Bash: gemini -p "[prompt]" --sandbox -o text

# Bad: SKILL.md references another file for the command
See [reference.md](reference.md) for invocation syntax.
```

---

## Scripts vs Markdown-Only

**Use scripts when you need:**
- Deterministic operations (sorting, file manipulation, validation)
- Complex multi-step workflows that would be error-prone in prose
- Operations requiring guaranteed consistency
- Cost optimization (avoiding token generation for algorithmic tasks)

**Use markdown-only when:**
- Scripts are thin wrappers around CLI tools
- The "validation" is just checking if arguments exist
- Claude can interpret errors directly from tool output
- Portability matters (avoid GNU vs BSD differences)

### Decision Framework

Ask: "Does this script do anything Claude couldn't do if I just told it the command?"

If the script just:
- Validates input → Claude sees errors anyway
- Formats output → Nice-to-have, not essential
- Wraps a CLI → Just document the CLI

Then consider markdown-only.

**Example**: A script that calls `gemini -p "$PROMPT" --sandbox` with input validation adds complexity without meaningful value. Claude can run the command directly and interpret any errors.

---

## File Structure Patterns

### Minimal Skill (Most Skills)
```
my-skill/
└── SKILL.md              # Everything in one file
```

### Medium Skill (Some Reference Material)
```
my-skill/
├── SKILL.md              # Core instructions (~80-100 lines)
└── reference.md          # Detailed reference (loaded on demand)
```

### Complex Skill (Multiple Concerns)
```
my-skill/
├── SKILL.md              # Core instructions
├── reference.md          # Capabilities, selection guides
├── templates.md          # Response templates, formats
├── docs/                 # External docs for GitHub (not loaded)
│   └── ARCHITECTURE.md
└── examples/             # Usage examples (not loaded)
    └── example.md
```

### Keep `docs/` for Humans

Files in `docs/` are for GitHub viewers, not loaded by the skill. Use this for:
- Architecture explanations
- Contribution guides
- Detailed documentation that exceeds what Claude needs

---

## Writing Effective Descriptions

The `description` field in frontmatter is critical - it's how Claude decides whether to load your skill.

### Include Trigger Words

```yaml
# Good: Multiple trigger scenarios
description: Query AI advisors for planning, debugging, research,
  finalizing plans, reviewing code, or getting alternative perspectives.

# Bad: Vague
description: A helpful skill for various tasks.
```

### Include Invocation Hints

```yaml
# Good: Shows how to use it
description: Multi-AI counsel system. Query Gemini, Codex, Claude
  advisors together with synthesis. Use /magi "prompt" to invoke.

# Bad: No usage hint
description: Queries multiple AI advisors.
```

### Match Natural Language

Think about how users will ask for help:
- "Help me plan..." → include "planning"
- "Debug this error..." → include "debugging"
- "What do you think about..." → include "alternative perspectives"

---

## Command Routing Patterns

If your skill has multiple modes, define explicit routing rules.

### Simple Routing

For skills with a single mode, keep it simple:

```markdown
## Usage

If ARGUMENTS is empty, show usage examples. Otherwise, execute with the prompt.
```

### Multi-Mode Routing

For skills with multiple modes, use first-token matching:

```markdown
## Command Routing

Parse ARGUMENTS:
1. Extract first whitespace-delimited token
2. If token matches a known mode (case-insensitive), route there
3. Otherwise: Default mode, entire ARGUMENTS is the prompt
4. If empty: Show usage examples
```

Include examples to prevent ambiguity.

---

## Error Handling Strategies

### Graceful Degradation

Define what happens when components fail:

```markdown
| Available | Action |
|-----------|--------|
| 3/3 | Full operation |
| 2/3 | Partial operation, note what's missing |
| 1/3 | Minimal operation, offer to retry |
| 0/3 | Explain failure, suggest fixes |
```

### Let Claude Interpret Errors

Instead of complex error classification in scripts:

```markdown
### Error Handling
- If command fails with network error, retry once
- If auth error (mentions "login" or "credentials"), suggest fix
- Don't retry auth failures
```

Claude can interpret error messages and act appropriately.

### Missing Dependencies

Don't require pre-flight checks. If a CLI isn't installed:

```markdown
If `gemini` command not found, explain:
"Gemini CLI not installed. Run: npm install -g @google/gemini-cli && gemini --login"
```

---

## Testing Your Skill

### Validation Checklist

Before releasing:

- [ ] SKILL.md under recommended length (~100 lines for complex skills)
- [ ] All command modes work as documented
- [ ] Empty input shows helpful usage
- [ ] Ambiguous inputs route correctly
- [ ] Failures degrade gracefully
- [ ] Error messages include fix suggestions
- [ ] Links to reference files work

### Edge Cases to Test

- Empty arguments
- Arguments containing command keywords ("use gemini for...")
- Very long prompts
- Special characters in prompts
- Missing dependencies
- Network failures
- Auth failures

---

## Common Mistakes

### Over-Engineering

**Mistake**: Creating scripts for every operation
**Fix**: Ask if Claude could just run the command directly

### Under-Documenting Routing

**Mistake**: "Parse the input to determine mode"
**Fix**: Explicit rules with examples and non-examples

### Stuffing SKILL.md

**Mistake**: 300+ line SKILL.md with everything
**Fix**: Progressive disclosure - move reference material to separate files

### Hiding Execution Details

**Mistake**: Moving CLI commands to reference files
**Fix**: Keep operational content in SKILL.md, move informational content out

### Rigid Templates

**Mistake**: "Always use this exact 7-section format"
**Fix**: "Use appropriate sections based on complexity"

---

## Resources

- [Anthropic Skills Documentation](https://code.claude.com/docs/en/skills) - Official comprehensive guide
- [Anthropic Subagents Documentation](https://code.claude.com/docs/en/sub-agents) - For skills that spawn agents
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) - General agentic coding patterns
- [Agent Skills Engineering Blog](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) - Design philosophy
- [anthropics/skills Repository](https://github.com/anthropics/skills) - Official examples
