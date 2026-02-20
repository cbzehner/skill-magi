# Local Development Guide

How to develop and test the magi skill locally.

## Setup

### 1. Clone the Repository

```bash
git clone https://github.com/cbzehner/claude-skill-magi.git
cd claude-skill-magi
```

### 2. Install Prerequisites

The skill requires the Gemini and Codex CLIs:

```bash
# Gemini CLI
npm install -g @google/gemini-cli
gemini --login

# Codex CLI
npm install -g @openai/codex
codex login
```

### 3. Symlink for Development

Create a symlink from your skills directory to your development repo:

```bash
# Remove any existing installation
rm -rf ~/.claude/skills/magi

# Create symlink (adjust path to your clone location)
ln -s /path/to/claude-skill-magi ~/.claude/skills/magi
```

This approach means:
- Changes in your dev repo are immediately available
- No reinstallation needed after edits
- Git operations work normally in your dev directory

### 4. Verify Installation

```bash
ls -la ~/.claude/skills/magi
# Should show: magi -> /path/to/claude-skill-magi
```

## Testing

### Start Fresh

Start a new Claude Code session or use `/clear` to reset context. This ensures the skill is reloaded.

### Manual Validation Checklist

| Command | Expected Behavior |
|---------|-------------------|
| `/magi` | Shows usage examples |
| `/magi "What is 2+2?"` | Queries all three advisors + synthesis |

### Error Handling

Test graceful degradation:

1. **Missing CLI**: Uninstall gemini temporarily, run `/magi "test"`. Should explain how to install.
2. **Auth expired**: Run with expired credentials. Should suggest `gemini --login` or `codex login`.
3. **Gemini 429**: If Gemini capacity is exceeded, should retry with flash model.

## File Structure

```
claude-skill-magi/
├── SKILL.md              # Main skill - edit this for core behavior
├── reference.md          # Advisor capabilities (loaded on demand)
├── synthesis-guide.md    # Synthesis patterns (loaded on demand)
├── README.md             # Public documentation
├── docs/
│   ├── ARCHITECTURE.md   # Technical design
│   ├── DEVELOPMENT.md    # This file
│   ├── REFERENCE.md      # Detailed reference
│   └── SKILL_GUIDE.md    # Guide to building skills
└── examples/
    ├── planning.md
    ├── debugging.md
    └── research.md
```

### Which Files to Edit

| Change | File(s) |
|--------|---------|
| Routing logic | `SKILL.md` |
| CLI commands/flags | `SKILL.md`, `reference.md` |
| Synthesis behavior | `SKILL.md`, `synthesis-guide.md` |
| Advisor capabilities | `reference.md` |
| Error handling | `SKILL.md` |
| Public docs | `README.md` |

## Progressive Disclosure

The skill uses progressive disclosure - Claude loads files on demand:

1. **Always loaded**: `name` + `description` from SKILL.md frontmatter
2. **On invocation**: Full SKILL.md content
3. **On demand**: `reference.md`, `synthesis-guide.md` (when Claude follows links)

Keep `SKILL.md` lean (~100 lines). Move detailed reference material to separate files.

## Debugging Tips

### Skill Not Loading

1. Check symlink: `ls -la ~/.claude/skills/magi`
2. Verify SKILL.md frontmatter is valid YAML
3. Start fresh session (`/clear` or new terminal)

### CLI Errors

Test CLI commands directly in terminal:

```bash
# Test Gemini
gemini -p "Hello" --sandbox -o text

# Test Codex
codex exec --sandbox read-only --skip-git-repo-check -- "Hello"
```

**"Must specify GEMINI_API_KEY"**: The Gemini CLI's non-interactive mode (`-p`) does not read credentials from its encrypted storage — it only checks `process.env.GEMINI_API_KEY` (loaded via dotenv). Even if `gemini --login` or `/auth login` works interactively, you need `~/.gemini/.env` with `GEMINI_API_KEY=your-key` for subprocess/magi use.

## Contributing

1. Make changes in your local clone
2. Test using the checklist above
3. Commit and push
4. Changes are live for anyone with the symlink setup
