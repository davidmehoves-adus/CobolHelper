# Knowledge Lookup Rules

This file is shared across all forked skills. Follow these rules when you need to understand COBOL code, identify a version, or find project context.

## Mind Maps

Mind maps are JSON index files in `src/mind_maps/` that allow efficient lookup into knowledge files. Always read the mind map first — never bulk-read all files in a folder.

Each entry has:
```json
{
  "name": "",
  "location": "",
  "description": "",
  "key_words": []
}
```

### Lookup Process

1. Read the relevant mind map JSON.
2. Match the task context against `name`, `description`, and `key_words`.
3. If more than three entries match on keywords, prioritize entries with **multiple keyword matches** before loading single-match files.
4. Load only the files you need. If the first batch answers the question, stop.

## Available Mind Maps

| File | Location | When to read |
|---|---|---|
| `cobol_map.json` | `src/mind_maps/cobol_map.json` | When you encounter unfamiliar COBOL syntax, patterns, or structures — check here before relying on training data |
| `version_map.json` | `src/mind_maps/version_map.json` | As soon as the COBOL version/dialect is known or inferred. If unknown and cannot be inferred, skip. |
| `project_map.json` | `src/mind_maps/project_map.json` | When the user references a project by name, or when code can be identified as belonging to a known project (e.g., by PROGRAM-ID) |

## Knowledge Hierarchy

When you need to understand something in COBOL code, follow this order:
1. **`src/versions/`** — load the relevant version file when the dialect is identified
2. **`src/cobol/`** — consult via `cobol_map.json` before relying on training knowledge
3. **Training knowledge** — last resort only

## Personality

Read and follow `src/personality.md`. These are non-negotiable rules governing tone, honesty, and communication style in all output — including subagent responses.
