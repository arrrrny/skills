---
name: apply-fixes
description: >-
  Apply fixes for all unresolved review comments on a GitHub pull request.
  Use when the user asks to fix review comments, resolve review threads, apply
  PR suggestions, or says "apply fixes" / "apply the review" for a PR. Reads
  the PR's review threads (including CodeRabbit-style comments with proposed
  fixes and suggestion blocks), verifies each against the current head code,
  applies them minimally, and delivers BY DEFAULT as a direct commit to the
  PR's head branch, or with --pr as a new PR targeting the head branch.
  Always reports fixed/skipped items back on the original PR.
---

# Apply Fixes — Resolve Review Comments on a PR

Replicate CodeRabbit's 🪄 Autofix for any GitHub pull request the user shares. Read the review threads, verify them against the current code, apply the fixes, and deliver.

## Modes

- **default** — apply fixes and commit **directly to the PR's head branch**.
- **`--pr`** — create a new branch off the head SHA, push the fixes there, and open a **new PR** targeting the original PR's head branch.

## Workflow

1. **Parse the PR reference** — Extract `owner`, `repo`, and PR number from the user's URL (e.g. `https://github.com/owner/repo/pull/123`).

2. **Fetch the PR and all review threads** using GitHub MCP tools:
   - `pull_request_read` (`get`) — title, `user.login` (PR author), head SHA/ref, base ref.
   - `pull_request_read` (`get_review_comments`) — all pages. Every thread that is **unresolved and not outdated** is a fix item; include threads authored by anyone (coderabbit-style, human, your own earlier review).
   - `pull_request_read` (`get_reviews`) — nitpicks listed in submitted review bodies count as fix items too.

3. **Verify before fixing** — Check each finding against the **current head code** (`get_file_contents` at the head SHA). Fix only still-valid issues. Record skipped ones with a brief reason (e.g. "already fixed by a later commit", "no longer applicable", "out of scope").

4. **Apply minimally** — Preserve each finding's stated intent and the repo's conventions; changes must be **compile-plausible**. When a comment includes a `Proposed fix` or a committable `suggestion` block that is still valid, apply that — it encodes the author's intent. One fix per thread. Do not add features or refactor unrelated code.

5. **Verify the change** — If a checkout is available (local project root, or the Daytona sandbox via `r.sh`/`r2.sh` for teleported use), run the repo's checks (`dart analyze`/`dart test`, `flutter analyze`/`flutter test`, etc.). If verification cannot be run, state that explicitly in the report — never claim it passed without running it.

6. **Deliver** — see [references/delivery.md](references/delivery.md) for templates:
   - **Default (direct commit)**: push the fixes to the PR's existing **head branch**.
     - Same-repo PR → `push_files` targeting the head branch. Batch large file sets into multiple commits. **Push integrity (verified 2026-07-30):** after pushing, fetch every pushed file back at the branch ref (`get_file_contents`) and diff against your local ground truth. Only finish when all files match — never report a push you haven't verified.
     - Cross-repo (fork) PR → fall back to the `--pr` path (you can't push to a fork you don't own).
   - **`--pr`**: create branch `fix/pr-<number>` off the head SHA, push the fixes there, and open a new PR targeting the original PR's **head** branch, titled `fix: address review comments on #<number>`. Verify pushed files (same integrity rule) before opening the PR.

7. **Report** — Comment on the original PR: every thread as **fixed** (file + lines) or **skipped** (reason), the verification result, and a link to the new PR when one was created.

## Review rules

- Every fix must map back to a specific thread (file + line range).
- If a thread is already resolved or outdated, skip it silently — don't re-fix.
- If nothing is actionable (all threads resolved/outdated/invalid), say so — don't create busywork.
- Be honest in the report: fixed counts, skipped counts with reasons, verification status.

## Format reference

Read [references/delivery.md](references/delivery.md) for the copy-ready commit message, PR body, and report-comment templates.
