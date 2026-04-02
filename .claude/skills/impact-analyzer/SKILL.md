---
name: impact-analyzer
description: "Traces the ripple effect of a proposed change across COBOL programs. This skill should be used when the user is planning a change and needs to understand what would be affected. Trigger phrases: what if we change, where is this used, what would break, what calls this, impact of changing, what programs use this, blast radius, upstream, downstream."
context: fork
agent: general-purpose
user-invocable: false
allowed-tools: Read Grep Glob
---

# Impact Analyzer

Given a change target (field, copybook, business rule, paragraph), identify everything that would be affected.

**Before starting, read [knowledge-lookup.md](../_shared/knowledge-lookup.md) for knowledge base access rules and personality guidelines.**

## Inputs

- Read source files from `input/` — may need multiple files to trace cross-program impact
- Follow the knowledge lookup hierarchy in the shared reference above

## Instructions

1. Read all source files in `input/`
2. Identify the change target (field name, copybook, paragraph, program, etc.)
3. Trace impact across all available code:

### Field-Level
Every place the field is referenced — MOVEs, conditions, COMPUTEs, I/O, CALL parameters.

### Copybook-Level
Every program that COPYs it, and what fields/logic depend on the copybook content.

### Paragraph-Level
What PERFORMs it, what it PERFORMs, data dependencies within it.

### Program-Level
CALLs to/from, LINKAGE SECTION dependencies, shared files or DB2 tables.

### Data Format Changes
If a field size or type changes, identify all downstream format dependencies (files, reports, other programs, database columns).

## Impact Categories

- **Direct**: Code that references the change target
- **Indirect**: Code affected by a direct impact (e.g., a report that reads a file written by the changed program)
- **Unknown**: Possible impacts that cannot be confirmed without additional source files

## Output Format

Structured by severity. Flag anything requiring coordinated changes across multiple programs. Note when the analysis is incomplete due to missing source files.
