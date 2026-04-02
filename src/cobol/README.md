# COBOL Knowledge Base

General, version-agnostic COBOL reference knowledge. Each file covers a single focused topic.

## File Standards

Every `.md` file in this folder must follow this template:

```markdown
# {Topic Title}

## Description
Brief summary (2-3 sentences) of what this file covers and when to reference it.

## Table of Contents
Links to all sections below.

## Core Concepts
The foundational knowledge for this topic. Define terms, explain mechanics.

## Syntax & Examples
Code snippets with inline explanations. Use fenced COBOL code blocks.

## Common Patterns
Typical real-world usage seen in production COBOL systems.

## Gotchas
Pitfalls, subtle bugs, common misunderstandings, things that trip people up.
Each gotcha should be a bulleted item with a brief explanation of why it's dangerous.

## Related Topics
Bulleted list of pointers to other files in this folder.
Do NOT duplicate content — just reference the file and explain the relationship.
```

## Rules

- **One topic per file.** If a file exceeds ~40,000 characters or covers more than one distinct topic, split it.
- **No version-specific content.** Version/dialect differences belong in `src/versions/`.
- **No project-specific content.** Application-specific knowledge belongs in `src/projects/`.
- **Cross-reference, don't duplicate.** Use the Related Topics section to point to other files.
- **All files must be indexed** in `src/mind_maps/cobol_map.json` via the mind_map skill.

## Minimum Categories

Every file must include all seven sections from the template. If a section is not applicable (e.g., Report Writer has no JCL interaction), include the heading with "Not applicable to this topic." — do not omit the section.
