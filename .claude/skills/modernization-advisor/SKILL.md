---
name: modernization-advisor
description: "Assesses COBOL programs for modernization readiness. This skill should be used when the user asks about migration, refactoring, rewriting, complexity, outdated patterns, or moving off mainframe. Trigger phrases: can we modernize, how hard to migrate, what's outdated, refactoring options, rewrite this, how complex, could this run on a different platform."
context: fork
agent: general-purpose
user-invocable: false
allowed-tools: Read Grep Glob
---

# Modernization Advisor

Assess COBOL programs for modernization readiness. Identify outdated patterns, external dependencies, migration risks, and provide actionable recommendations.

## Inputs

- Read source files from `input/`
- Load version file from `src/versions/` if known — dialect-specific features affect migration difficulty
- Check project context from `src/projects/` for system-level dependencies

## Instructions

1. Read the full source file(s) in `input/`
2. Perform modernization assessment:

### Complexity Score
Estimate based on LOC, cyclomatic complexity (paragraph branches), number of external dependencies, data complexity.

### Deprecated Patterns
GO TO spaghetti, ALTER, NEXT SENTENCE, legacy CICS HANDLE, non-structured flow.

### External Dependencies
Catalog all EXEC CICS, EXEC SQL, CALL, file I/O, JCL dependencies — each is a migration surface.

### Data Layer
VSAM files, DB2 tables, flat files, report outputs — what would need replacement or mapping.

### Business Logic Density
How much is reusable business logic vs platform plumbing (I/O, error handling, formatting).

### Refactoring Opportunities
Dead code removal, paragraph consolidation, copybook extraction, structured flow conversion.

### Migration Path Options
- **Rehost**: Recompile on Linux/cloud
- **Replatform**: Re-database, swap middleware
- **Refactor**: Restructure COBOL, modernize patterns
- **Rewrite**: New language entirely

Assess feasibility of each.

## Output Format

Structured assessment. Rate each area (low/medium/high risk). Be honest about difficulty — do not oversell easy migration.
