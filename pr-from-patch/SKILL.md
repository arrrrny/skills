---
name: pr-from-patch
description: Take a git-patch file from an external agent, apply it with adaptations, address review comments, create a PR closing the referenced issue. Run this when the user says "/pr-from-patch <patch-file> [<review-notes-file>]".
---

# PR from Patch Workflow

Use this skill when you receive a git-patch file from an external agent (e.g. Kimi, Daytona sandbox) and optionally a review notes file with architectural commentary. Your job is to:

1. **Understand the patch** — Read the full patch file. Understand every file it creates or modifies.
2. **Read the review notes** (if provided) — Extract all actionable critiques and suggested fixes.
3. **Understand the project** — Read the project structure, AGENTS.md, existing relevant source files, and the issue referenced in the patch (look for `Closes #N` or similar in the commit message).
4. **Adapt the patch** — The patch may have been generated against a different directory layout (e.g. `packages/zuraffa_core/lib/` when the project is a single package). Map paths correctly:
   - Strip or adjust prefixes that don't match the project structure.
   - If the patch introduces a new `Result` type but the project already has one, adapt the patch to use the existing `Result` type, adding any missing variants (e.g. `LoadingResult`) to the existing file.
   - If the patch introduces new modules (signals, usecase patterns, context, etc.), create them at the project-appropriate paths.
5. **Integrate into the existing barrel file** — Don't create a new barrel file unless the project is set up as a true monorepo. Instead, add the new exports to the existing main library file.
6. **Address ALL review comments** — Each critique is a task. Fix them before creating the PR. If a review point is already handled by the existing project code (e.g. `==`/`hashCode` already exists), note that explicitly.
7. **Modify the patch code as needed** to integrate correctly with the existing project:
   - Different type parameter counts on Result
   - Different import paths
   - Different naming conventions
8. **Run analysis and tests** — Ensure the code compiles and tests pass before creating the PR.
9. **Create a branch**, commit all changes, push, and open a PR that closes the referenced issue.

## Procedure

### Phase 1: Assessment

1. Read the patch file completely.
2. Read the review notes if provided.
3. Check `git branch --show-current` to know the base branch.
4. Check the referenced issue on GitHub (`gh issue view <number> --repo <owner/repo>`).
5. Read the project's barrel file and key source files to understand integration points.

### Phase 2: Adaptation

1. Map the patch paths to the actual project structure.
2. For each new file in the patch, create it at the adapted path.
3. For each modified file in the patch, adapt the changes to the existing code.
4. For each review critique, implement the fix:
   - **Code improvements** (new methods, safety guards): Implement directly.
   - **Design critiques** (equality, ergonomics): Address or document why already solved.
5. Add exports to the existing barrel file.

### Phase 3: Validation

1. Run `dart analyze` or `flutter analyze` on the package.
2. Run the new tests to ensure they pass.
3. Fix any issues.

### Phase 4: PR Creation

1. Create a branch named after the feature (e.g. `v6-track-1-1-async-signal-pipeline`).
2. Commit all changes with a message matching the patch's intent.
3. Push the branch.
4. Create a PR with:
   - Title matching the patch subject
   - Body summarizing changes and noting that review comments were addressed
   - Reference to the issue (`Closes #N`)

## Important Notes

- Never blindly apply a patch — always adapt it to the project structure.
- If the patch creates a separate sub-package (e.g. `packages/foo/`) that doesn't exist in the project, integrate the files directly into the main package instead.
- Review comments are authoritative — they represent architectural guidance. Address every single one.
- If a review comment suggests something that's already implemented in the existing codebase, note that clearly in the PR description.
- Always verify the patch compiles before creating the PR.
