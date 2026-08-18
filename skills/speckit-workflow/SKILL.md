---
name: "speckit-workflow"
description: "Self-bootstrapping SDD workflow — introspects any project, generates a customized project-local speckit-workflow tuned to the project's architecture, then runs the full Specify → Plan → Tasks → Implement lifecycle autonomously."
---

# Speckit Workflow (Global) — Self-Bootstrapping SDD

This global skill runs the full Specification-Driven Development lifecycle in **any** project. On first run in a project, it:

1. **Bootstraps** — introspects the project's language, framework, build system, testing tools, code generation, and architecture, then generates a customized project-local `.agents/skills/speckit-workflow/SKILL.md`
2. **Executes** — runs the full SDD lifecycle (Pre-Flight → Specify → Plan → Tasks → Implement → Post)

On subsequent runs in that project, the project-local version takes over automatically with all its customizations, and the global version is only needed for new/unbootstrapped projects.

**No review gates. No questions. No stops.** If information is missing, search the project. If truly blocked, make a reasonable default decision and continue.

---

## User Input

```text
$ARGUMENTS
```

Parse the user input for the following:

**Required:**

- **Feature description** — A natural language description of what to build.

**Optional (extracted from input if present):**

- **scope** — One of `full`, `core-only`, `ui-only`, `data-only`, `docs-only`. Defaults to `full`.
- **branch** — Explicit branch name (e.g., `--branch my-feature`). If not provided, auto-generates from the feature description.

If no feature description is provided, **stop immediately** — the workflow cannot proceed.

---

## Core Rule: Never Ask, Always Search

When you encounter ambiguity or missing information during any step, follow this priority order:

1. **Search the project**: Use `grep`, `find_path`, or `claude_context_search_code` to find existing patterns.
2. **Read the docs**: Check project docs — `docs/`, `openwiki/`, `wiki/`, `README.md`, etc.
3. **Check existing specs**: Look for `specs/` or `features/` directories for prior art.
4. **Read `AGENTS.md`**: Project-specific rules and conventions.
5. **Read `INSIGHTS.md`**: Hard-won knowledge from past sessions.
6. **Read project config files**: `pubspec.yaml`, `package.json`, `Cargo.toml`, etc. for build/test scripts.
7. **Check existing implementations**: Look at actual source code for established patterns.

Only if absolutely nothing exists should you make a reasonable default decision and document it.

---

## Bootstrap Detection

Check whether this project already has a customized project-local `speckit-workflow`:

1. Does `<project-root>/.agents/skills/speckit-workflow/SKILL.md` exist?
2. Read its frontmatter — does it have `x-generated-by: speckit-workflow`?

**If a project-local version exists with the `x-generated-by` marker** → **Skip to Execution Phase.** The project is already bootstrapped.

**If a project-local version exists but without the marker** → It was likely hand-crafted. Do NOT overwrite it. Skip to the **Execution Phase** and use the global workflow directly — the existing project-local file will take over in future sessions.

**If no project-local version exists** → **Run Bootstrap Phase** below, then Execution Phase.

---

## Bootstrap Phase

Run this phase only once per project (or when the project architecture changes significantly).

### Step 1: Project Introspection

Run ALL of the following checks and collect the results. Use exact grep and file detection — do not guess.

#### Language & Package Manager

Check for the presence of project configuration files in the project root:

| File                                 | Language              | Package Manager           |
| ------------------------------------ | --------------------- | ------------------------- |
| `pubspec.yaml`                       | Dart/Flutter          | `pub`                     |
| `package.json`                       | JavaScript/TypeScript | `npm` or `yarn` or `pnpm` |
| `Cargo.toml`                         | Rust                  | `cargo`                   |
| `go.mod`                             | Go                    | `go mod`                  |
| `pyproject.toml`                     | Python                | `pip` or `poetry`         |
| `Gemfile`                            | Ruby                  | `bundler`                 |
| `CMakeLists.txt`                     | C/C++                 | `cmake`                   |
| `build.gradle` or `build.gradle.kts` | Java/Kotlin           | `gradle`                  |
| `Package.swift`                      | Swift                 | `swift`                   |

