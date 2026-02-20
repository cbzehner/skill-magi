# Planning Example: Add JWT Authentication

## Request

"Add user authentication with JWT tokens"

## Query All Advisors

```bash
# Run in parallel (all in one message with run_in_background: true)

# Gemini
gemini -p "Task: Add JWT auth to Express API
Context: Express backend, PostgreSQL, no existing auth
Provide: 1) Approach 2) Steps 3) Risks 4) Alternatives" --sandbox -o text

# Codex
codex exec --sandbox read-only --skip-git-repo-check -- "Task: Add JWT auth to Express API
Context: Express backend, PostgreSQL, no existing auth
Provide: 1) Approach 2) Steps 3) Risks 4) Alternatives"

# Claude via Task subagent
Task:
  subagent_type: "general-purpose"
  model: "opus"
  prompt: "You are a senior software architect advisor.
           Task: Add JWT auth to Express API
           Context: Express backend, PostgreSQL, no existing auth
           Provide: 1) Approach 2) Steps 3) Risks 4) Alternatives"
```

## Advisor Responses

**Gemini**:
- OAuth 2.0 with PKCE recommended
- Use passport.js for strategy abstraction
- OWASP guidelines for token storage

**Codex**:
- Simple JWT + refresh token pattern
- Custom middleware (avoid passport dependency)
- Store tokens in httpOnly cookies

**Claude**:
- Consider XSS/CSRF attack vectors
- Use httpOnly + SameSite cookies
- Implement token rotation

## Synthesis

**Pattern**: Complementary - different but compatible advice

**Decision**: Custom middleware (matches existing patterns) + OWASP guidelines (security) + httpOnly cookies (Claude's security advice)

**Reasoning**: Codex's simpler approach fits the codebase, but we incorporate Gemini's OWASP research and Claude's security considerations. Passport.js adds unnecessary abstraction for a single auth strategy.

## Result

Implemented:
1. Custom JWT middleware with refresh tokens
2. httpOnly + SameSite cookies for token storage
3. Token rotation on refresh
4. Rate limiting on auth endpoints
