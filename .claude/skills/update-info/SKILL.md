---
name: update-info
description: "Keeps the knowledge base current. This skill runs continuously — whenever new information surfaces during a session from the user, code analysis, or web research, it determines what gets written where. Also updates personality rules when the user gives communication feedback."
user-invocable: false
---

# Update Info

Keeps the knowledge base current. Whenever new information surfaces during a session — from the user, from code analysis, or from web research — this skill governs what gets written where and how.

This skill runs implicitly. Do not wait for the user to ask. If new knowledge was learned, update the docs.

## When to Trigger

- The user provides new information about a project (environment, programs, business context, contacts, schedules)
- The user provides new information about a COBOL version or dialect
- Claude discovers something from code analysis that was not previously documented (a new program in a project, an undocumented business rule, a version-specific behavior)
- Claude finds information online that fills a gap in the existing knowledge base
- The user corrects something already in the docs
- The user expresses frustration with Claude's communication style or explicitly requests tone/behavior changes ("don't answer like...", "I wish you would...", "stop doing...")

## Where Things Go

### `src/projects/{project_name}.md`
- Environment details (OS, DB, scheduler, COBOL version)
- Program names and their roles
- Business context, process descriptions, schedules
- Known relationships between programs
- Error handling approaches
- Open questions and TODOs
- **Does NOT belong here**: General COBOL syntax, version-specific behavior, reusable patterns

### `src/cobol/{topic}.md`
- General COBOL knowledge that applies across versions and projects
- One topic per file — if new info doesn't fit an existing file, create a new one
- Max ~40,000 characters per file — split if exceeded
- **Does NOT belong here**: Version-specific behavior, project-specific details
- Follow the 7-section template in `src/cobol/README.md`

### `src/versions/{vendor}_{product}_{version}.md`
- Only what differentiates this version from standard COBOL
- Unique syntax, behavioral differences, compiler directives, runtime details
- **Does NOT belong here**: General COBOL knowledge that applies to all versions
- Follow the template in `src/versions/README.md`

### `src/personality.md`
- Updated when the user expresses frustration with communication style
- Updated when the user explicitly asks for tone or behavior changes
- Add the rule in the same direct style as existing entries
- Never remove existing rules unless the user explicitly contradicts a prior one

## How to Update

1. **Identify the category** — project, cobol, version, or personality
2. **Check if a file exists** — read the relevant mind map or file directly
3. **If the file exists**: Read it, add the new information in the appropriate section. Do not rewrite the whole file — surgical additions only.
4. **If no file exists**: Create one following the appropriate README template. Add a mind map entry using the `mind-map` skill rules.
5. **Check for duplication** — before adding, verify the information isn't already captured elsewhere. If it is, determine which file should own it based on the rules above.
6. **Update the mind map** if new keywords or description changes are warranted by the new information.

## What NOT to Do

- Do not create a new `src/cobol/` file for something that fits in an existing file — add to it
- Do not put version-specific info in general cobol docs
- Do not put general cobol info in version docs
- Do not put reusable patterns in project docs
- Do not duplicate content across files — one owner per piece of knowledge, cross-reference the rest
- Do not announce every update to the user — just do it quietly. If the update is significant (new file created, major addition), mention it briefly.
