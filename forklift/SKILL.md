---
name: forklift
description: "Level 2 automation: multi-feature batch orchestration. Takes a milestone of interlinked features, resolves dependencies, and runs /palletjack for each feature in correct order. Manages inter-feature contracts, verifies integration, and delivers a complete batch. The warehouse manager that moves pallets in sequence."
---

# Forklift — Multi-Feature Batch Orchestrator

You are a forklift. Your job is to manage a queue of interlinked
features, resolve their dependencies, and execute each one via
`/palletjack` in the correct order. You ensure the output of feature N
feeds cleanly into feature N+1.

You do not write code directly. You orchestrate. You manage the batch.

## User Input

```text
$ARGUMENTS
```

Parse the user input for:

**Required:**
- **Milestone description** — A description of the batch goal and the
  features it contains. Can be:
  - A comma-separated list of feature descriptions.
  - A milestone name that maps to a spec directory.
  - A natural language description of a multi-feature objective.

**Optional:**
- **scope** — `full`, `backend-only`, `frontend-only`. Default: `full`.
- **max-features** — Maximum features to process in this run. Default: 7.
- **dry-run** — Analyze dependencies and produce execution plan without
  running implementation. Default: false.

If no milestone description is provided, **stop immediately**.

## Core Principle: Dependency-Ordered Execution

Features are not independent. Feature B may depend on Feature A's
entities, services, or API contracts. Running them out of order produces
broken integrations and wasted work.

**Your primary job is dependency resolution.** Before any feature runs,
you MUST build a dependency graph and determine the correct execution
order.

## Execution Pipeline

### Step 1: Feature Extraction

Parse the milestone description into individual feature specs:

1. If the input is a list of features, extract each one.
2. If the input is a milestone name, look in `specs/` for related
   feature directories.
3. If the input is a natural language objective, decompose it into
   concrete features.

For each feature, capture:
- **Name**: Short identifier (e.g., "amazon-deal-scraper").
- **Description**: What it does.
- **Dependencies**: What must exist before it can run.
- **Outputs**: What it produces for downstream features.

### Step 2: Dependency Graph

Build a directed acyclic graph (DAG) of feature dependencies:

1. For each feature, identify what it needs from other features:
   - Entity definitions (e.g., Feature B uses ChannelListing from Feature A).
   - Service interfaces (e.g., Feature B calls DealScraperService).
   - GraphQL schema extensions (e.g., Feature B extends types from Feature A).
   - Configuration (e.g., Feature B needs config keys set by Feature A).
2. Detect circular dependencies. If found, **stop** and report — cycles
   must be resolved by splitting or merging features.
3. Topological-sort the DAG to produce an execution order.

Output the execution order as a table:

```markdown
## Forklift Execution Plan

| Order | Feature             | Depends On          | Scope   |
| ----- | ------------------- | ------------------- | ------- |
| 1     | barcode-vectorize   | (none)              | backend |
| 2     | deal-scraper-v2     | barcode-vectorize   | backend |
| 3     | listing-matcher     | deal-scraper-v2     | backend |
| 4     | dashboard-deals     | listing-matcher     | frontend|
| 5     | user-engagement     | dashboard-deals     | full    |
```

If `--dry-run` was specified, output the plan and stop.

### Step 3: Inter-Feature Contract Verification

Before running the batch, verify that inter-feature contracts are
compatible:

1. For each dependency edge (A -> B), check:
   - Does Feature A produce what Feature B expects?
   - Are the entity/service interfaces compatible?
   - Are there schema conflicts?
2. If a contract is broken, either:
   - Adjust Feature B's spec to match Feature A's output.
   - Adjust Feature A's spec to produce what Feature B needs.
   - Insert an adapter feature between them.

### Step 4: Sequential Execution

For each feature in dependency order:

1. **Log**: `Forklift: Starting feature {N}/{total} — {name}`
2. **Pre-check**: Verify the previous feature's outputs exist:
   - Entities created? Services exported? Schema updated?
   - If outputs are missing, **stop the batch** and report.
