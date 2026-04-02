---
name: mind-map
description: "Manages mind map index files in src/mind_maps/. This skill should be used when creating or updating knowledge files in src/cobol/, src/versions/, or src/projects/ to keep the JSON indexes current. Handles keyword discipline, validation, and deduplication."
user-invocable: false
---

# Mind Map Management

How to create and update mind map index files in `src/mind_maps/`.

## Structure

Each mind map is a JSON array. Each entry follows this template:

```json
{
  "name": "short identifier — file or concept name",
  "location": "relative path to the .md file (e.g. src/projects/paybat01.md)",
  "description": "one-line summary of what this entry covers",
  "key_words": ["specific", "distinctive", "terms"]
}
```

## Mind Map Files

| File | Indexes |
|---|---|
| `project_map.json` | `src/projects/*.md` |
| `cobol_map.json` | `src/cobol/*.md` |
| `version_map.json` | `src/versions/*.md` |

## Naming Conventions

- **Map files**: `{category}_map.json` — lowercase, underscores
- **Knowledge files**: `{short_name}.md` — lowercase, underscores, concise

## Creating an Entry

1. Write the `.md` knowledge file first.
2. Add one entry to the appropriate mind map JSON.
3. The `location` field must match the actual file path exactly.

## Keyword Discipline

Keywords drive lookup efficiency. Poor keywords cause every file to be loaded, defeating the purpose.

**Do:**
- Use terms specific to this entry that distinguish it from others
- Use COBOL-specific terms when applicable (e.g. `EXEC CICS`, `QSAM`, `VSAM`, `COMP-3`)
- Include version identifiers in version entries (e.g. `COBOL-85`, `COBOL-2002`, `Enterprise COBOL`)
- Include program names, system names, or unique business terms for project entries

**Do not:**
- Use generic terms that apply to most entries (e.g. `COBOL`, `program`, `code`, `mainframe`)
- Duplicate keywords already present in other entries unless the overlap is meaningful
- Add more than 8-10 keywords per entry — if you need more, the description should carry the rest

## Updating an Entry

- When new information is added to an existing `.md` file, review its mind map entry.
- Update `description` if the scope has changed.
- Add keywords only if they are distinctive and not already covered by existing entries.
- Remove keywords that have become too common across entries.

## Validation

After any mind map update:
- Confirm the `location` path exists.
- Confirm no duplicate entries for the same file.
- Scan for keyword overlap — if the same keyword appears in more than 3 entries, it is too generic and should be removed from most of them.
