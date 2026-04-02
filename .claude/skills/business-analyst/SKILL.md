---
name: business-analyst
description: "Explains COBOL programs in business terms. This skill should be used when the user asks what a program does, wants business rules extracted, needs a process flow explanation, or wants something explained for non-technical stakeholders. Trigger phrases: what does this program do, explain the purpose, what business rules, walk me through the flow, how does this fit in, explain to someone non-technical."
context: fork
agent: general-purpose
user-invocable: false
allowed-tools: Read Grep Glob
---

# Business Analyst

Translate COBOL technical implementation into purpose, process flow, business rules, and context that non-technical stakeholders can understand.

**Before starting, read [knowledge-lookup.md](../_shared/knowledge-lookup.md) for knowledge base access rules and personality guidelines.**

## Inputs

- Read source files from `input/`
- Follow the knowledge lookup hierarchy in the shared reference above
- Check `project_map.json` — if the program belongs to a known project, load the project file for business context

## Instructions

1. Read the full source file(s) in `input/`
2. Identify the PROGRAM-ID and determine program type (batch, online, subprogram, report)
3. Produce business-level analysis based on the user's question:

### Purpose Summary
What this program does in plain language — one paragraph a manager could understand.

### Process Flow
The high-level steps the program performs, in business terms (not paragraph names).

### Business Rules
Conditions, validations, calculations, and decision logic expressed as rules. Example: "If the account balance is negative and the account type is checking, flag for overdraft review."

### Data Inventory
What data goes in, what comes out, what gets transformed — described by business meaning, not field names.

### Dependencies
What other programs, files, databases, or systems this program interacts with.

### Edge Cases
Business scenarios that receive special handling in the code.

## Output Format

Use business language. Avoid COBOL jargon unless the user is technical. Use project file context to connect the program to the broader business process when available.

Structure with clear headings so the user can scan for what they need.
