# Magi Architecture

## Overview

Magi queries three AI advisors (Gemini, Codex, Claude) and synthesizes their responses. It uses direct CLI invocations documented in SKILL.md—no shell scripts.

## Full Counsel Mode

Full counsel runs as a **single background Task** that handles all advisor queries and synthesis internally.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ORCHESTRATOR (Main Claude Code)                          │
│                                                                                  │
│  Receives /magi "prompt", spawns single background Task                         │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ Task (background)
                                      │ subagent_type: general-purpose
                                      │ model: opus
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           MAGI EXECUTOR (Task Subagent)                          │
│                                                                                  │
│  Queries all 3 advisors, waits for all, synthesizes, returns complete result   │
└─────────────────────────────────────────────────────────────────────────────────┘
           │                          │                          │
           │ Bash                     │ Bash                     │ (internal)
           ▼                          ▼                          ▼
┌───────────────────┐      ┌───────────────────┐      ┌───────────────────┐
│      GEMINI       │      │      CODEX        │      │      CLAUDE       │
│  gemini CLI       │      │  codex CLI        │      │  The subagent     │
│  --sandbox        │      │  --sandbox        │      │  itself provides  │
│  -o text          │      │  read-only        │      │  Claude's view    │
└───────────────────┘      └───────────────────┘      └───────────────────┘
           │                          │                          │
           └──────────────────────────┼──────────────────────────┘
                                      ▼
                              Complete synthesis
                              returned to orchestrator
```

The single-Task approach ensures the main thread only receives the complete synthesis, not partial results.

## Why Task Subagent for Claude?

Running `claude -p` as a subprocess causes session contention and hangs. Task subagents avoid this—the subagent IS Claude, no subprocess needed.

## Model Selection

| Advisor | Method | Model |
|---------|--------|-------|
| Gemini | Bash → CLI | Gemini's default |
| Codex | Bash → CLI | gpt-5.3-codex |
| Claude | Task subagent | opus |

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill instructions, routing, CLI patterns |
| `reference.md` | Advisor capabilities, selection guide |
| `synthesis-guide.md` | Synthesis patterns, report template |
| `docs/ARCHITECTURE.md` | This file |
| `docs/DEVELOPMENT.md` | Local development guide |
| `docs/REFERENCE.md` | Detailed reference |
