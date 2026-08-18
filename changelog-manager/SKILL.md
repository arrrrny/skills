---
name: changelog-manager
description: Update all CHANGELOG.md files in a project with a new version entry. Given a version number and changelog content, prepends the entry to every CHANGELOG.md found in the project.
---

# Changelog Manager

Update CHANGELOG.md files across the project for a new release. Steps are executed directly without asking for input — all required information is derived from context or git history.

## When to Use

Use this skill when the user says something like:

- "Update changelogs for version 1.2.3"
- "Add a changelog entry for the new release"
- "Prepare changelogs for version X.Y.Z"
- Any request involving updating CHANGELOG.md files with a version entry

## Workflow

1. **Determine version**: Use the version provided by the user or caller. If missing, read the current version from `pubspec.yaml` (or `package.json`) and derive the next version from the scope of changes in git log. Must be valid semver (`X.Y.Z`).

2. **Generate changelog content from git log**: Run `git log --pretty=format:"%s" <last_tag>..HEAD` to get commits since the last tag. Categorize entries under appropriate headings:
   - **Features** for `feat:` / `feature:` commits
   - **Fixes** for `fix:` commits
   - **Refactors** for `refactor:` commits
   - **Chores** for `chore:` / `deps:` / `ci:` commits
   - **Docs** for `docs:` commits
   - **Style** for `style:` commits
   - **Tests** for `test:` commits

   Non-conventional commits are grouped under **Changes**. If no tags exist, use all commits since the beginning.

3. **Find all CHANGELOG.md files** using `find_path` with glob `**/CHANGELOG.md`. Exclude files inside `build/`, `.dart_tool/`, `.pub-cache/`, `node_modules/`, and similar generated directories.

4. **For each CHANGELOG.md file**, prepend a new version entry at the top:

   ```markdown
   ## VERSION - DATE

   CHANGELOG_CONTENT
   ```

   Where:
   - `VERSION` is the version number (e.g., `1.2.3`)
   - `DATE` is today's date in `YYYY-MM-DD` format
   - `CHANGELOG_CONTENT` is the generated content (keep its original formatting)

   Use `read_file` then `edit_file` or `write_file` to prepend.

5. **Verify** by reading the first 5 lines of each modified CHANGELOG.md.
