---
name: insights
description: Extract non-obvious key findings from the current session and persist them as project rules. Reviews the full conversation context for hard-won insights discovered through trial and error. Only writes when genuine insights exist — otherwise reports back with no changes.
---

# Insights

Extract hard-won, non-obvious key findings from the current conversation and persist them into the project's `INSIGHTS.md` so future agents start with an upper hand.

## When to Use

- The user asks to "capture insights", "save findings", or "record what we learned"
- At the end of a long debugging or implementation session
- Before compacting a conversation, to preserve non-obvious discoveries
- When the user suspects future sessions will stumble on the same traps

## What Counts as an Insight

**NOT insights** (do NOT record):

- Things that are obvious from reading the code or docs
- Standard patterns anyone would know (e.g., "run tests before committing")
- Information already captured in AGENTS.md, README, or existing INSIGHTS.md
- Basic syntax or API usage that's self-documenting

**YES insights** (DO record):

- Non-obvious gotchas discovered through trial and error
- Unexpected behaviors that contradicted initial assumptions
- Hidden dependencies or coupling that isn't visible from the API surface
- Workarounds for platform-specific bugs or limitations
- Things that look correct but silently fail (e.g., shadowing, race conditions)
- Configuration pitfalls that produce no error but wrong behavior
- Architecture decisions and the reasoning behind them (the "why", not the "what")
- Debugging paths that wasted time (so the next agent skips them)

## Process

1. **Review the entire conversation** from start to current point. Pay special attention to:
   - Bugs that required multiple attempts to fix
   - Assumptions that turned out wrong
   - Code that looks right but doesn't work
   - Cross-platform inconsistencies discovered
   - Implicit coupling or hidden side effects
   - Things that compiled/ran but produced wrong results silently

2. **Evaluate findings** against the criteria above. Filter ruthlessly:
   - Is this already documented somewhere? → Skip
   - Would any competent developer figure this out in 5 minutes? → Skip
   - Did we discover this through trial and error (not just reading)? → Keep
   - Would this save significant time if known upfront? → Keep

3. **Check existing INSIGHTS.md** if it exists. Read it. Deduplicate — don't re-state what's already there, but do update stale entries if new information supersedes them.

4. **If no genuine insights exist**, inform the user:

   > No key insights found in this session. Nothing was discovered through trial and error that wouldn't be obvious from reading the codebase. Not creating/updating INSIGHTS.md.

   Then stop. Do not create or modify INSIGHTS.md.

5. **If insights exist**, write them to `INSIGHTS.md` at the project root. Use the format described in the Writing Rules section below.

6. **Auto-commit** the INSIGHTS.md change to the current branch:

   ```bash
   cd /path/to/project/root
   git add INSIGHTS.md
   git commit -m "docs(insights): add key insights from [brief description of session]"
   ```

   - Only `INSIGHTS.md` should be staged — do not stage other modified files
   - The commit message should include a short description of the session (e.g., "deal-card-like-dislike-backend", "scraper-config-debugging")
   - Run from the project root directory containing `INSIGHTS.md`

## Writing Rules

- **Be specific**: Reference actual file paths, class names, line numbers, or config keys. No vague statements.
- **Be concise**: Each insight should be digestible in under 30 seconds. No narratives.
- **Use bold for the finding**, then brief context/impact below it.
- **Group by category** (e.g., "Codegen", "Cross-Platform", "WebView", "Registry Pattern").
- **No filler**: Don't pad to make it look substantial. 2 real insights > 10 obvious ones.
- **Date-tag sections** with `<!-- Updated: YYYY-MM-DD -->` so staleness is visible.
- **Append, don't replace**: Add new insights. Only remove/update existing ones if new information supersedes them.

## Example Output

```markdown
# Key Insights

<!-- Updated: 2025-06-16 -->

## WebView Timeout

- **URLRequest.timeoutInterval is per-request, not per-WebView**
  - Context: It lives on URLRequest, not InAppWebViewSettings. Setting it on the wrong object silently does nothing.
  - Impact: Timeouts won't fire, requests hang indefinitely.

## macOS Plugin Gap

- **macOS InAppWebView.swift creates bare URLRequest(url:) — drops method, body, headers, timeout**
  - Context: iOS uses URLRequest(fromPluginMap:) but macOS had no extension. POST, custom headers, and timeoutInterval were silently ignored.
  - Impact: Any non-trivial URLRequest on macOS would default to GET with no headers and no timeout.

## Dart Variable Shadowing

- **Dart local variable declarations shadow function parameters even when the local is declared later in the function body**
  - Context: A function parameter `Object? body` and a later `String? body` in the same function — Dart treats all references to `body` as the later-declared local, causing "referenced before declaration" errors.
  - Impact: Use `requestBody` or similar for the parameter to avoid shadowing the response-body local variable.
```
