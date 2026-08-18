---
name: coderabbit
description: >-
  CodeRabbit-style pull request review. Use when the user shares a GitHub pull
  request (URL or owner/repo#number) and asks for a code review, PR review, or
  "review like coderabbit". Fetches the PR diff and files, analyzes the changes,
  and produces a CodeRabbit-replica review: walkthrough summary with changes
  table and effort estimate, pre-merge checks, and inline-style findings with
  category/severity/effort badges, diff fix blocks, committable suggestions,
  and AI-agent prompts. Posts the finished review directly to GitHub (inline
  review comments + walkthrough comment) without asking for confirmation.
  With the --fix flag, applies fixes for all unresolved review comments on the
  PR: commits directly to the PR branch when zikzak-ai authored the PR,
  otherwise opens a new PR with the fixes.
---

# CodeRabbit-Style PR Review

Replicate CodeRabbit's code review output for any GitHub pull request the user shares.

## Workflow

1. **Parse the PR reference** — Extract `owner`, `repo`, and PR number from the user's URL (e.g. `https://github.com/owner/repo/pull/123`).

2. **Fetch PR data** using GitHub MCP tools (`pull_request_read`):
   - `get` — title, description, base/head branches, linked issues.
   - `get_diff` — the full diff. This is the primary review subject.
   - `get_files` — list of changed files (paginate if needed).
   - For surrounding context a diff hunk hides (e.g. full function bodies, definitions a change depends on), use `get_file_contents` at the head ref.
   - If the PR description links issues, read them to judge out-of-scope changes.

3. **Review the code** — Analyze every changed file for:
   - Functional correctness bugs (broken logic, unhandled edge cases, races, leaks)
   - Stability/availability risks (crashes, hangs, resource cleanup)
   - Security issues (injection, secrets, unsafe deserialization)
   - Performance problems (needless work, N+1, unbounded loops)
   - Maintainability (naming, dead code, contract mismatches)
   - Test quality (weak assertions, missing regression coverage)
   Verify each suspicion against actual file contents before reporting — never report an issue you haven't confirmed in code. Read enough surrounding context to avoid false positives.

4. **Classify each finding** with three badges (see references/format.md for the full taxonomy):
   - Category: 🎯 Functional Correctness, 🩺 Stability & Availability, 🔒 Security, ⚡ Performance, 📐 Maintainability & Code Quality, 🧪 Tests
   - Severity: 🔴 Critical, 🟠 Major, 🟡 Minor, 🔵 Trivial
   - Effort: ⚡ Quick win, 🔨 Medium, 🏗️ Heavy lift

5. **Produce the review** in CodeRabbit's exact format. Read references/format.md for the complete templates:
   - **Walkthrough comment**: prose summary, changes table (`Layer / File(s) | Summary`), estimated review effort (1–5 scale with minutes), pre-merge checks table, and a short rabbit-themed poem.
   - **Inline comments** (one per finding): badge header line, bold title, explanation anchored to exact line numbers, optional `Proposed fix` diff block in `<details>`, optional committable ```suggestion block, and a `🤖 Prompt for AI Agents` collapsible containing a self-contained fix instruction.
   - **Review status summary**: "Actionable comments posted: N", grouped nitpicks, and a combined "Prompt for all review comments".
   - Order findings by severity (Critical → Trivial). Group trivial/style items as nitpicks.

6. **Post the review to GitHub — always, without asking.** Never ask the user whether to post or where to output; posting is the default and only flow. Use GitHub MCP tools:
   - `pull_request_review_write` (`create`) to open a pending review on the PR head commit.
   - `add_comment_to_pending_review` once per actionable finding, anchored to exact diff lines (`subjectType: LINE`, `startLine`/`line`, `side: RIGHT`). Before posting, verify each anchor range against the real file contents — committable `suggestion` blocks only work when the range exactly covers the replaced lines.
   - `pull_request_review_write` (`submit_pending`, `event: COMMENT`) with the review status body: "Actionable comments posted: N", grouped nitpicks, the combined AI-agent prompt, and a one-paragraph verdict.
   - `add_issue_comment` for the walkthrough comment (walkthrough, pre-merge checks, poem).
   - In chat, report only a concise recap: findings count by severity, what was posted, and the PR link. Do not paste the full review in chat unless the user explicitly asks for chat-only output or a Markdown file.

## Fix mode (`--fix`)

When the user passes `--fix <PR URL>`, mirror CodeRabbit's 🪄 Autofix: apply fixes for all unresolved review comments on the PR, without asking for confirmation.

1. **Collect threads** — Fetch the PR (`get`: note `user.login`, head SHA/ref, base ref) and all review threads (`get_review_comments`, all pages). Every unresolved, non-outdated thread authored by zikzak-ai or coderabbitai is a fix item; nitpicks from review bodies count too.
2. **Verify before fixing** — Check each finding against the current head code (`get_file_contents` at head SHA). Fix only still-valid issues; record skipped ones with a brief reason (e.g. already fixed by a later commit).
3. **Apply minimally** — Preserve each finding's stated intent and the repo's conventions; changes must be compile-plausible.
4. **Deliver by PR author:**
   - PR author is `zikzak-ai` → commit the fixes directly to the PR's head branch (recommended path).
   - Otherwise → create a new branch off the head SHA (`coderabbit-fixes/pr-<number>`), push the fixes there, and open a new PR targeting the original PR's **head** branch, titled `fix: address review comments on #<number>`.
   - **Push integrity** (verified 2026-07-30): when pushing via `push_files` inline content, batch large file sets into multiple commits; then fetch every pushed file back at the branch ref (`get_file_contents`) and diff against the local ground truth. Open the PR only after all files match — never PR unverified pushes.
5. **Report** — The commit message / PR body lists every thread as fixed (file + lines) or skipped (reason). Finish by commenting on the original PR with the autofix summary and a link to the new PR when one was created.

## Review rules

- Every finding must cite file path and line range from the actual diff.
- Proposed diffs must use proper `-`/`+` lines with correct indentation and compile-plausible code.
- Be direct and specific: state what's wrong, why it matters, and the concrete fix. No generic advice.
- If the diff is clean, say so — CodeRabbit celebrates zero actionable comments ("🎉") rather than inventing issues.
- Match severity honestly; don't inflate nitpicks into majors.

## Format reference

Read [references/format.md](references/format.md) for the full badge taxonomy and copy-ready templates for the walkthrough comment, inline comments, suggestion blocks, and AI-agent prompts.