If multiple are found, prioritize based on source directory structure.

#### Framework & Build System

- **Flutter**: Check for `flutter` in pubspec.yaml dependencies, or `flutter` CLI availability.
- **Node.js**: Check package.json for `next`, `react`, `vue`, `angular`, `express`, `nest`.
- **Rust**: Check Cargo.toml for `actix-web`, `axum`, `rocket`, `tauri`.
- **Python**: Check for `django`, `flask`, `fastapi` in dependencies.
- **Build commands**: Check for `Makefile`, scripts section in `package.json`, `justfile`, `taskfile`.

Detect ALL available build commands:

```bash
# Check for common build tools (run inside project root)
which flutter 2>/dev/null && echo "flutter available"
which dart 2>/dev/null && echo "dart available"
which zfa 2>/dev/null && echo "zfa available"
which cargo 2>/dev/null && echo "cargo available"
which go 2>/dev/null && echo "go available"
which node 2>/dev/null && echo "node available"
which npm 2>/dev/null && echo "npm available"
```

#### Test System

- **Dart/Flutter**: `dart test`, `flutter test`, check for `test/` directory.
- **JavaScript/TypeScript**: `jest`, `mocha`, `vitest`, `playwright` in package.json.
- **Rust**: `cargo test`.
- **Python**: `pytest`, `unittest`.
- **Go**: `go test`.

#### Analysis & Linting

- **Dart**: `analysis_options.yaml`, `dart analyze`.
- **JavaScript/TypeScript**: `.eslintrc*`, `tsconfig.json`, `prettier`.
- **Rust**: `clippy`, `rustfmt`.
- **Python**: `.flake8`, `pylint`, `mypy`.

#### Code Generation

- **Dart**: `build.yaml` (build_runner), `zfa` CLI.
- **TypeScript**: `graphql-codegen`, `prisma`, `typeorm`.
- **Rust**: `cargo generate`.
- **General**: Check for `generators/`, `templates/`, or scaffolding scripts.

#### Architecture Patterns

Detect project structure by checking for directory layouts:

```bash
# Check for common architecture patterns (run inside project root)
# Clean Architecture / Domain-Driven
ls -d lib/src/domain lib/src/data lib/src/presentation 2>/dev/null && echo "clean_architecture"
ls -d lib/domain lib/data lib/presentation 2>/dev/null && echo "clean_architecture"

# Feature-based
ls -d lib/features 2>/dev/null && echo "feature_based"
ls -d src/features 2>/dev/null && echo "feature_based"
ls -d app/features 2>/dev/null && echo "feature_based"

# MVC
ls -d app/models app/views app/controllers 2>/dev/null && echo "mvc"

# Standard layouts
ls -d src lib app 2>/dev/null

# Check for source root (where does main code live?)
ls -d lib/src lib app/src src 2>/dev/null
```

Also check for state management (BLoC, Riverpod, Redux, Provider) and DI (get_it, injectable, Provider, manual).

#### Documentation

Check for:

- `docs/` directory
- `openwiki/` directory with `quickstart.md`
- `AGENTS.md` file
- `INSIGHTS.md` file
- `specs/` or `features/` directory
- `CHANGELOG.md`
- `README.md` with architecture description

#### Project Rules

Read `AGENTS.md` if it exists — extract key conventions, generation contracts, hard rules, and architecture guidance. This is critical for generating a customized workflow that respects project-specific rules.

#### Package Scripts

Read `package.json` scripts section (if Node.js) to discover available npm/yarn/pnpm scripts for build, test, lint, and code generation.

### Step 2: Architecture Profile

After all checks, compile an Architecture Profile like this (store as a reference for the template):

