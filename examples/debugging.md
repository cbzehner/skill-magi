# Debugging Example: Async Function Returns Undefined

## Request

"My async function returns undefined"

## Query All Advisors

```bash
# Run in parallel (all in one message with run_in_background: true)

# Gemini
gemini -p "Error: Function returns undefined
Code: const data = response.json(); return data;
Provide: 1) Cause 2) Investigation 3) Fix 4) Prevention" --sandbox -o text

# Codex
codex exec --sandbox read-only --skip-git-repo-check -- "Error: Function returns undefined
Code: const data = response.json(); return data;
Provide: 1) Cause 2) Investigation 3) Fix 4) Prevention"

# Claude via Task subagent
Task:
  subagent_type: "general-purpose"
  model: "opus"
  prompt: "You are a senior software architect advisor.
           Error: Function returns undefined
           Code: const data = response.json(); return data;
           Provide: 1) Cause 2) Investigation 3) Fix 4) Prevention"
```

## Advisor Responses

**Gemini**:
- Missing `await` on `response.json()`
- `response.json()` returns a Promise
- Add `await` keyword

**Codex**:
- Missing `await` - returning Promise instead of resolved value
- Ensure parent function is `async`
- Consider using `.then()` chain as alternative

**Claude**:
- Missing `await` on async operation
- Also check if `response.ok` before parsing
- Add try/catch for JSON parse errors

## Synthesis

**Pattern**: Consensus - all agree on root cause

**Decision**: Add `await` and improve error handling

**Reasoning**: All three identified the same issue. Claude's additional suggestions for error handling are valuable additions.

## Result

```javascript
// Before
const data = response.json();
return data;

// After
if (!response.ok) {
  throw new Error(`HTTP ${response.status}`);
}
const data = await response.json();
return data;
```
