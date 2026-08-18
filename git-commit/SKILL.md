---
name: git-commit
description: Stage all modified files and create a commit with a standardized conventional commit message based on the diff content.
---

# Git Commit

Use this skill when the user asks to "commit", "create a commit", "make a commit", "commit changes", or "write a commit message".

### Flags

- `-f` / `--force`: Skip confirmation and commit directly with the generated message.

## What it does

1. Stages all modified, added, deleted, and renamed files.
2. Analyzes the diff to determine the appropriate conventional commit type and scope.
3. Generates a commit message in `type(scope): description` format.
4. Ensures all CHANGELOG.md files have an `[Unreleased]` section at the top.
5. Creates the commit.

## Process

### 1. Check repository state

Run `git status --short` to see all changes. Categorize by:

- **Modified** (`M`): Existing files changed
- **Added** (`??` or `A`): New files
- **Deleted** (`D`): Removed files
- **Renamed** (`R`): Moved/renamed files

### 2. Determine commit type

Look at the aggregate of all changes to pick a type:

| Type       | When to use                                      |
| ---------- | ------------------------------------------------ |
| `feat`     | A new feature, new file, new capability          |
| `fix`      | Bug fix, error correction, edge case handling    |
| `refactor` | Code restructuring without behavior change       |
| `style`    | Formatting, renaming, import cleanup, lint fixes |
| `docs`     | Documentation, comments, readme changes          |
| `test`     | Adding or updating tests                         |
| `chore`    | Dependencies, build config, CI, tooling          |
| `perf`     | Performance improvements                         |

If multiple types apply, pick the most significant one (`feat` > `fix` > `refactor` > `chore` > `style`).

### 3. Determine scope

Scan the changed file paths for a common prefix or logical grouping:

- `git-commit`: changes to git-commit skill or related tooling
- `deps`: dependency updates
- `ci`: CI/CD changes
- `config`: configuration changes
- If changes span multiple unrelated areas, omit the scope.

Scope is optional — omit if no single clear scope.

### 4. Generate a short description

Write a concise imperative description (lowercase, no period, under 72 chars). Examples:

- `add similar listing barcode search feature`
- `fix crash when scanning invalid barcode`
- `refactor listing sheet title alignment`
- `update path-dependencies and package-upgrades skills`
- `bump flutter_svg to 2.3.0`

### 5. Update CHANGELOG.md files with [Unreleased] section

Before committing, ensure every `CHANGELOG.md` in the project has an `[Unreleased]`
section at the top for tracking what will go into the next release.

1. **Find** all CHANGELOG.md files using `find_path` with glob `**/CHANGELOG.md`.
   Exclude files inside `build/`, `.dart_tool/`, `.pub-cache/`, `node_modules/`,
   `.specify/`, and similar generated or tool-specific directories.

   **CRITICAL: Only update CHANGELOG.md files that are part of the project's own release tracking**
   (e.g., a root `CHANGELOG.md` or one alongside a releasable package). Do NOT touch
   changelogs inside `.specify/`, `.agents/`, `vendor/`, or other tooling/extension directories
   that track changes unrelated to the current feature work.

2. **Check** each file: read the first 5 lines.

3. **If** a `## [Unreleased]` header already exists at the top (before any version
   entry like `## 1.0.0` or `## [1.0.0]`), skip that file — it's already set up.

4. **Otherwise, prepend** the Unreleased section:
   - If the file starts with a top-level heading (`# Changelog`, `# Change Log`, etc.),
     insert `## [Unreleased]\n\n` right after that heading and before the first version entry.
   - If the file starts directly with version entries (`## 1.0.0`), prepend
     `## [Unreleased]\n\n` at the very top.

   Use `edit_file` to make the change:

   ```
   If file starts with a version entry:
     old_text: "## 1.0.0"
     new_text: "## [Unreleased]\n\n## 1.0.0"

   If file starts with a # header:
     old_text: "# Changelog\n\n## 1.0.0"
     new_text: "# Changelog\n\n## [Unreleased]\n\n## 1.0.0"
   ```

5. **Verify** by reading the first 3 lines of each modified file.

> The changelog modifications will be automatically included in the commit
> since step 8 uses `git add -A`.

### 6. Show the proposed commit message

Print the generated message clearly:

```
Proposed commit message:

  feat(scope): description

Files:
  M lib/src/file1.dart
  M lib/src/file2.dart
  A lib/src/new_file.dart
```

### 7. Ask the user to confirm (unless `-f` flag is set)

If the `-f` or `--force` flag was passed, **skip confirmation** and proceed directly to step 8.

Otherwise, display the proposed message and ask: "Create this commit? (y/n)" or let them edit the message.

- If yes → proceed.
- If no → let the user provide a custom message or cancel.

### 8. Create the commit

```bash
git add -A
git commit -m "type(scope): description"
```

If the user provided a custom message, use that instead.