```
project_name: {derived from repo or directory name}
language: {dart | typescript | rust | python | go | ...}
framework: {flutter | next | react | actix | django | ...}
package_manager: {pub | npm | yarn | pnpm | cargo | pip | poetry | go-mod | ...}

# Build system
build_commands:
  primary: {e.g., "flutter build", "npm run build", "cargo build", "zfa build"}
  secondary: {e.g., "dart run build_runner build", "npm run compile", ...}
  fallback: {e.g., "dart compile", "tsc", ...}

# Test system
test_commands:
  primary: {e.g., "flutter test", "npm test", "cargo test", "go test ./..."}
  secondary: {e.g., "dart test", "npm run test:unit", ...}
  per_file: {e.g., "flutter test test/path/to/test.dart", "npx jest path/to/test.ts", ...}

# Analysis/linting
analyze_commands:
  primary: {e.g., "dart analyze", "npm run lint", "cargo clippy", "go vet ./..."}
  secondary: {e.g., "npm run typecheck", ...}

# Code generation
code_generation:
  has_generators: {true | false}
  tools:
    - name: {zfa | build_runner | prisma | graphql-codegen | ...}
      commands:
        entity_create: {e.g., "zfa entity create"}
        make: {e.g., "zfa make"}
        build: {e.g., "zfa build", "dart run build_runner build"}
  available: {true | false}

# Architecture
architecture_pattern: {clean_architecture | feature_based | mvc | layer_based | unknown}
source_root: {e.g., "lib/src", "lib", "app/src", "src"}
has_domain_layer: {true | false}
has_data_layer: {true | false}
has_presentation_layer: {true | false}
state_management: {bloc | riverpod | provider | redux | none | unknown}
di_system: {get_it | injectable | provider | manual | unknown}

# Project structure
test_root: {e.g., "test/", "tests/", "__tests__/"}
specs_dir: {e.g., "specs/", "features/", "rfcs/"}
docs_dirs:
  - {e.g., "docs/", "openwiki/", "wiki/"}

# Documentation
has_agents_md: {true | false}
has_insights_md: {true | false}
has_openwiki: {true | false}
has_readme: {true | false}

# Commands (from package.json scripts or known tools)
custom_scripts:
  build: {e.g., "npm run build"}
  test: {e.g., "npm run test"}
  lint: {e.g., "npm run lint"}
  gen: {e.g., "npm run generate"}

# Project rules summary
project_rules: |
  {key rules from AGENTS.md, if any}
```

### Step 3: Generate Custom Workflow

Create the project-local `.agents/skills/speckit-workflow/SKILL.md` using the Architecture Profile above.

**First, ensure the directory exists:**

Create `<project-root>/.agents/skills/` if it doesn't exist, then `<project-root>/.agents/skills/speckit-workflow/`.

**Then, generate the SKILL.md** using the template below. Replace ALL placeholders with values from the Architecture Profile. Every `{{placeholder}}` must be filled.

<generated-workflow-template>
---
name: "speckit-workflow"
description: "Custom SDD workflow for {{project_name}} — generated by speckit-workflow (global)"
x-generated-by: speckit-workflow
x-project-language: {{language}}
x-project-framework: {{framework}}
---

# Speckit Workflow ({{project_name}}) — Custom SDD

Customized for {{project_name}} ({{language}}/{{framework}}). This workflow was generated by the global `speckit-workflow` skill based on the project's detected architecture.

## Project-Specific Conventions

| Aspect           | Value                    |
| ---------------- | ------------------------ |
| Language         | {{language}}             |
| Framework        | {{framework}}            |
| Package Manager  | {{package_manager}}      |
| Architecture     | {{architecture_pattern}} |
| State Management | {{state_management}}     |
| DI System        | {{di_system}}            |
| Source Root      | {{source_root}}          |
| Docs             | {{docs_dirs}}            |

