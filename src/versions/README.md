# COBOL Version Reference

Version-specific knowledge files. Each file covers one COBOL version/dialect and documents **only what differentiates it** from standard COBOL.

## Purpose

These files are loaded into context as soon as the COBOL version is identified or inferred from source code. They supplement the general knowledge in `src/cobol/` with version-specific behaviors, syntax extensions, restrictions, and quirks.

## File Standards

Every `.md` file in this folder must follow this template:

```markdown
# {Version Name}

## Description
Version name, vendor, release info, and primary use context (2-3 sentences).

## Table of Contents
Links to all sections below.

## Unique Syntax & Extensions
Syntax, keywords, or features that exist only in this version — not in standard COBOL.

## Behavioral Differences
Where this version behaves differently from the COBOL standard or other common versions on the same operation.

## Compiler Directives & Options
Version-specific compiler flags, directives, or configuration that affect code behavior.

## Runtime Environment
Execution environment specifics — runtime system, deployment, configuration, platform behavior.

## Deprecated & Removed Features
Standard COBOL features that are unsupported, restricted, or behave unexpectedly in this version.

## Migration Notes
Key differences to be aware of when migrating code to or from this version.

## Gotchas
Version-specific pitfalls — things that work differently than expected compared to other COBOL versions.
```

## Rules

- **Only version-specific content.** Do not repeat general COBOL knowledge from `src/cobol/`. If a concept is standard across versions, it belongs there.
- **One file per version/dialect.** If a vendor has multiple major versions with meaningful differences, each gets its own file.
- **Differentiation focus.** Every section should answer: "What is different about this version?" If there is nothing different for a section, include the heading with "No significant differences from standard COBOL."
- **Cross-reference general knowledge.** Point to `src/cobol/*.md` files when a topic has a general reference and this file only covers the delta.
- **All files must be indexed** in `src/mind_maps/version_map.json` via the mind_map skill.
- **Keywords in the mind map** should include version identifiers, vendor names, unique compiler directives, and distinctive syntax that would help Claude identify this version from source code.

## Naming Convention

`{vendor}_{product}_{version}.md` — lowercase, underscores. Examples:
- `micro_focus_cobol_9.md`
- `ibm_enterprise_cobol_6.md`
- `gnucobol_3.md`
