---
name: comparator
description: "Compares two versions of a COBOL program or two related programs and explains differences in both technical and business terms. This skill should be used when the user provides two files and wants to understand what changed. Trigger phrases: what changed, diff these, compare old vs new, what's different, what was added, what was removed."
context: fork
agent: general-purpose
user-invocable: false
allowed-tools: Read Grep Glob
---

# Comparator

Compare two versions of a COBOL program (or two related programs) and explain the differences.

**Before starting, read [knowledge-lookup.md](../_shared/knowledge-lookup.md) for knowledge base access rules and personality guidelines.**

## Inputs

- Read source files from `input/` — expects two files (old/new, version A/B, or two related programs)
- If only one file is provided, ask the user for the second
- Follow the knowledge lookup hierarchy in the shared reference above

## Instructions

1. Read both source files from `input/`
2. Perform structural comparison:

### Added
New paragraphs, fields, copybooks, CALL statements, business logic.

### Removed
Deleted paragraphs, fields, logic paths.

### Modified
Changed conditions, calculations, field definitions, flow control.

### Moved/Reorganized
Same logic relocated to different paragraphs or restructured.

3. For each change, provide:
   - **Technical description**: What changed in code terms
   - **Business impact**: What the change means functionally — new business rule, bug fix, reporting change
   - **Risk assessment**: Could this change introduce issues? Breaking changes to interfaces, data format changes, altered error handling

## Output Format

Group changes by category (Added/Removed/Modified). Lead with a summary of the most significant changes, then detail.

For large diffs, prioritize business-logic changes over cosmetic or structural reorganization.
