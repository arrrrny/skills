---
name: skill-to-bash
description: Analyze a skill's SKILL.md, identify deterministic steps that can be implemented as fast/reliable bash scripts, create and verify those scripts, then update the skill to reference them instead. Scripts live at both user-global (~/.agents/scripts/) and project-local (.agents/scripts/) levels, mirroring the skill convention. Preserves original skill content as doc comments in the .sh files.
---

# Skill to Bash Converter

Use this skill when the user says something like:

- "Turn this skill into bash scripts"
- "Convert the changelog-manager skill to scripts"
- "Make parts of this skill faster by using bash"
- "Refine skill X into atomic bash scripts"
- "Create deterministic scripts from skill instructions"

## Workflow

### 1. Determine the target skill

If the user specifies a skill name (e.g., "changelog-manager"), find its SKILL.md:

- Check `~/.agents/skills/<name>/SKILL.md` (global skills)
- Check `.agents/skills/<name>/SKILL.md` (project-local)
- Also check any `.agents/skills/` directories in other project roots

If no skill is named, ask the user which skill to convert.

Read the full `SKILL.md` file.

### 2. Analyze the skill and identify bash-able parts

Read the SKILL.md instructions carefully and determine which steps can be implemented as standalone bash scripts.

**Good candidates for bash scripts (deterministic):**

- File operations (find, read, write, replace text in files)
- Git commands (log, tag, status, add, commit, push)
- Running pub/dart commands (pub get, pub publish, pub add, dart compile)
- Grep/searching files
- Version string manipulation
- Running tests
- Creating/modifying CHANGELOG.md entries (prepend text to files)
- Any step where input/output is well-defined and no AI/LLM judgment is needed

**Poor candidates (keep as skill instructions):**

- Deciding on version bumps (breaking/major/minor judgment)
- Categorizing git commits into feature/fix/chore groups
- Complex decision-making about what to include/exclude
- Writing descriptive changelog text from git log
- Generating release notes
- Any step requiring human-style judgment or natural language generation

**For each bash-able part**, define:

- Exact inputs (arguments, files, environment variables)
- Expected output (stdout, file changes, exit code)
- Error conditions and how to handle them

### 3. Create the bash scripts

Scripts can live at two levels, mirroring the skill convention:

| Scope             | Path                 | When to use                                                                                                                                                         |
| ----------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Global**        | `~/.agents/scripts/` | Only when the script applies identically across **all projects** (e.g., a universal "find all CHANGELOG.md files" helper). Rare.                                    |
| **Project-local** | `.agents/scripts/`   | **Default for most scripts.** The logic is tied to a specific project's structure, conventions, or toolchain. Preferred unless there's a clear reason to go global. |

**Rule of thumb**: Most scripts should be project-local. Only promote to global if you're certain the script produces the same results in any project without modification.

Script naming convention:

```
<scope>/agents/scripts/<target-skill-name>-<step-name>.sh
```

When the skill being converted is **global** (`~/.agents/skills/<name>/`): still default to creating the script in the **current project's** `.agents/scripts/` unless the operation is truly universal. Only use `~/.agents/scripts/` for genuinely cross-project helpers.

**Script structure:**

```bash
#!/usr/bin/env bash
# (c) <target-skill-name> > <step-name>
# Original from: <target-skill-name> SKILL.md
# ---
# <Original skill content lines as doc comments, prefixed with #>
# ---
set -euo pipefail

# ...
```

Rules:

- `set -euo pipefail` for safety
- Accept all inputs as CLI arguments (positional or `--flags`)
- Use `realpath` or relative paths from the repo root
- Validate arguments before executing
- Print clear error messages to stderr
- Exit 0 on success, 1 on failure
- Be idempotent where possible (e.g., don't fail if file already exists)
- Use `--dry-run` flag when applicable for testing
- Keep each script focused on ONE operation

### 4. Test and verify each script

Before replacing any skill content:

1. **Dry run**: Run the script with `--dry-run` (if implemented) or in a safe context
2. **Run the script**: Execute it with real arguments
3. **Verify output**:
   - Check exit code is 0
   - Check files were modified correctly (if applicable)
   - Check stdout/stderr contains expected output
   - For git operations, verify with `git --no-pager log/status/diff`
4. **Test error handling**: Try invalid inputs to verify proper error messages

For scripts that modify files, verify both:

- The modification happened correctly
- The file is still valid (not corrupted)

### 5. Update the skill

After a script is verified, update the target SKILL.md:

1. Replace the old instruction text with a reference to the bash script:

   ```markdown
   Run `.agents/scripts/<name>.sh <args>` to accomplish this step.
   ```

2. Keep the surrounding context and any non-deterministic steps as skill instructions.

3. Ensure the script reference includes the exact arguments needed.

### 6. Validation checklist

Before finishing:

- [ ] All bash scripts have `set -euo pipefail`
- [ ] All bash scripts validate arguments
- [ ] All bash scripts exit with meaningful error messages
- [ ] All bash scripts are idempotent where possible
- [ ] All bash scripts have been run and verified at least once
- [ ] Original skill content is preserved as doc comments in the .sh file
- [ ] SKILL.md has been updated to reference the scripts
- [ ] `git status` shows only expected changes

### 7. Rollback safety

If a script fails during verification:

- DO NOT update the SKILL.md
- Report the failure with the error message
- Offer to debug the script
- Only proceed when the script passes verification

## Example implementation

For a "find all CHANGELOG.md files" step:

**Old skill text:**

```markdown
Find all CHANGELOG.md files in the project using `find_path` with the glob `**/CHANGELOG.md`.
```

**Generated bash script (`.agents/scripts/changelog-manager-find-changelogs.sh`):**

```bash
#!/usr/bin/env bash
# (c) changelog-manager > find-changelogs
# Original from: changelog-manager SKILL.md
# ---
# 3. **Find all CHANGELOG.md files** in the project using `find_path` with
#    the glob `**/CHANGELOG.md`. Exclude files inside `build/`, `.dart_tool/`,
#    `.pub-cache/`, and similar generated directories.
# ---
set -euo pipefail

exclude_dirs=("build" ".dart_tool" ".pub-cache" ".git" "node_modules" ".specify")
find_pattern="CHANGELOG.md"

for dir in "$@"; do
  result=$(find "$dir" -name "$find_pattern" -type f \
    $(printf "! -path '*/%s/*' " "${exclude_dirs[@]}") 2>/dev/null)
  echo "$result"
done
```

**Updated skill text:**

```markdown
Run `.agents/scripts/changelog-manager-find-changelogs.sh <project-root>` to find all CHANGELOG.md files.
```
