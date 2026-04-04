---
name: documentor
description: "Produces human-readable documentation files in the output/ folder. This skill should be used only when the user explicitly asks for documentation, reports, diagrams, or shareable artifacts. Trigger phrases: write this up, create a report, I need docs, give me something I can share, document this program, create a diagram, put that in a file."
context: fork
agent: general-purpose
user-invocable: false
allowed-tools: Read Write Grep Glob Bash
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
- If a document contains more than one section (## heading), include a **Table of Contents** after the problem statement listing all sections as an ordered list

## Required Document Structure

Every document must open with a **Problem Statement** section immediately after the title. This section restates the specific questions or goals the document was created to answer. The reader should understand the purpose of the document before reading any analysis.

Example:

```markdown
# Clearinghouse Unit Calculation and Enrollment Status Logic

## Problem Statement

This document addresses the following questions:

1. How does FA0155CB increase and decrease ...
2. How does FA0155CB determine ...
...
```

Sections that follow should map 1:1 to the questions or goals in the problem statement. Do not include orphan sections that don't trace back to the stated purpose.

## Review-Then-Convert Workflow

**Do NOT generate `.docx` files immediately.** Follow this two-phase workflow:

### Phase 1 — Draft `.md` and return for review

1. Write all `.md` files to `output/`.
2. Return the file path(s) to the calling agent for review. Do not proceed to `.docx` conversion.
3. Wait for the calling agent to respond with one of:
   - **Approved** — proceed to Phase 2.
   - **Rework instructions** — revise the `.md` file(s) per the feedback, then return the updated path(s) for another review round.

### Phase 2 — Generate `.docx`

Once the calling agent approves the `.md` content, convert to `.docx`:

```bash
python "${CLAUDE_SKILL_DIR}/scripts/md_to_docx.py" --batch output/
```

Or for a single file:

```bash
python "${CLAUDE_SKILL_DIR}/scripts/md_to_docx.py" output/{filename}.md
```

The script produces styled Word documents with dark blue headers, dark gray table headers, alternating row shading, and monospace code blocks. The `.docx` files are written alongside the `.md` files in `output/`.

**If rework is requested after `.docx` already exists:** Update the `.md` first, then re-run the conversion script to regenerate the `.docx`. Both files must always be in sync.
