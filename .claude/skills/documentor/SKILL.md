---
name: documentor
description: "Produces human-readable documentation files in the output/ folder. This skill should be used only when the user explicitly asks for documentation, reports, diagrams, or shareable artifacts. Trigger phrases: write this up, create a report, I need docs, give me something I can share, document this program, create a diagram, put that in a file."
context: fork
agent: general-purpose
user-invocable: false
allowed-tools: Read Write Grep Glob
---

# Documentor

Transform analysis into clean, shareable documentation files in `output/`.

**This skill is never auto-invoked.** Only trigger when the user explicitly asks for documentation output.

**Before starting, read [knowledge-lookup.md](../_shared/knowledge-lookup.md) for knowledge base access rules and personality guidelines.**

## Inputs

- Read source files from `input/` if needed for direct analysis
- Use findings from other skills already returned in the conversation when available — do not re-analyze from scratch
- Follow the knowledge lookup hierarchy in the shared reference above

## Output Location

All output goes to `output/` as `.md` files. Prefer multiple focused documents over one monolithic file.

## Standard Document Set

Produce what is relevant, not all of them:

| File | Contents |
|---|---|
| `{program}_summary.md` | One-page business summary — purpose, inputs, outputs, dependencies |
| `{program}_flow.md` | Process flow using ASCII diagrams — paragraph call chains, decision trees |
| `{program}_data_map.md` | Data dictionary — fields, their types, business meaning, where used |
| `{program}_business_rules.md` | Extracted business rules in plain language, numbered and organized |
| `{program}_technical_notes.md` | Technical findings — bugs, risks, patterns, recommendations |
| `{program}_call_tree.md` | Program-to-program call chain with COMMAREA/LINKAGE details |

## ASCII Diagram Style

For diagram conventions, see [diagram-guide.md](references/diagram-guide.md).

## Document Rules

- Use markdown headings and tables — keep it scannable
- Name files with the PROGRAM-ID prefix (lowercase, hyphens)
- Each document should stand on its own — don't require reading another doc first
- Include a generation timestamp and source file reference at the top of each doc
- Keep individual documents under 40,000 characters — split if needed
