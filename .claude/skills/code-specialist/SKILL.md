---
name: code-specialist
description: "Deep technical COBOL code analysis. This skill should be used when the user asks to find bugs, debug abends, trace logic flow, identify dead code, or explain what specific paragraphs or sections do at the implementation level. Trigger phrases: what's wrong, find the bug, why is this abending, trace X, what does this paragraph do, dead code, risky patterns."
context: fork
agent: general-purpose
user-invocable: false
allowed-tools: Read Grep Glob Bash(wc:*)
---

# Code Specialist

Deep technical analysis of COBOL source code. Find bugs, trace logic, identify risky patterns, and explain code mechanics at the implementation level.

**Before starting, read [knowledge-lookup.md](../_shared/knowledge-lookup.md) for knowledge base access rules and personality guidelines.**

## Inputs

- Read source files from `input/`
- Follow the knowledge lookup hierarchy in the shared reference above

## Instructions

1. Read the full source file(s) in `input/`
2. Identify the PROGRAM-ID and overall structure
3. Based on the user's question, perform targeted analysis:

### Bug Hunting
- Uninitialized fields, unchecked file status, subscript out of range
- Incorrect PERFORM THRU ranges, numeric/alphanumeric mismatches
- Missing AT END, division by zero paths, sign handling errors

### Logic Tracing
- Follow execution flow through PERFORM chains
- Identify all paths to/from a paragraph
- Map condition branches and evaluate completeness

### Data Tracing
- Track a field from definition through all MOVEs, COMPUTEs, reads, and writes
- Map field dependencies across paragraphs

### Dead Code Detection
- Paragraphs never PERFORMed or CALLed
- Unreachable branches, commented-out blocks

### Risk Assessment
- GO TO spaghetti, PERFORM THRU with fall-through
- Missing error handling on I/O, unchecked SQLCODE/EIBRESP

## Output Format

For each finding include:
- **What**: The issue, trace result, or explanation
- **Where**: Paragraph name, approximate line reference
- **Why it matters**: Impact, risk level for bugs
- **Suggested fix**: If applicable

Keep the response focused on what was asked. Do not exhaustively analyze everything unless asked for a full review.
