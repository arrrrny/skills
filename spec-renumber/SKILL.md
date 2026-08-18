---
name: spec-renumber
description: >-
  Renumber a feature spec directory (e.g. 033→034) to resolve numbering
  conflicts across worktrees/computers. Updates the spec folder,
  feature.json, AGENTS.md SPECKIT section, and the git branch.
---

# Spec Renumber

When you have a spec number conflict across worktrees or computers, use this
skill to renumber the current feature to a new number. It updates everything
in one shot: the spec directory, `.specify/feature.json`, `AGENTS.md`, and
the git branch.

## When to Use

Use this skill when the user says something like:

- "Renumber this spec to 034"
- "Change the feature number to 042"
- "We have a spec conflict, need to renumber this feature"

The user **must** provide the new spec number. If they don't, ask for it.

## Workflow

### 1. Detect the current feature

Determine these values from the project state:

| Value           | How to get it                                                                             |
| --------------- | ----------------------------------------------------------------------------------------- |
| Current branch  | `git branch --show-current` (e.g. `033-offline-category-configs`)                         |
| Old spec number | First 3 digits of the branch name (e.g. `033`)                                            |
| Feature name    | Everything after the `-` following the number (e.g. `offline-category-configs`)           |
| Spec directory  | `.specify/feature.json` → `feature_directory` (e.g. `specs/033-offline-category-configs`) |

**Validate**: Confirm the old spec directory actually exists at
`specs/{old_num}-{name}/` and the new number doesn't already exist at
`specs/{new_num}-{name}/`.

### 2. Rename the spec directory

Rename the spec folder:

```bash
mv specs/{old_num}-{name} specs/{new_num}-{name}
```

### 3. Update `.specify/feature.json`

Change the `feature_directory` value to the new path:
`specs/{new_num}-{name}`.

### 3b. Commit the renumbering changes

Before continuing with branch rename, stage and commit the spec directory rename
and the updated `feature.json` to the old branch:

```bash
git add specs/{new_num}-{name}/ .specify/feature.json
git commit -m "chore: renumber spec from {old_num} to {new_num}"
```

If the spec directory rename created unstaged changes in other files (e.g. internal
spec `.md` files updated in a later step), only stage the directory rename and
feature.json here. The remaining file updates will be committed in a subsequent
step after the branch rename.

### 4. Update `AGENTS.md` SPECKIT section

Find the line with `specs/{old_num}-{name}/plan.md` in the SPECKIT section
(between `<!-- SPECKIT START -->` and `<!-- SPECKIT END -->`) and replace
`{old_num}` with `{new_num}`.

### 5. Update internal spec file references

In every `.md` file under `specs/{new_num}-{name}/`, replace occurrences of
`{old_num}-{name}` with `{new_num}-{name}`. This catches:

- `**Feature Branch**: `{old_num}-{name}`` in spec.md
- `**Branch**: `{old_num}-{name}`` in plan.md
- `**Feature**: {old_num}-{name}` in data-model.md, quickstart.md,
  research.md, contracts
- Path references like `/specs/{old_num}-{name}/` in tasks.md

Be precise: only replace the full `{old_num}-{name}` pattern so you don't
accidentally rewrite unrelated numbers (like task IDs).

### 6. Rename the git branch

```bash
# Rename local branch
git branch -m {old_branch} {new_branch}

# Push new branch to remote
git push origin -u {new_branch}

# Delete old remote branch (if it exists — it may not if never pushed)
git push origin --delete {old_branch} 2>/dev/null || true
```

### 7. Verify

- Check `git branch --show-current` returns the new branch name
- Read the first few lines of `specs/{new_num}-{name}/plan.md` to confirm
  references are updated
- Read `.specify/feature.json` to confirm the path
- Check the SPECKIT section of `AGENTS.md`
