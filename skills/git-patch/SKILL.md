---
name: git-patch
description: Generate a git patch file from a PR number or branch diff, saving it to the patches/ folder for external agent handoff (Daytona, Kimi, review, etc.).
---

# Git Patch Generator

Generate a standard `git format-patch`-style patch file from either a GitHub PR or a local branch diff, saved to the project's `patches/` directory.

This is the inverse of `pr-from-patch`: where that skill consumes a patch and creates a PR, this skill takes a PR or branch and produces a patch file for external agents to work with.

## Usage

Activate with: `@git-patch` followed by the mode:

- `@git-patch pr 189` — generate patch from PR #189
- `@git-patch branch v6-track-1-1-async-signal-pipeline` — generate patch from a branch diff
- `@git-patch branch v6-track-1-1-async-signal-pipeline --base=master` — diff against a specific base
- `@git-patch` — prompt interactively for what to generate

## Procedure

### 1. Determine the Source

Parse the user's request: is it a PR number, a branch name, or unspecified?

- If a PR number: verify it exists with `gh pr view <number> --repo <owner/repo>`
- If a branch name: verify it exists locally with `git branch --list <name>` and note the base branch (default `development`)
- If neither: list recent PRs (`gh pr list --limit 5`) and branches (`git branch`) to help the user decide

### 2. Determine Output Path

The patch file goes into the project's `patches/` directory. Name it descriptively:

- For PRs: `patches/pr-<number>-<feature-slug>.patch` (extract slug from PR title)
- For branches: `patches/<branch-name>.patch`
- If the file already exists, overwrite it with a warning

Ensure the `patches/` directory exists (create it if not).

### 3. Generate the Patch

#### Mode A: From a PR

Use `git format-patch` style for consistency. Fetch the PR ref and generate:

```bash
# Fetch the PR head
git fetch origin pull/<number>/head:<branch-name>-pr-<number>

# Generate format-patch against the base branch
git format-patch <base-branch>..<branch-name>-pr-<number> --stdout > patches/<filename>.patch

# Clean up the temporary branch
git branch -D <branch-name>-pr-<number> 2>/dev/null
```

If the PR's base branch is unknown, query it with `gh pr view <number> --json baseRefName --jq .baseRefName`.

#### Mode B: From a Branch

```bash
git format-patch <base>..<branch> --stdout > patches/<filename>.patch
```

Default base is `development`. User can override with `--base=master` or similar.

### 4. Verify the Patch

- Check the patch file was created: `wc -l patches/<filename>.patch`
- Print the first 10 lines (the commit message header) so the user can confirm it's correct
- Run `git apply --stat patches/<filename>.patch` to show what files are touched (don't actually apply it)

### 5. Report

Tell the user:
- Where the patch was saved (absolute path)
- How many files it touches
- The commit subject line
- How many lines total
- Suggested command for the receiving agent: `/pr-from-patch patches/<filename>.patch`

## Notes

- Always use `--stdout` with `format-patch` so we get a single file, not multiple per-commit files.
- If a branch has multiple commits, `format-patch` includes all of them — that's correct.
- For single-commit branches, the patch will contain exactly one commit.
- The patch file should be in standard git format so `git am` can apply it on the receiving end.
- Do NOT modify the branch or PR content — this is a read-only generation tool.
- Prefer `git --no-pager` on read-only git commands to avoid pager hangs.