{{#if has_agents_md}}

## Project Rules

Read `AGENTS.md` in the project root for critical project-specific rules, generation contracts, and conventions.

{{project_rules}}
{{/if}}

## Commands

### Build

```
{{build_commands.primary}}
```

{{#if build_commands.secondary}}
Secondary: `{{build_commands.secondary}}`
{{/if}}

Fallback: `{{build_commands.fallback}}`

### Test

```
{{test_commands.primary}}
```

To run a specific test file:

```
{{test_commands.per_file}}
```

{{#if test_commands.secondary}}
Also: `{{test_commands.secondary}}`
{{/if}}

### Analyze

```
{{analyze_commands.primary}}
```

{{#if analyze_commands.secondary}}
Also: `{{analyze_commands.secondary}}`
{{/if}}

### Code Generation

{{#if code_generation.has_generators}}
{{#each code_generation.tools}}
**{{name}}**: Use `{{commands.build}}` to regenerate after making changes.

{{#if commands.entity_create}}
Entity creation: `{{commands.entity_create}}`
{{/if}}
{{#if commands.make}}
Architecture generation: `{{commands.make}}`
{{/if}}
{{/each}}
**Hard rules (if applicable):** Do not hand-create generated files. Do not call underlying generators directly — always use the project's code generation toolchain. Do not invent alternate folder structures.
{{else}}
No code generation tools detected. Create files by hand following existing patterns.
{{/if}}

## Documentation References

{{#if has_openwiki}}

- **OpenWiki**: `openwiki/` — start with `openwiki/quickstart.md` for architecture overview, then follow cross-references.
  {{/if}}
- **AGENTS.md**: Project rules and conventions (project root).
- **INSIGHTS.md**: Hard-won knowledge from past sessions.
- **Existing specs**: `{{specs_dir}}` for prior feature designs, plans, and tasks.
- {{#each docs_dirs}}**{{this}}**: Project documentation.{{/each}}

---

## Workflow Execution

Run the full SDD lifecycle **without any user interaction**.

### Pre-Flight: System Context & Project Health

#### 1. Triage — Task Classification & Depth Assessment

1. **Classify the task**:
   - **Bug Fix**: Something is broken
   - **New Feature**: Building something new
   - **Change to Existing Feature**: Modifying existing behavior
2. **Assess complexity** (Simple vs Complex vs Ambiguous) based on scope, patterns, cross-cutting concerns, clarity, and risk.
3. **Determine depth**:
   - **Simple/Straightforward** → Skip deep docs reading.
   - **Complex or Ambiguous** → Full deep-dive: read docs, AGENTS.md, INSIGHTS.md, existing specs.

Output:

> **Triage Result**: {Bug Fix | New Feature | Change to Existing Feature}
> **Complexity**: {Simple | Complex | Ambiguous}

#### 2. Context Gathering

Based on triage depth:

- **Simple**: Read README.md, check source structure lightly. Proceed quickly.
- **Complex or Ambiguous**:
  - Read `AGENTS.md` for project rules.
  - Read `INSIGHTS.md` for prior learnings.
  - Read {{docs_dirs}} documentation.
  - Check {{specs_dir}} for similar features.
  - Read the relevant source code sections.
  - Read `{{source_root}}` to understand patterns.

#### 3. Project Health Baseline

Establish a baseline before making changes:

1. Run the project's analyze/lint command:
   ```
   cd <project-root> && {{analyze_commands.primary}}
   ```
2. If pre-existing errors exist, note them:
   ```
   **Health Baseline**: `{{analyze_commands.primary}}` shows {N} errors and {M} warnings. {X} errors are pre-existing.
   ```
3. If no errors:
   ```
   **Health Baseline**: ✅ No pre-existing analysis errors — proceeding.
   ```

{{#if code_generation.has_generators}} 4. Check code generation viability:

```
cd <project-root> && {{code_generation.tools.0.commands.build}}
```

{{/if}}

**If the project has pre-existing failures, proceed anyway** — partial implementation is better than none.

#### 4. Scope Assessment

Before implementation, determine which parts of the project the feature touches:

1. **Source code** — Which `{{source_root}}` subdirectories are affected?
2. **Configuration** — Is any config file (package manager config, build config, lint config) affected?
3. **Documentation** — Is any documentation affected?
4. **Tests** — Which `{{test_root}}` files are affected?
5. **Assets/Resources** — Are any assets, migrations, or resource files affected?

> **Affected Areas**: {list}
> **Scope**: {scope}

---

### Step 1: Specify

Call `/speckit-specify` with the feature description as arguments.

Wait for `/speckit-specify` to complete before proceeding.

**Health check**: Run `{{analyze_commands.primary}}` — if new errors appear, note them but continue.

Do NOT present the spec for review. Do NOT ask for approval. Proceed immediately.

---

### Step 2: Plan

Call `/speckit-plan` with the feature description as arguments.

Wait for `/speckit-plan` to complete before proceeding.

**Health check**: Run `{{analyze_commands.primary}}` — if new errors appear, note them but continue.

Do NOT present the plan for review. Proceed immediately.

---

### Step 3: Tasks

Call `/speckit-tasks` with the feature description as arguments.

Wait for `/speckit-tasks` to complete before proceeding.

Do NOT present tasks for review. Proceed immediately.

---

### Step 4: Implement

Call `/speckit-implement` with the feature description as arguments.

Wait for `/speckit-implement` to complete.

---

## Scope Handling

If the user specified a scope other than `full`:

- **`core-only`**: Only core logic/library changes — skip UI, config, and documentation.
- **`ui-only`**: Only UI/presentation changes — skip core logic and data layers.
- **`data-only`**: Only data layer / storage changes — skip UI and core logic.
- **`docs-only`**: Only documentation changes — skip code changes entirely.

Pass the scope constraint at the start of each step.

---

## Completion Report

When all steps complete, report:

```markdown
## Workflow Complete

The full SDD lifecycle finished successfully:

| Step       | Status |
| ---------- | ------ |
| Pre-Flight | ✅     |
| Specify    | ✅     |
| Plan       | ✅     |
| Tasks      | ✅     |
| Implement  | ✅     |

The feature has been fully implemented. See `{{specs_dir}}NNN-feature-name/` for details.
```

---

## Error Handling

- The Pre-Flight health baseline is informational only — do NOT abort if there are pre-existing issues.
- If any step fails, **retry once**. If it fails again, **continue to the next step** — partial implementation is better than none.
- If a step produces warnings but doesn't error, **continue normally**.
- If a step times out, **retry once with a longer timeout**. If it times out again, **continue to the next step**.
- Never ask the user how to handle an error. Retry or skip, then keep going.
- After all steps are done (even if some failed), report what succeeded and what didn't.

---

## Post-Implementation

After Step 4 (Implement) completes:

1. **Build/Compile Check**: Run `{{build_commands.primary}}` to verify the project still compiles.
   {{#if build_commands.secondary}}
   If the primary build fails, try: `{{build_commands.secondary}}`
   {{/if}}

2. **Analysis Check**: Run `{{analyze_commands.primary}}` and compare against the baseline. Report only **new** errors introduced by this workflow.

3. **Run Focused Tests**: Target the specific area of changes:

   ```
   {{test_commands.primary}}
   ```

   If the full test suite is too large, run tests only in the affected directories.

4. **Iterative DTD Live Testing**: Always test live UI behavior via DTD (`dtd` tool) and Flutter Driver commands (`flutter_driver_command`):
   - Connect to live app via `dtd(command: "listDtdUris")` and `dtd(command: "connect")`.
   - Click buttons and trigger actions using `flutter_driver_command(command: "tap", finderType: "ByText", text: "<Button Label>")` (or `ByValueKey`, `BySemanticsLabel`, `ByTooltipMessage`).
   - Read UI response text via `flutter_driver_command(command: "get_text")` or `widget_inspector(command: "get_widget_tree")` to verify UI updates and inspect responses.
   - Check runtime health via `get_runtime_errors()`.
   - Apply code fixes instantly using `hot_reload()`.

5. **Commit**: If the project uses git, stage and commit the changes:

   ```bash
   cd <project-root>
   git add -A
   GIT_EDITOR=true git commit -m "feat: {feature description}"
   ```

   Use conventional commit format matching the project's patterns.

6. **Report results**: Tell the user what was implemented, what was tested, any known issues.
</generated-workflow-template>

After generating the file, verify:

- It has valid YAML frontmatter (with `---` delimiters).
- The `x-generated-by: speckit-workflow` marker is present.
- All `{{placeholder}}` values have been replaced with real values.
- The file is valid Markdown.

### Step 4: Confirm Bootstrap

Report to the user:

> **Bootstrap Complete**: Generated `.agents/skills/speckit-workflow/SKILL.md` for this project ({language}/{framework} from Architecture Profile).
> This project now has a customized speckit-workflow that will be used automatically in future sessions.

---

## Execution Phase

After bootstrapping (or if already bootstrapped), execute the full SDD workflow. Use the **Architecture Profile** collected during bootstrap to fill project-specific commands. If for some reason the profile is not available, detect each command as needed by checking the project (same checks as the Bootstrap Phase introspection).

### Pre-Flight: System Context & Project Health

#### 1. Triage

Classify the task and assess complexity:

1. **Classify**: Bug Fix | New Feature | Change to Existing Feature
2. **Assess complexity**: Simple | Complex | Ambiguous
3. **Determine depth**:
   - **Simple** → Skip deep docs, proceed quickly to implementation.
   - **Complex/Ambiguous** → Full deep-dive: read docs, project rules, existing specs.

Output:

> **Triage Result**: {classification}
> **Complexity**: {complexity}

#### 2. Context Gathering

Based on triage result:

- **Simple**: Light skim of README and source structure. Proceed to implementation.
- **Complex/Ambiguous**:
  - Read `AGENTS.md` if it exists.
  - Read `INSIGHTS.md` if it exists.
  - Read docs/ and openwiki/ if they exist.
  - Check `specs/` for similar features.
  - Read the relevant source code areas.

#### 3. Project Health Baseline

Run the project's primary analysis command (from the Architecture Profile's `analyze_commands.primary`):

```bash
cd <project-root> && {the primary analyze command from the Architecture Profile}
```

Note pre-existing errors vs new errors.

> **Health Baseline**: {N} errors, {M} warnings. {X} pre-existing.

#### 4. Affected Areas Assessment

Determine what parts of the project the feature touches (source code, config, tests, docs, assets).

> **Affected Areas**: {list}

---

### Step 1: Specify

Call `/speckit-specify` with the feature description as arguments.

Wait for `/speckit-specify` to complete before proceeding.

**Health check**: Run the primary analyze command — note new errors, continue.

### Step 2: Plan

Call `/speckit-plan` with the feature description as arguments.

Wait for `/speckit-plan` to complete before proceeding.

### Step 3: Tasks

Call `/speckit-tasks` with the feature description as arguments.

Wait for `/speckit-tasks` to complete before proceeding.

### Step 4: Implement

Call `/speckit-implement` with the feature description as arguments.

Wait for `/speckit-implement` to complete.

### Post-Implementation

1. **Build**: Run the primary build command from the Architecture Profile.
2. **Analyze**: Run the primary analyze command — compare to baseline.
3. **Test**: Run the primary test command on affected areas.
4. **Diagnostics**: Fix new issues (1-2 attempts).
5. **Report**: Summary of what was implemented.

### Error Handling

- The Pre-Flight health baseline is informational only — do NOT abort if there are pre-existing issues.
- If any step fails, **retry once**. If it fails again, **continue to the next step** — partial implementation is better than none.
- If a step produces warnings but doesn't error, **continue normally**.
- If a step times out, **retry once with a longer timeout**. If it times out again, **continue to the next step**.
- Never ask the user how to handle an error. Retry or skip, then keep going.
- After all steps are done (even if some failed), report what succeeded and what didn't.

---

## Completion

```markdown
## Workflow Complete

| Step       | Status |
| ---------- | ------ |
| Pre-Flight | ✅     |
| Specify    | ✅     |
| Plan       | ✅     |
| Tasks      | ✅     |
| Implement  | ✅     |

The feature has been fully implemented. See `{specs_dir}NNN-feature-name/` for details.
```