3. **Run `/palletjack`** with the feature description.
4. **Post-check**: Verify the feature completed successfully:
   - Build passes? Server healthy? Clean code audit passed?
   - If failed, **retry once**. If still failing, **stop the batch**.
5. **Integration check**: Verify this feature's outputs are accessible
   to the next feature:
   - Can the next feature import this feature's entities/services?
   - Is the GraphQL schema updated if needed?
6. **Log**: `Forklift: Completed feature {N}/{total} — {name} (PASS/FAIL)`

### Step 5: Batch Integration Verification

After all features are complete, run a holistic integration check:

1. **Build**: `bun run build` — zero errors.
2. **Server health**: `grep " ERROR:" server.log` — zero errors.
3. **Clean code audit**: Run the full grep suite across all files
   touched by the entire batch.
4. **Schema consistency**: If GraphQL schema was extended by any
   feature, run `bun x @vendure/cli schema --api shop` and
   `--api admin` to verify.
5. **Cross-feature smoke test**: Verify that the key user flow that
   spans all features works end-to-end.

### Step 6: Batch Completion Report

```markdown
## Forklift Batch Complete

**Milestone**: {description}
**Features**: {completed}/{total}
**Duration**: {start} -> {end}

| Order | Feature             | Status | Audit | Notes          |
| ----- | ------------------- | ------ | ----- | -------------- |
| 1     | {name}              | PASS   | PASS  |                |
| 2     | {name}              | PASS   | PASS  |                |
| 3     | {name}              | FAIL   | —     | Server errors  |

**Batch Integration**: PASS/FAIL
**Build**: PASS/FAIL
**Server**: healthy/errors
**Clean Code**: {N violations found, N fixed}

**Failed Features**: {list with failure reasons}
**Next Actions**: {recommendations}
```

## Error Handling

- **Feature failure**: Retry once. If still failing, stop the batch.
  Do NOT skip failed features and continue — downstream features may
  depend on the failed feature's outputs.
- **Integration failure**: Stop the batch. An integration failure means
  the contract between features is broken — continuing will compound
  the damage.
- **Server errors**: Abort the batch immediately. No feature should run
  on an unhealthy server.
- **Clean code violations**: Fix within the current feature before
  moving to the next. Do not accumulate debt across the batch.
- **Dependency cycle**: Abort and report. Cycles require human
  architectural intervention.

## When to Use

- You have 2-7 interlinked features that need to be built in sequence.
- A milestone requires multiple features with clear dependencies.
- You want autonomous batch delivery without managing each feature.
- `/straddle` generated a batch of specs and needs them executed.

## When NOT to Use

- Single feature (use `/palletjack` directly).
- Features have no dependencies between them (run palletjacks in parallel).
- You need to decide WHAT to build (use `/straddle`).
- The batch requires more than 7 features (split into multiple forklift runs).

## Forklift Limits

- **Maximum 7 features per batch.** Beyond this, context degradation
  makes the orchestrator unreliable. Split larger batches into
  multiple forklift runs with explicit handoff points.
- **Sequential only.** Forklift does not parallelize features. If two
  features are independent, run two forklifts manually or use multiple
  palletjack invocations.
- **No rollback.** Forklift does not undo completed features if a
  later feature fails. Each feature is committed before the next
  begins. Failed features are reported for manual intervention.

## Pool ops — per-repo concurrency rule (2026-08-16)

The forklift pool's per-repo cloud/daytona cap is switched by ONE box file:
`/workspace/active_branch_rule.md` (server re-reads it every dispatch tick,
≤60s cache, applies to the WHOLE pool — no restart). `mode: single` (default)
= 1 live task per repo (safe); `mode: worktree` + `per_repo: N` = up to N
parallel same-repo tasks, each in its OWN git worktree (accounts still bound
the heat — the 1-task-per-account rule never changes). Flip the file to cool
the accounts back down at peak hours; the server logs the transition.
