# Apply Fixes — Delivery Templates

Copy-ready templates for committing, opening the `--pr` PR, and reporting back on the original PR.

## 1. Direct Commit (default mode)

Push to the PR's existing head branch. One commit per delivery run (all threads in one commit) unless a large file set requires batching.

```text
fix: address review comments on #<number>

Fixed:
- <file>:<lines> — <what was changed, e.g. "replaced hand-rolled JSON with dart:convert">
- <file>:<lines> — <what was changed>

Skipped:
- <file>:<lines> — <reason, e.g. "already fixed in <sha>">
```

> ⚠️ **Push integrity** (verified 2026-07-30): after pushing, fetch each pushed file back at the branch ref and diff against your local ground truth. Never report a push you haven't verified byte-for-byte.

## 2. New PR (`--pr` mode)

Branch: `fix/pr-<number>` (created off the PR's **head SHA**).

```markdown
# fix: address review comments on #<number>

Targets: <original-head-branch> (head of PR #<number>)
Base: this branch feeds into PR #<number>; merge it there to bring the fixes into that PR.

## Fixed (N)
- [x] `<file>:<lines>` — summary of fix
- [x] `<file>:<lines>` — summary of fix

## Skipped (M)
- [ ] `<file>:<lines>` — reason

## Verification
- <command> — ✅ / ❌ / not run (state why)
```

## 3. Report Comment (on the original PR — always)

```markdown
🪄 Autofix applied via `apply-fixes`.

**Fixed (N):**
- `file:lines` — what changed
- `file:lines` — what changed

**Skipped (M):**
- `file:lines` — reason

**Verification:** `dart analyze` ✅ · `dart test` ✅ (or: not run — <reason>)

Delivered: [direct commit on <branch>](<commit-url>) — or — [New PR #<n>](<pr-url>) targeting <branch>
```

## Nitpick handling

Nitpicks (🔵 Trivial) that appeared in a review body count as fix items — apply them too (e.g. removing an unused import) and list them under Fixed. If a nitpick is style-only and the repo convention conflicts, note it in Skipped with the reason.
