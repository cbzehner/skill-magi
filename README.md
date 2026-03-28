# Magi

Query multiple AI advisors (Gemini, Codex, Claude) in parallel and synthesize their responses into unified recommendations.

> *Three minds. One synthesis.*
>
> Named for the biblical wise men who offered counsel—and for fans of giant
> robots, NERV's [MAGI system](https://evangelion.fandom.com/wiki/MAGI): three
> computers with distinct perspectives that vote on critical decisions.

## Why Use This?

- **Multiple perspectives**: Get advice from Gemini (web search, 1M context), Codex (fast, sandboxed), and Claude (deep reasoning)
- **Automatic synthesis**: Identifies consensus, complements, and conflicts across advisors
- **Better decisions**: Cross-reference recommendations to catch blind spots

## Prerequisites

You must have the Gemini and Codex CLIs installed and authenticated:

```bash
# Gemini CLI
npm install -g @google/gemini-cli
gemini --login

# Codex CLI
npm install -g @openai/codex
codex login
```

> **Note:** Magi runs Gemini in non-interactive mode, which requires the API key
> as an environment variable. If you authenticated via `gemini --login` or `/auth login`,
> you also need to create `~/.gemini/.env`:
>
> ```bash
> echo 'GEMINI_API_KEY=your-key-here' > ~/.gemini/.env
> chmod 600 ~/.gemini/.env
> ```
>
> Get your key from [AI Studio](https://aistudio.google.com/app/apikey).

## Installation

### From Marketplace

```bash
# Add the marketplace
/plugin marketplace add cbzehner/skill-magi

# Install the skill
/plugin install magi@cbzehner
```

### Manual Installation

Clone into your `.claude/skills/` directory:

```bash
cd ~/.claude/skills/
git clone https://github.com/cbzehner/skill-magi.git magi
```

## Usage

```
/magi "prompt"    # Query all three advisors + synthesis
```

### Example

```
You: /magi "How should we implement authentication for my app?"

Claude: [Queries all 3 advisors in parallel, synthesizes responses]

Based on input from Gemini, Codex, and Claude:

**Consensus**: All recommend JWT with OAuth providers
**Gemini**: Google/GitHub OAuth integration (found current best practices)
**Codex**: jsonwebtoken + passport middleware pattern
**Claude**: Consider XSS risks, use httpOnly cookies

Recommended approach: Use passport.js with JWT stored in httpOnly
cookies. Implement Google OAuth first, add GitHub later...
```

The skill also triggers automatically when you mention:
- "plan", "debug", "research"
- "alternative perspective"
- "ask Gemini", "ask Codex", "ask Claude"

## How It Works

Full counsel mode runs as a single background Task that queries all advisors:

```
Orchestrator (Claude Code)
     |
     +-- Task (background) --> Magi Executor
                                    |
                                    +-- Bash --> gemini CLI
                                    +-- Bash --> codex CLI
                                    +-- (Claude reasoning)
                                    |
                                    +-- Synthesize --> Complete result
```

This ensures you receive the complete synthesis, not partial results. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for details.

## Advisor Strengths

| Advisor | Best For | Why |
|---------|----------|-----|
| Gemini | Web/API docs, current info | Web search, 1M token context |
| Codex | Fast analysis, security | Speed, sandboxing |
| Claude | Architecture, code review | Deep reasoning (Opus) |

## Files

```
magi/
├── SKILL.md              # Main skill definition
├── reference.md          # Advisor capabilities (loaded on demand)
├── synthesis-guide.md    # Synthesis patterns (loaded on demand)
├── README.md             # This file
├── LICENSE               # MIT
├── docs/
│   ├── ARCHITECTURE.md   # Technical design
│   ├── DEVELOPMENT.md    # Local development guide
│   ├── REFERENCE.md      # Detailed reference
│   └── SKILL_GUIDE.md    # Guide to building skills
└── examples/
    ├── planning.md       # Feature planning example
    ├── debugging.md      # Error debugging example
    └── research.md       # Library research example
```

## Privacy & Cost

**Data sent**: Your prompts are sent to Google (Gemini), OpenAI (Codex), and Anthropic (Claude) APIs.

**Cost**: Each query invokes 3 AI models. Monitor your API usage accordingly. To query just one model, call the CLI directly: `gemini -p "prompt" --sandbox -o text` or `codex exec --sandbox read-only -- "prompt"`.

## License

MIT
