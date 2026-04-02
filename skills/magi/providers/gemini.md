# Gemini Provider Card

## Role

Best for current information, large-context comparative reading, API/docs
research, and outside-in framing.

## Transport

```bash
gemini -p "[prompt]" --model gemini-3.1-pro-preview --sandbox -o json
```

| Flag | Purpose |
|------|---------|
| `-p` | Non-interactive mode (required for subprocess use) |
| `--model` | `gemini-3.1-pro-preview` (max reasoning) |
| `--sandbox` | Read-only, no file writes |
| `-o json` | JSON output — includes `response` (content) and `stats` (metrics) |

**Parsing JSON output:** The response content is in the `response` field.
Token metrics are in `stats.models.<model>.tokens` (input, prompt, candidates,
thoughts, cached, total). API latency is in `stats.models.<model>.api.totalLatencyMs`.

## Setup

**Install**: `npm install -g @google/gemini-cli && gemini --login`

**Non-interactive auth**: The `-p` flag does not read the CLI's stored
credentials. You must create `~/.gemini/.env`:

```bash
echo 'GEMINI_API_KEY=your-key-here' > ~/.gemini/.env
chmod 600 ~/.gemini/.env
```

Get a key from https://aistudio.google.com/app/apikey

## Common Failure Modes

| Status | Reason | Action |
|--------|--------|--------|
| `failed` | `rate_limit` | Wait 60s, retry once, then skip |
| `unavailable` | `auth` | Show `.env` setup instructions above |
| `unavailable` | `missing_cli` | Show install command above |
| `blocked` | `permission` | Host blocks Bash execution — show host-specific setup |

## Capability Framing

Append to the advisor prompt (after the shared question):

> "You have native web search via google_web_search. Ground claims in current
> sources where relevant — cite what you find."

This references a verified built-in capability, not an aspirational persona.

## Notes

- Treat Gemini as optional when unavailable; do not block the council
- Pro preview has limited capacity — 429 errors are expected under load
- Normalize all failures into the shared result schema before synthesis
