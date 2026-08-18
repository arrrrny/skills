---
name: commit-and-pr
description: Commit all changes on the current branch, open a PR (default to master), merge it, then checkout the target branch and pull locally.
---

# Commit and PR

Use this skill when the user says something like:

- "Commit and PR"
- "Create a PR and merge it"
- "Ship this"
- "Push changes and open a pull request"
- "Merge my branch"
- "Commit and PR to master" / "PR to develop" / "PR to main" (overrides the target branch)

## Target Branch

If the user specifies a target branch (e.g. "PR to develop", "merge into staging"), use that
branch as the PR base. Otherwise, default to `master`.

This value is referred to below as `{target_branch}`.

## Prerequisites

- The `gh` (GitHub CLI) must be installed and authenticated (`gh auth status`).
- The remote must be configured (`git remote -v` should show `origin`).
- The current branch must not be the target branch (`{target_branch}` or `main`).

## Behavior

**Fully automatic. No confirmations. No questions.** Every step executes immediately without prompting the user.

## Workflow

### 1. Determine target branch

Check the user's request for a target branch. If they said something like "PR to develop" or
"merge into staging", extract that branch name. If they didn't specify one, use `master`.

Also check if the current branch is already `{target_branch}` or `main` — if so, abort.

### 2. Commit changes (if any)

Run `git status --short`. If there are changes:

1. Stage all files: `git add -A`
2. Run `git --no-pager diff --cached --stat` to see what's staged.
3. Determine commit type and scope by analyzing the changed files.
4. Generate a commit message in `type(scope): description` format (e.g. `feat(category-config): add priority field to channel config query`).
5. Create the commit immediately: `git commit -m "<message>"`

If there are **no changes**, simply **skip** the commit step and continue to the next step.

### 3. Push the current branch

```
git push -u origin <current-branch>
```

If the remote branch already exists, `-u` simply sets upstream — it's safe to always include.

### 4. Create a Pull Request

```
gh pr create --base {target_branch} --head <current-branch> --title "$(git log -1 --pretty=%s)" --body "$(git log -1 --pretty=%B)"
```

- This avoids opening an editor by providing title and body directly.
- If there were no new commits (no changes to commit in step 2), use the branch name as a descriptive fallback title.

After creation, log the PR URL for the user.

### 5. Merge the Pull Request

```
gh pr merge <current-branch> --merge
```

- **⚠️ CRITICAL: Do NOT pass `--delete-branch` or `-d` to `gh pr merge`.** The default merge keeps both the local and remote branch intact.
- If the merge fails (conflicts, etc.), inform the user with the error and abort.
- **Verify the remote branch still exists** after the merge:
  ```
  git ls-remote --heads origin <current-branch>
  ```
  If this returns empty (branch was deleted), stop immediately and notify the user.
- **Do NOT delete the local branch.** Skip any `git branch -d` or `-D` step entirely — the local branch is harmless and keeps future history traversal clean.

### 6. Checkout target branch and pull locally

```
git checkout {target_branch}
git pull origin {target_branch}
```

### 7. Done

Confirm to the user what happened:

- Branch committed (if there were changes) and pushed
- PR created and merged
- Local repo is now on `{target_branch}` with latest changes
