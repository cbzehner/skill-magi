# Magi Architecture

## Overview

Magi queries AI advisors (Gemini, Codex, Claude) and synthesizes their responses.
The workflow is host-neutral — the same protocol runs in Claude Code and Codex,
with host adapters supplying execution details.

## File Structure

```text
SKILL.md                    # Main skill — self-contained happy path
reference.md                # Advisor capabilities, CLI details, setup
synthesis-guide.md          # Synthesis patterns, report template
hosts/claude.md             # Claude Code host adapter
hosts/codex.md              # Codex host adapter
providers/gemini.md         # Gemini provider card
providers/codex.md          # Codex provider card
providers/claude.md         # Claude provider card
```

SKILL.md is self-sufficient for the common case. Reference files add depth.

## Execution Model

1. Identify host (Claude Code or Codex)
2. Build advisor roster (host-native first, then external)
3. Query advisors in parallel
4. Normalize results to shared schema
5. Synthesize agreement and disagreement
6. Review when high-stakes or conflicted
7. Return concise recommendation with inspectable evidence

## Host Adapters

### Claude Code

- Local advisor: Task subagent (model: opus)
- External advisors: Gemini CLI, Codex CLI via Bash
- Concurrency: single background Task handles all queries
- Setup: `.claude/settings.local.json` permissions

### Codex

- Local advisor: current Codex session (not `codex exec`)
- External advisors: Gemini CLI when sandbox/network allows
- Concurrency: sequential if parallel isn't safe
- Setup: Codex approval/sandbox policy

## Design Rules

- Never double-count the host-native provider
- Keep synthesis separate from review
- Normalize failures before rendering output
- Permission blocked = STOP, not degrade
