---
name: triage
description: "Classify incoming work as bug, new feature, or change to existing feature, and determine the appropriate depth of investigation and protocol to follow."
---

# Triage — Task Classification & Depth Protocol

Classifies incoming work into one of three categories and determines the appropriate depth of investigation. Call this as the first step before any feature development workflow begins.

## Classification

Analyze the feature description or issue to classify the task:

### 1. Bug Fix

**Characteristics:**
- Something is broken or not working as expected
- Existing behavior needs correction
- Error in production or development output
- Regression from a previous change

**Protocol:**
1. Reproduce the issue or understand the failure from logs/errors
2. Search existing code for the relevant area
3. Read any project INSIGHTS.md or similar knowledge base for known issues and workarounds
4. Read project docs if the affected area is unfamiliar
5. Fix the root cause — not the symptom
6. Verify the fix with tests or runtime checks

### 2. New Feature

**Characteristics:**
- Building something that doesn't exist yet
- Adding new functionality, screens, or capabilities
- Creating new entities, usecases/services, or data sources
- No existing implementation to modify

**Protocol:**
- If **complex or ambiguous**: Follow full Pre-Flight (read knowledge base + docs deeply), then the standard feature development workflow
- If **simple/straightforward**: Proceed directly to implementation with minimal Pre-Flight investigation

### 3. Change to Existing Feature

**Characteristics:**
- Modifying existing behavior without breaking it
- Extending existing functionality (new methods, new fields)
- Refactoring or optimizing existing code
- Upgrading dependencies or migrating patterns
- Adding a new variant of an existing pattern

**Protocol:**
1. Understand the existing implementation by reading relevant source files
2. Read project INSIGHTS.md or knowledge base for nuances about the affected area
3. Read project docs if the architecture interaction is complex
4. Make targeted, minimal changes
5. Verify backward compatibility

## Complexity Assessment

Evaluate the task against these criteria to determine investigation depth:

| Factor | Simple | Complex |
|--------|--------|---------|
| Scope | Single file/entity | Multiple files/layers |
| Patterns | Existing, well-established | New or unfamiliar |
| Cross-cutting | None | Multiple subsystems involved |
| Requirements | Clear, unambiguous | Ambiguous or underspecified |
| Risk | Low (easy to revert) | High (data, performance, UX) |

- **Simple/Straightforward**: Proceed with minimal investigation — the existing workflow handles it efficiently.
- **Complex**: MUST read project knowledge base (INSIGHTS.md or equivalent) and relevant docs before implementation. Search for similar features as prior art.
- **Ambiguous**: MUST read knowledge base and docs, and search for prior art before making design decisions.

## Output

After triage, output the classification and complexity assessment so the workflow can adjust its depth accordingly:

> **Triage Result**: {Bug Fix | New Feature | Change to Existing Feature}
> **Complexity**: {Simple | Complex | Ambiguous}
> **Depth Plan**: {Brief description of what will be investigated}
